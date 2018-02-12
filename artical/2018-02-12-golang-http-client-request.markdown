---
layout:     post
title:      "Golang http客户端重用request问题"
subtitle:   "Golang"
date:       2018-02-12
author:     "ZhangZhe"
tags:
    - Golang
    - http客户端
---

# Golang http 客户端
Golang在[issues](https://github.com/golang/go/issues/21838)中新增了http.Request重用的特性，能够复用同一个http.Request，减少对象创建的频率。但是如果request中有body数据需要上传，并且期待使用KeepAlive特性的时候，会产生TIME_WAIT增多的问题[分析](#1)。另外，在http客户端有并发请求同一个host的情况下，也会产生TIME_WAIT[分析](#2)
## <span id=1>request body 重用问题</span>
### 代码如下：
``` golang
func postDefaultTest() {
	httpClient := http.DefaultClient
	b := bytes.NewBuffer([]byte("{\"userID\" : \"xxx\",\"score\" :89}"))
	req, err := http.NewRequest("POST", "http://192.168.5.25:9200/vatest/user", b)
	if err != nil {
		fmt.Println("error new request", err)
		return
	}
	req.Header.Add("Content-Type", "application/json")
	for i := 0; i < 100; i++ {
		fmt.Println(i)
		response, err := httpClient.Do(req)
		if err != nil {
			fmt.Println("Error do request ", err)
			continue
		}
		retB, err := ioutil.ReadAll(response.Body)
		fmt.Println(string(retB), err)
		response.Body.Close()
		time.Sleep(time.Second * 3)
	}
}
```
### 问题分析：
1. 调用httpClient.Do()，首先进行状态与数据构建，然后在http.Client.send()中进行真正的数据发送。
``` go
if resp, didTimeout, err = c.send(req, deadline); err != nil {
    ...
}
```
2. Client.send中会进行cookie的相关检查，并调用内部send()函数发送请求。其中Client.transport()会返回底层tcp连接池示例RoundTripper（接口），默认实例或者自定义的实例。
``` go
// didTimeout is non-nil only if err != nil.
func (c *Client) send(req *Request, deadline time.Time) (resp *Response, didTimeout func() bool, err error) {
	if c.Jar != nil {
		for _, cookie := range c.Jar.Cookies(req.URL) {
			req.AddCookie(cookie)
		}
	}
	resp, didTimeout, err = send(req, c.transport(), deadline)
	if err != nil {
		return nil, didTimeout, err
	}
	if c.Jar != nil {
		if rc := resp.Cookies(); len(rc) > 0 {
			c.Jar.SetCookies(req.URL, rc)
		}
	}
	return resp, nil, nil
}
```


3. send()中，调用特定的的RoundTripper实例中数据收发方法。
``` go
    resp, err = rt.RoundTrip(req)
	if err != nil {
		stopTimer()
		if resp != nil {
			log.Printf("RoundTripper returned a response & error; ignoring response")
		}
		if tlsErr, ok := err.(tls.RecordHeaderError); ok {
			// If we get a bad TLS record header, check to see if the
			// response looks like HTTP and give a more helpful error.
			// See golang.org/issue/11111.
			if string(tlsErr.RecordHeader[:]) == "HTTP/" {
				err = errors.New("http: server gave HTTP response to HTTPS client")
			}
		}
		return nil, didTimeout, err
	}
```
4. 系统自带的RoundTrip实现是http.Transport，其实现了大部分的http1.x/http2.0特性，并在tcp曾支持连接池。上述的问题产生的愿意也在这个实现里面，具体分析如下：
``` go

	for {
        // treq gets modified by roundTrip, so we need to recreate for each retry.
        // 使用request请求数据结构，构造底层tcp连接池与连接实现需要的数据结构。
		treq := &transportRequest{Request: req, trace: trace}
		cm, err := t.connectMethodForRequest(treq)
		if err != nil {
			req.closeBody()
			return nil, err
		}

		// Get the cached or newly-created connection to either the
		// host (for http or https), the http proxy, or the http proxy
		// pre-CONNECTed to https server. In any case, we'll be ready
        // to send it requests.
        // 从连接池获取连接或者新建连接
		pconn, err := t.getConn(treq, cm)
		if err != nil {
			t.setReqCanceler(req, nil)
			req.closeBody()
			return nil, err
		}

		var resp *Response
		if pconn.alt != nil {
            // HTTP/2 path.
            // HTTP/2 支持
			t.setReqCanceler(req, nil) // not cancelable with CancelRequest
			resp, err = pconn.alt.RoundTrip(req)
		} else {
            // HTTP/1.x数据发送，在上述情况下，body不能重复读，这里会返回err，并会关闭底层tcp连接，产生TIME_WAIT
			resp, err = pconn.roundTrip(treq)
		}
		if err == nil {
			return resp, nil
		}
        // 判断重定向的错误返回
		if !pconn.shouldRetryRequest(req, err) {
			fmt.Println("### shouldRetryRequest return ", pconn.shouldRetryRequest(req, err))
			// Issue 16465: return underlying net.Conn.Read error from peek,
			// as we've historically done.
			if e, ok := err.(transportReadFromServerError); ok {
				err = e.err
			}
			return nil, err
		}

		testHookRoundTripRetried()
		// Rewind the body if we're able to.  (HTTP/2 does this itself so we only
		// need to do it for HTTP/1.1 connections.)
		if req.GetBody != nil && pconn.alt == nil {
			newReq := *req
			var err error
            // 在request创建的时候，会根据body数据类型不同，指定不同的GetBody()方法，将
            // 原始body数据内容备份，并在GetBody方法中返回。所以这里会根据系统缓存的数据，
            // 重建body，并替换原始req数据，重新发起一次请求。
            // 由于之前的连接已经关闭（无并发的情况下，连接池中将没有连接可用），所以新请求
            // 会新建一个tcp连接。
            // 上述过程重复进行，产生大量的TIME_WAIT状态。
			newReq.Body, err = req.GetBody()
			if err != nil {
				return nil, err
			}
			req = &newReq
		}
	}
```

5. 底层tcp连接封装实现是http.persistConn，需要说明以下几点：
* 发送函数与实际发送的实现不在一起，数据与返回通过channel传输。
``` go
	// Write the request concurrently with waiting for a response,
	// in case the server decides to reply before reading our full
	// request body.
	startBytesWritten := pc.nwrite
    writeErrCh := make(chan error, 1)
    // 数据发送指定channel中
    pc.writech <- writeRequest{req, writeErrCh, continueCh}
    
```
* 数据发送在另外的goroutine中，从channel读取数据，并将操作结构通过channel返回出去。
``` go
func (pc *persistConn) writeLoop() {
	defer close(pc.writeLoopDone)
	for {
		select {
		case wr := <-pc.writech:
			startBytesWritten := pc.nwrite
			err := wr.req.Request.write(pc.bw, pc.isProxy, wr.req.extra, pc.waitForContinue(wr.continueCh))
			if bre, ok := err.(requestBodyReadError); ok {
				err = bre.error
				// Errors reading from the user's
				// Request.Body are high priority.
				// Set it here before sending on the
				// channels below or calling
				// pc.close() which tears town
				// connections and causes other
				// errors.
				wr.req.setError(err)
			}
			if err == nil {
				err = pc.bw.Flush()
			}
			if err != nil {
				wr.req.Request.closeBody()
				if pc.nwrite == startBytesWritten {
					err = nothingWrittenError{err}
				}
			}
			pc.writeErrCh <- err // to the body reader, which might recycle us
			wr.ch <- err         // to the roundTrip function
			if err != nil {
				pc.close(err)
				return
			}
		case <-pc.closech:
			return
		}
	}
}
```
* 具体实例在新建的时候会创建相关channel以及对应的接受发送goroutine
``` go

func (t *Transport) dialConn(ctx context.Context, cm connectMethod) (*persistConn, error) {
	pconn := &persistConn{
		t:             t,
		cacheKey:      cm.key(),
		reqch:         make(chan requestAndChan, 1),
		writech:       make(chan writeRequest, 1),
		closech:       make(chan struct{}),
		writeErrCh:    make(chan error, 1),
		writeLoopDone: make(chan struct{}),
	}
    ......
	go pconn.readLoop()
	go pconn.writeLoop()
	return pconn, nil
}

```
* 数据发送错误，则会将连接关闭。
``` go
    for {
		testHookWaitResLoop()
		select {
		case err := <-writeErrCh:
			if debugRoundTrip {
				req.logf("writeErrCh resv: %T/%#v", err, err)
			}
			if err != nil {
                // 读取到发送错误信息，关闭连接
				pc.close(fmt.Errorf("write error: %v", err))
				return nil, pc.mapRoundTripError(req, startBytesWritten, err)
			}
			if d := pc.t.ResponseHeaderTimeout; d > 0 {
				if debugRoundTrip {
					req.logf("starting timer for %v", d)
				}
				timer := time.NewTimer(d)
				defer timer.Stop() // prevent leaks
				respHeaderTimer = timer.C
			}
```
## <span id=2>http客户端并发请求</span>
### 问题代码
``` go
func main() {
    for i := 0; i < 50; i++ {
	    go postTest()
    }
    postTest()
}
func postTest(){
    httpClient := http.Client{
		Transport: &http.Transport{
			//MaxIdleConns:        100,
			//MaxIdleConnsPerHost: 100,
		},
	}
	b := bytes.NewBuffer([]byte("{\"userID\" : \"xxx\",\"score\" :89}"))
	req, err := http.NewRequest("POST", "http://192.168.5.25:9200/vatest/user", b)
	if err != nil {
		fmt.Println("error new request", err)
		return
	}
	req.Header.Add("Content-Type", "application/json")
	for i := 0; i < 100; i++ {
		response, err := httpClient.Do(req)
		if err != nil {
			fmt.Println("Error do request ", err)
			continue
		}
		retB, err := ioutil.ReadAll(response.Body)
		fmt.Println(string(retB), err)
		response.Body.Close()
		time.Sleep(time.Second)
	}
}
```
### 问题分析
不用并发，以及设置两路并发不会产生TIME_WAIT问题。并发设置一旦超过2,就会开始产生TIME_WAIT。
问题产生的原因在与默认Transport参数设置：
``` go
type Transport struct {
    ......
	// MaxIdleConns controls the maximum number of idle (keep-alive)
	// connections across all hosts. Zero means no limit.
	MaxIdleConns int

	// MaxIdleConnsPerHost, if non-zero, controls the maximum idle
	// (keep-alive) connections to keep per-host. If zero,
    // DefaultMaxIdleConnsPerHost is used.
    // 根据host进行统一，默认情况下一个host只会保存两个连接，超过的会在使用后关闭。
    MaxIdleConnsPerHost int
    ......
}
// DefaultMaxIdleConnsPerHost is the default value of Transport's
// MaxIdleConnsPerHost.
const DefaultMaxIdleConnsPerHost = 2
```

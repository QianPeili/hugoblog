---
title: "go 的request.Body rewind 功能" 
date: 2017-09-16T18:31:52+08:00
category:
 - 笔记
tags: 
 - go
---

近日参加了实验楼的 [楼赛 第15期 Go语言项目挑战 - 实验楼](https://www.shiyanlou.com/contests/lou15)

第一道题的题目要求是：

> 实现一个简单的数据校验功能，对请求 body 做 md5 计算，然后把结果转为十六进制添加到请求 header 中，头部名称为X-Md5。也就是在HTTP 请求 header 中添加一个 X-Md5: <hex md5 of body> 键值对，如果body为空那么就不填。

我一开始的实现方式很直接暴力：

    func (t *Transport) RoundTrip(req *http.Request) (*http.Response, error) {
    
    	if req.Body == nil {
    		return t.RoundTripper.RoundTrip(req)
    	}
    	b, err := ioutil.ReadAll(req.Body)
    	if len(b) == 0 {
    		return t.RoundTripper.RoundTrip(req)
    	}
    	if err != nil {
    			log.Fatal(err.Error())
    	}
    	hexB := md5.Sum(b)
    	req.Header.Add("X-Md5", hex.EncodeToString(hexB[:]))    
    	return t.RoundTripper.RoundTrip(req)
    }

直接读出 `req.Body` 中的字节数据，进行`md5`运算。我在自己的电脑上测试通过，但是实验环境下 `go test` 会报类似错误：

	net/http: http: ContentLength=N with Body length 0

就是说我的 `req.Body` 本应有数据，突然没数据了。

原因就在于我 `ioutil.ReadAll(req.Body)` 后，缓冲区的数据被我读取完了，发送的时候 `req.Body` 自然为空了。

所以我额外添加了一个操作，完成了题目：

    func (t *Transport) RoundTrip(req *http.Request) (*http.Response, error) {
    
    	if req.Body == nil {
    		return t.RoundTripper.RoundTrip(req)
    	}
    	b, err := ioutil.ReadAll(req.Body)
    	if len(b) == 0 {
    		return t.RoundTripper.RoundTrip(req)
    	}
    	if err != nil {
    			log.Fatal(err.Error())
    	}
    	hexB := md5.Sum(b)
    	req.Header.Add("X-Md5", hex.EncodeToString(hexB[:]))
    	
		// 恢复 req.Body 中被读取的数据
    	bodyBuffer := bytes.NewReader(b)
    	req.Body = ioutil.NopCloser(bodyBuffer)
    
    	return t.RoundTripper.RoundTrip(req)
    } 

那么为何我在我的电脑上操作的时候，就不存在这个问题呢。

这是因为我电脑上的版本是 `go 1.9` 而实验室环境为 `go 1.8.3`。 在 `1.9` 版本中，`net/http/transport.go` 中主要多了这么一段代码[net/http: make Transport retry GetBody requests if nothing written · golang/go@eea8c88](https://github.com/golang/go/commit/eea8c88a095d4aa21893d96441cb5074a7314532#diff-6951e7593bfb1e773c9121df44df1c36R425)：

	func (t *Transport) RoundTrip(req *Request) (*Response, error) {
		
		// ...	
	
	    // Rewind the body if we're able to.  (HTTP/2 does this itself so we only
	    // need to do it for HTTP/1.1 connections.)
	    if req.GetBody != nil && pconn.alt == nil {
	    	newReq := *req
	    	var err error
	    	newReq.Body, err = req.GetBody()
	    	if err != nil {
	    		return nil, err
	    	}
	    	req = &newReq
	    }
	}

这段代码主要功能是，如果之前一次 `RoundTrip` 出错了（比如前面的 `ContentLength=N with Body length 0`），并且程序认定需要重试的时候，会执行一次 `req.GetBody` 方法，获取原先缓存的数据重新放入 `req.Body` 中，所以在 `go 1.9` 版本中，我们并不需要担心 `req.Body` 在请求预处理中因为读取而被清空了。
    
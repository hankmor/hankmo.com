---
title: Nginx Bad Gateway和no live upstreams错误分析
slug: nginx-bad-gateway-no-live-upstreams
categories:
  - 开发运维
tags:
  - nginx
  - 502
description: Nginx返回 `502 Bad Gateway` 时，注意观察其错误日志，通过日志来分析是请求超时或服务不可用导致的，如果输出 `no live upstreams while connecting to upstream` 说明并发请求量大，Nginx将服务下线后仍有请求被转发到下线服务，也可能是服务资源紧缺导致响应不及时，超时后所致。适当的增加 `max_fails` 参数可以有效避免由于极端情况服务被下线造成 `502` 问题。
date: 2023-10-07
updated: 2023-10-07
---

最近项目的生产环境中客户端出现大量的Nginx 502 Bad Gateway错误，逐步排查最终定位到是由于被ddos攻击造成服务器资源耗尽无法响应造成的问题。遂整理过程著文以记之。

## 场景

线上4个节点，每个节点都有两个相同服务通过Nginx作负载均衡，均采用Nginx默认值，未做过多配置，配置类似：
```shell
upstream test-server {
    server 127.0.0.1:8002;
    server 127.0.0.1:8001;
}
```

客户端出现大量的 `502 Bad Gateway` 信息，查看Nginx错误日志，信息如下：
```
no live upstreams while connecting to upstream
```

初步定位问题，发现后台出现了很多莫名其妙的错误，查看Nginx错误日志发现也打印了很多上述错误。怀疑是后台某个接口请求出错导致返回了 `500`，导致Nginx将这个服务下线，后经过排查，后台确实存在一些错误信息，但是出错上述错误的时间不匹配。

后整理思路并认真思考，发现可能思路存在偏差：后台http状态码怎么会影响Nginx将服务下线呢？

因为Http状态码表示http响应的状态，也就是表示响应结果的正确与否，比如 `2xx` 表示服务端正确处理并响应了数据，`4xx` 表示客户端存在错误妨碍了服务端的处理，`5xx` 表示服务端错误导致处理失败了。但是，能够返回这些状态码说明服务端连接正常，只是由于特定原因导致服务端响应错误了。返回了这些状态码Nginx当真会将服务下线吗？试想一下，某一个客户端请求查询一条不存在的数据导致服务端处理时抛出异常并响应 500 状态码，然后Nginx认为服务不可用并将其下线，导致一定时间内所有的请求都返回上述错误，那么罪魁祸首到底是服务端还是客户端？Nginx这么处理是不是太过了呢？Nginx肯定不会这么来设计。

所以，我猜测Nginx不可能如此是非不分，应该是别的原因导致的。

查阅upstream[官方文档](http://nginx.org/en/docs/http/ngx_http_upstream_module.html)，看到这么两个参数配置：

* fail_timeout=time: 设置多少时间内**不能与下游服务成功通信** `max_fails` 次后可以认为下游服务不可用，这是一个周期时间，即每隔 `fail_timeout` 时间都会进行统计，默认为 10 秒。
* `max_fails`=`number`:  设置在 `fail_timeout` 时间内与下游服务**通信失败**的次数，达到该次数后认为服务不可用，默认为1。

按照Nginx的默认设置，也就是说每个10秒统计下有服务的通信失败次数，达到1次就认为服务不可用，此时Ngingx会将其踢下线，后续所有转发到该服务的请求都会返回 `no live upstreams while connecting to upstream`，直到下一个10秒再重新处理（`max_fails` 为0又重新将服务上线）。

关键在于这个 **通信失败** 的理解。通信失败，表示Nginx转发请求给服务，但是服务没有任何响应，而不是一开始怀疑的 HTTP 状态不是200，能成功响应，不论是什么状态码，都应该认为与服务通信成功。

实践才能出真知，为了验证我的猜想，必须进行实验。

## 实验

使用golang编写一个http服务，代码如下：

```go
var inErr bool  
func main() {  
   port := os.Args[1]  
   node := os.Args[2]  
   r := gin.Default()  
   go func() {  
      for {  
         if node == "node1" {  
            break  
         }  
         inErr = !inErr  
         time.Sleep(3 * time.Second)  
      }  
   }()  
   r.GET("/", func(ctx *gin.Context) {  
		if inErr {  
		   ctx.String(http.StatusInternalServerError, "error: "+node)  
		} else {  
		   ctx.JSON(http.StatusOK, gin.H{"msg": "ok, " + node})  
		}
   })  
   r.GET("/err", func(ctx *gin.Context) {  
      ctx.String(http.StatusInternalServerError, "error: "+node)  
      ctx.Abort()  
   })  
   r.GET("/timeout", func(ctx *gin.Context) {  
      time.Sleep(time.Second * 10)  
      ctx.String(http.StatusOK, "after 10s responsed")  
   })  
   _ = r.Run(":" + port) // 参数0为执行文件本身信息，真正的参数下标为1  
}
```
上述代码使用了 [gin](https://gin-gonic.com/zh-cn/) 框架，大致的逻辑：
* 通过命令行运行时传递参数（通过 `os.Args` 获取）来捕获端口`port`和节点名称`node`信息，以便区分不同的节点及端口。
* 新开一个goroutine来切换 `inErr` 状态，该状态表示将 `node2` 设置为错误，为 `true` 时接口会返回 `500` 的http状态码，否则返回 `200`
* 注册了三个接口：`/`、`/err`和`/timeout`，`/`需要处理`inErr`状态，`/err`直接返回500,而`/timeout`会让请求挂起120秒用来模拟请求超时

然后在命令行启动两个服务代表两个节点：
```shell
➜  hello-gin git:(main) ✗ go run main.go 8001 node1
➜  hello-gin git:(main) ✗ go run main.go 8002 node2
```

现在，我们需要使用Nginx来做负载，配置如下：
```go
upstream test-server {
    server 127.0.0.1:8002;
    server 127.0.0.1:8001;
}

server {
	listen 9000;
	server_name 127.0.0.1;

	error_log /var/logs/nginx/error.log;
	access_log /var/logs/nginx/access.log;

	location / {
		proxy_read_timeout 2s;
		proxy_pass http://test-server;
	}
}
```
这里同样使用Nginx默认配置简单将上述两个服务节点做了负载，策略为默认的轮询。

### 节点返回500后不影响其可用性

最后，我们还需要编写代码来模拟并发访问的情况：
```go
func TestReq(t *testing.T) {  
   url := "http://127.0.0.1:9000" // nginx负载地址
   n := 4  
   total := 0  
   for {  
      total++  
      if total > 10 {  
         break  
      }  
      var wg sync.WaitGroup  
      wg.Add(n)  
      for i := 0; i < n; i++ {  
         go func(idx int) {  
            r, _ := http.Get(url)  
            bs, _ := io.ReadAll(r.Body)  
            fmt.Println(idx, " => ", string(bs))  
            wg.Done()  
         }(i)  
      }  
      wg.Wait()  
      time.Sleep(time.Second * 1)  
      fmt.Println("==============")  
   }  
}
```
测试代码请求10次，每次启用4个goroutine来并发请求`/`接口。

此时，node2一开始会进入`inErr`状态，3秒内返回的都是500，然后每三秒切换；而node1始终处于正常状态。如果node2返回500 nginx将其踢下线，那么请求都不会再发到node2。
运行测试代码，结果如下：
```go
3  =>  error: node2
2  =>  error: node2
1  =>  {"msg":"ok, node1"}
0  =>  {"msg":"ok, node1"}
==============
3  =>  error: node2
0  =>  {"msg":"ok, node1"}
1  =>  {"msg":"ok, node1"}
2  =>  error: node2
==============
1  =>  {"msg":"ok, node2"}
3  =>  {"msg":"ok, node1"}
2  =>  {"msg":"ok, node1"}
0  =>  {"msg":"ok, node2"}
==============
0  =>  {"msg":"ok, node1"}
2  =>  {"msg":"ok, node2"}
3  =>  {"msg":"ok, node2"}
1  =>  {"msg":"ok, node1"}
==============
1  =>  {"msg":"ok, node2"}
3  =>  {"msg":"ok, node1"}
0  =>  {"msg":"ok, node1"}
2  =>  {"msg":"ok, node2"}
==============
1  =>  error: node2
3  =>  {"msg":"ok, node1"}
2  =>  {"msg":"ok, node1"}
0  =>  error: node2
……
```
可以看到，node2返回500 Nginx同样认为其有效并继续讲请求转发到该节点。所以，**节点返回500并不影响其在Nginx处的可用性**。

### 请求超时Nginx下线节点并返回502

现在，我们再编码来测试 `/timeout`接口，来模拟服务不可用的真实情况。此时由于Nginx请求超时时间设置为2秒（`proxy_read_timeout`，默认为60秒），所以会很快超时，根据nginx默认的 `max_fails` 为1，此时Nginx会认为服务不可用，将其下线，并返回给客户端 502 Bad Gateway，并在其error.log中打印：`no live upstreams while connecting to upstream` 错误信息。

测试代码如下：
```go
func TestTimeout(t *testing.T) {  
   url := "http://127.0.0.1:9000"  
   timeoutUrl := "http://127.0.0.1:9000/timeout"  
   n := 11  
   var sg sync.WaitGroup  
   sg.Add(n)  
   go func() {  
      _, _ = http.Get(timeoutUrl)  
      sg.Done()  
   }()  
   for i := 0; i < n-1; i++ {  
      go func(x int) {  
         r, _ := http.Get(url)  
         bs, _ := io.ReadAll(r.Body)  
         fmt.Println(string(bs))  
         sg.Done()  
      }(i)  
   }  
   sg.Wait()  
}
```
代码一共并发请求了11次，优先请求 `/timeout` 接口，使某一个服务超时，后续的请求Nginx并不再交给下游服务，而是直接返回 `502 Bad Gateway`。结果如下：

```html
...多个502的html

<html>
<head><title>502 Bad Gateway</title></head>
<body>
<center><h1>502 Bad Gateway</h1></center>
<hr><center>nginx/1.25.2</center>
</body>
</html>

{"msg":"ok, node1"}
```

本次请求测试中，一次成功请求node1的`/`接口和一次请求`/timeout`，node2上第一次就请求了`/timeout`导致超时，后续两个节点的请求均被Nginx拦截并返回502，这一点可以通过服务端和Nginx的错误日志看到:
node2日志：
```shell
[GIN] 2023/10/07 - 14:38:40 | 200 | 10.001371321s |       127.0.0.1 | GET      "/timeout"
```

node1日志：
```shell
[GIN] 2023/10/07 - 14:38:30 | 200 |      39.784µs |       127.0.0.1 | GET      "/"
[GIN] 2023/10/07 - 14:38:42 | 200 |  10.00079605s |       127.0.0.1 | GET      "/timeout"
```

Nginx错误日志：
```go
2023/10/07 14:38:30 [error] 62931#0: *86 no live upstreams while connecting to upstream, client: 127.0.0.1, server: 127.0.0.1, request: "GET / HTTP/1.1", upstream: "http://test-server/", host: "127.0.0.1:9000"
2023/10/07 14:38:30 [error] 62931#0: *87 no live upstreams while connecting to upstream, client: 127.0.0.1, server: 127.0.0.1, request: "GET / HTTP/1.1", upstream: "http://test-server/", host: "127.0.0.1:9000"
2023/10/07 14:38:30 [error] 62931#0: *88 no live upstreams while connecting to upstream, client: 127.0.0.1, server: 127.0.0.1, request: "GET / HTTP/1.1", upstream: "http://test-server/", host: "127.0.0.1:9000"
2023/10/07 14:38:30 [error] 62931#0: *91 no live upstreams while connecting to upstream, client: 127.0.0.1, server: 127.0.0.1, request: "GET / HTTP/1.1", upstream: "http://test-server/", host: "127.0.0.1:9000"
2023/10/07 14:38:30 [error] 62931#0: *90 no live upstreams while connecting to upstream, client: 127.0.0.1, server: 127.0.0.1, request: "GET / HTTP/1.1", upstream: "http://test-server/", host: "127.0.0.1:9000"
2023/10/07 14:38:30 [error] 62931#0: *89 no live upstreams while connecting to upstream, client: 127.0.0.1, server: 127.0.0.1, request: "GET / HTTP/1.1", upstream: "http://test-server/", host: "127.0.0.1:9000"
2023/10/07 14:38:30 [error] 62931#0: *94 no live upstreams while connecting to upstream, client: 127.0.0.1, server: 127.0.0.1, request: "GET / HTTP/1.1", upstream: "http://test-server/", host: "127.0.0.1:9000"
2023/10/07 14:38:30 [error] 62931#0: *95 no live upstreams while connecting to upstream, client: 127.0.0.1, server: 127.0.0.1, request: "GET / HTTP/1.1", upstream: "http://test-server/", host: "127.0.0.1:9000"
2023/10/07 14:38:30 [error] 62931#0: *96 no live upstreams while connecting to upstream, client: 127.0.0.1, server: 127.0.0.1, request: "GET / HTTP/1.1", upstream: "http://test-server/", host: "127.0.0.1:9000"
2023/10/07 14:38:32 [error] 62931#0: *84 upstream timed out (60: Operation timed out) while reading response header from upstream, client: 127.0.0.1, server: 127.0.0.1, request: "GET /timeout HTTP/1.1", upstream: "http://127.0.0.1:8002/timeout", host: "127.0.0.1:9000"
2023/10/07 14:38:34 [error] 62931#0: *84 upstream timed out (60: Operation timed out) while reading response header from upstream, client: 127.0.0.1, server: 127.0.0.1, request: "GET /timeout HTTP/1.1", upstream: "http://127.0.0.1:8001/timeout", host: "127.0.0.1:9000"
```

## 总结

Nginx返回 `502 Bad Gateway` 时，注意观察其错误日志，通过日志来分析是请求超时或服务不可用导致的，如果输出 `no live upstreams while connecting to upstream` 说明并发请求量大，Nginx将服务下线后仍有请求被转发到下线服务，也可能是服务资源紧缺导致响应不及时，超时后所致。适当的增加 `max_fails` 参数可以有效避免由于极端情况服务被下线造成 `502` 问题。

此外，请求超时时Nginx会打印 `upstream timed out (60: Operation timed out) while reading response header from upstream` 错误，便于分析超时的具体接口，定位问题。

## 参考文档

* http://nginx.org/en/docs/http/ngx_http_upstream_module.html
* https://blog.csdn.net/qq_34556414/article/details/106638157
* https://cloud.tencent.com/developer/article/2168754
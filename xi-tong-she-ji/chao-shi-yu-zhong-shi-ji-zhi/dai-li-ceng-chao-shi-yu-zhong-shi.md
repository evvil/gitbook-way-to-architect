# 代理层超时与重试

## Nginx的超时设置

Nginx主要有四类超时设置：客户端超时设置、DNS解析超时设置、代理超时设置，如果使用ngx\_lua，则还有lua相关的超时设置。

### 1、客户端超时设置

通过客户端超时设置避免客户端恶意或者网络状况不佳造成连接长期占用，影响服务端的可处理的能力。

对于客户端的超时主要有：读取请求头超时时间、读取请求体超时时间、发送响应超时时间、长连接超时时间。

* `client_header_timeout` ：**设置读取客户端请求头超时时间**，默认为60s，如果在此超时时间内客户端没有发送完请求头，则响应408\(RequestTime-out\)状态码给客户端。
* `client_body_timeout` ：**设置读取客户端内容体超时时间**，默认为60s，此超时时间指的是两次成功读操作间隔时间，而不是发送整个请求体的超时时间，如果在此超时时间内客户端没有发送任何请求体，则响应408\(RequestTime-out\)状态码给客户端。
* `send_timeout` ：**设置发送响应到客户端的超时时间**，默认为60s，此超时时间指的也是两次成功写操作间隔时间，而不是发送整个响应的超时时间。如果在此超时时间内客户端没有接收任何响应，则Nginx关闭此连接。
* `keepalive_timeout timeout [header_timeout]`：**设置HTTP长连接超时时间**，其中，第一个参数timeout是告诉Nginx长连接超时时间是多少，默认为75s。第二个参数header\_timeout是用于设置响应头“Keep-Alive: timeout=time”，即告知客户端长连接超时时间。两个参数可以不一样，“Keep-Alive:timeout=time”响应头可以在Mozilla和Konqueror系列浏览器起作用，而MSIE长连接默认大约为60s，而不会使用“Keep-Alive: timeout=time”。如Httpclient框架会使用“Keep-Alive: timeout=time”响应头的超时\(如果不设置默认，则认为是永久\)。如果timeout设置为0，则表示禁用长连接。此参数要配合`keepalive_disable` 和`keepalive_requests`一起使用。`keepalive_disable` 表示禁用哪些浏览器的长连接，默认值为`msie6`，即禁用一些老版本的MSIE的长连接支持。`keepalive_requests`参数作用是限制一个客户端可以通过此长连接的请求次数，默认为100。

### 2、DNS超时设置

当在Nginx中使用域名时，如下两个域名会在Nginx解析配置文件的阶段被解析成`IP`地址并记录到`upstream`上。

```text
upstream backend {     
    server c0.3.cn;     
    server c1.3.cn; 
} 
```

此时，就需要设置域名解析超时，参数为`resolver_timeout` 和`resolver address ... [valid=time]`。

### 3、代理超时设置

Nginx配置如下所示：

```lua
upstream backend{
  server 192.168.61.1: 9080 max_fails=2 fail_timeout=10s weight=1;
  server 192.168.61.1: 9090 max_fails=2 fail_timeout=10s weight=1;
}
server {
  ... ... 
  location /test {
      ... ... 
      proxy_connect_timeout 5s;
      proxy_read_timeout 5s;
      proxy_send_timeout 5s;
  
      proxy_next_upstream error timeout;
      proxy_next_upstream_timeout 0;
      proxy_next_upstream_tries 0;
  
      proxy_pass http://backend; 
      
      add_header upstream_addr $upstream_addr;
  }
} 
```

 上面的配置中包含了主要有三组不同的超时设置：网络连接/读/写超时设置、失败重试机制设置、upstream存活超时设置。

#### 网络连接/读/写超时设置

* `proxy_connect_timeout`：与后端/上游服务器建立连接的超时时间，默认为60s，此时间不超过75s。
* `proxy_read_timeout：`设置从后端/上游服务器读取响应的超时时间，默认为60s，此超时时间指的是两次成功读操作间隔时间，而不是读取某个请求响应体的超时时间，如果在此超时时间内上游服务器没有发送任何响应，则Nginx关闭此连接。
* `proxy_send_timeout`：设置往后端/上游服务器发送请求的超时时间，默认为60s，此超时时间指的是两次成功写操作间隔时间，而不是发送整个请求的超时时间，如果在此超时时间内上游服务器没有接收任何响应，则Nginx关闭此连接。

对于内网高并发服务，请根据需要调整这几个参数，比如内网服务TP999为1s，可以将连接超时设置为100~500毫秒，而读超时可以为1.5~3秒左右。

#### 失败重试机制设置

* `proxy_next_upstream`：配置在什么情况下需要请求下一台上游服务器进行重试。默认为`error timeout`。
  * 配置在error表示与上游服务器建立连接、写请求或者读响应头出错。
  * timeout表示与上游服务器建立连接、写请求或者读响应头超时。
  * invalid\_header表示上游服务器返回空的或错误的响应头。
  * http\_XXX表示上游服务器返回特定的状态码。
  * non\_idempotent表示RFC-2616定义的非幂等HTTP方法\(POST、LOCK、PATCH\)，也可以在失败后重试下一台上游服务器\(即默认幂等方法GET、HEAD、PUT、DELETE、OPTIONS、TRACE才可以重试\)。
  * off表示禁用重试。
* `proxy_next_upstream_tries`：设置重试次数，默认0表示不限制，注意此重试次数指的是所有请求次数\(包括第一次和之后的重试次数之和\)。
* `proxy_next_upstream_timeout` ：设置重试最大超时时间，默认0表示不限制。

即在`proxy_next_upstream_timeout`时间内允许`proxy_next_upstream_tries`次重试。如果超过了其中一个设置，则`Nginx`也会结束重试并返回客户端响应\(可能是错误码\)。

#### upstream存活超时设置

`max_fails`和`fail_timeout`：配置什么时候Nginx将上游服务器认定为不可用/不存活。当上游服务器在`fail_timeout`时间内失败了`max_fails`次，则认为该上游服务器不可用/不存活，并在接下来的`fail_timeout`时间内从`upstream`摘掉该节点\(即请求不会转发到该上游服务器\)。

什么情况下被认定为失败呢?其由 `proxy_next_upstream`定义，不过，不管 proxy\_next\_upstream如何配置，`error / timeout  / invalid_header` 都将被认为是失败。

如`server 192.168.61.1:9090 max_fails=2 fail_timeout=10s`表示在10s内如果失败了2次，则在接下来的10s内认定该节点不可用/不存活。这种存活检测机制是只有当访问该上游服务器时，采取惰性检查，可以使用`ngx_http_upstream_check_module`配置主动检查。

`max_fails`设置为0表示不检查服务器是否可用\(即认为一直可用\)，如果upstream中仅剩一台上游服务器时，则该服务器是不会被摘除的，将从不被认为不可用。

#### ngx\_lua超时设置

当我们使用ngx\_lua时，也可以考虑设置如下网络连接/读/写超时。

```text
lua_socket_connect_timeout  100ms; 
lua_socket_send_timeout    200ms; 
lua_socket_read_timeout    500ms; 
```

在使用lua时，我们会按照如下策略进行重试。

```lua
if (status == 502 or status == 503 or status ==504) and request_time < 200 then     
    resp =capture(proxy_uri)     
    status =resp.status     
    body =resp.body    
    request_timerequest_time = request_time + tonumber(var.request_time) * 1000 
end 
```

即如果状态码是500/502/503/504时，并且该次请求耗时在200毫秒以内，则我们进行一次重试。

## **2. Twemproxy**

`Twemproxy`是`Twitter`开源的`Redis`和`Memcache`代理中间件，其目的是减少与后端缓存服务器的连接数。

* `timeout`：表示与后端服务器建立连接、接收响应的超时时间，默认永不超时。
* `server_retry_timeout`和`server_failure_limit`：当开启`auto_eject_hosts`，即当后端服务器不可用时自动摘除这些节点并在一定时间后进行重试。`server_failure_limit`设置连续失败多少次后将节点临时摘除，`server_retry_timeout`设置摘除节点后等待多久进行重试，从而保证不永久性的将节点摘除。

> 当Redis作为缓存使用时，一定要为Redis客户端设置连接超时
>
> 使用缓存的意义在于加速数据访问，如果Redis长时间出现连接超时，就失去了缓存的意义。因此及时抛出超时异常，主动分析原因，排除故障。



## 内容来源

[Nginx动态解析域名方案](http://blog.51cto.com/renzhiyuan/1980890)

《亿级流量网站架构核心技术》：超时与重试机制


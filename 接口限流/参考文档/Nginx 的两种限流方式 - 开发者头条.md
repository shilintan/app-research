## Nginx 的两种限流方式

#### 独家号 [陈一乐](https://toutiao.io/subjects/261093) 作者 码代码的陈同学[原文链接](https://chenyongjun.vip/articles/81)

系统设计时一般会预估负载，当系统暴露在公网中时，恶意攻击或正常突发流量等都可能导致系统被压垮，而限流就是保护措施之一。限流即控制流量，本文将记录 Nginx 的二种限流设置。

## 生活中的 “限流”？

限流并非新鲜事，在生活中亦无处不在，下面例举一二：

- 博物馆：限制每天参观总人数以保护文物
- 高铁安检：有若干安检口，旅客依次排队，工作人员根据安检快慢决定是否放人进去。遇到节假日，可以增加安检口来提高处理能力(**横向拓展**)，同时增加排队等待区长度(**缓存待处理任务**)。
- 办理银行业务：所有人先领号，各窗口叫号处理。每个窗口处理速度根据客户具体业务而定，所有人排队等待叫号即可。若快下班时，告知客户明日再来(**拒绝流量**)。
- 水坝泄洪：水坝可以通过闸门控制泄洪速度(**控制处理速度**)。

以上”限流”例子，可以让服务提供者稳定的服务客户。

## Nginx 限流

Nginx 提供两种限流方式，一是控制速率，二是控制并发连接数。

### 控制速率

#### 正常限流

[ngx_http_limit_req_module](http://nginx.org/en/docs/http/ngx_http_limit_req_module.html) 模块提供限制请求处理速率能力，使用了漏桶算法(leaky bucket)。下面例子使用 nginx limit_req_zone 和 limit_req 两个指令，限制单个IP的请求处理速率。

在 **nginx.conf** http 中添加限流配置：

> 格式：limit_req_zone key zone rate

```
    http {
        limit_req_zone $binary_remote_addr zone=myRateLimit:10m rate=10r/s;
    }
```

配置 **server**，使用 **limit_req** 指令应用限流。

```
    server {
        location / {
            limit_req zone=myRateLimit;
            proxy_pass http://my_upstream;
        }
    }
```

- **key** ：定义限流对象，**binary_remote_addr** 是一种key，表示基于 **remote_addr(客户端IP)** 来做限流，**binary_** 的目的是压缩内存占用量。
- **zone**：定义共享内存区来存储访问信息， **myRateLimit:10m** 表示一个大小为10M，名字为myRateLimit的内存区域。1M能存储16000 IP地址的访问信息，10M可以存储16W IP地址访问信息。
- **rate** 用于设置最大访问速率，**rate=10r/s** 表示每秒最多处理10个请求。Nginx 实际上以毫秒为粒度来跟踪请求信息，因此 10r/s 实际上是限制：**每100毫秒处理一个请求**。这意味着，自上一个请求处理完后，若后续100毫秒内又有请求到达，将拒绝处理该请求。

#### 处理突发流量

上面例子限制 10r/s，如果有时正常流量突然增大，超出的请求将被拒绝，无法处理突发流量，可以结合 **burst** 参数使用来解决该问题。

```
    server {
        location / {
            limit_req zone=myRateLimit burst=20;
            proxy_pass http://my_upstream;
        }
    }
```

**burst** 译为突发、爆发，表示在超过设定的处理速率后能额外处理的请求数。当 **rate=10r/s** 时，将1s拆成10份，即每100ms可处理1个请求。

此处，**burst=20** ，若同时有21个请求到达，Nginx 会处理第一个请求，剩余20个请求将放入队列，然后每隔100ms从队列中获取一个请求进行处理。若请求数大于21，将拒绝处理多余的请求，直接返回503.

不过，单独使用 **burst** 参数并不实用。假设 **burst=50** ，rate依然为10r/s，排队中的50个请求虽然每100ms会处理一个，但第50个请求却需要等待 50 * 100ms即 5s，这么长的处理时间自然难以接受。

因此，**burst** 往往结合 **nodelay** 一起使用。

```
    server {
        location / {
            limit_req zone=myRateLimit burst=20 nodelay;
            proxy_pass http://my_upstream;
        }
    }
```

**nodelay** 针对的是 burst 参数，**burst=20 nodelay** 表示这20个请求立马处理，不能延迟，相当于特事特办。不过，即使这20个突发请求立马处理结束，后续来了请求也不会立马处理。**burst=20** 相当于缓存队列中占了20个坑，即使请求被处理了，这20个位置这只能按 100ms一个来释放。

这就达到了速率稳定，但突然流量也能正常处理的效果。

### 限制连接数

[ngx_http_limit_conn_module](http://nginx.org/en/docs/http/ngx_http_limit_conn_module.html) 提供了限制连接数的能力，利用 **limit_conn_zone** 和 **limit_conn** 两个指令即可。下面是 Nginx 官方例子：

```
    limit_conn_zone $binary_remote_addr zone=perip:10m;
    limit_conn_zone $server_name zone=perserver:10m;
    server {
        ...
        limit_conn perip 10;
        limit_conn perserver 100;
    }
```

**limit_conn perip 10** 作用的key 是 **$binary_remote_addr**，表示限制单个IP同时最多能持有10个连接。

**limit_conn perserver 100** 作用的key是 **$server_name**，表示虚拟主机(server) 同时能处理并发连接的总数。

需要注意的是：只有当 **request header** 被后端server处理后，这个连接才进行计数。

### 设置白名单

限流主要针对外部访问，内网访问相对安全，可以不做限流，通过设置白名单即可。利用 Nginx [ngx_http_geo_module](http://nginx.org/en/docs/http/ngx_http_geo_module.html) 和 [ngx_http_map_module](http://nginx.org/en/docs/http/ngx_http_map_module.html) 两个工具模块即可搞定。

在 **nginx.conf** 的 **http** 部分中配置白名单：

```
    geo $limit {
        default 1;
        10.0.0.0/8 0;
        192.168.0.0/24 0;
        172.20.0.35 0;
    }
    map $limit $limit_key {
        0 "";
        1 $binary_remote_addr;
    }
    limit_req_zone $limit_key zone=myRateLimit:10m rate=10r/s;
```

**geo** 对于白名单(子网或IP都可以) 将返回0，其他IP将返回1。

**map** 将 **$limit** 转换为 **$limit_key**，如果是 **$limit** 是0(白名单)，则返回空字符串；如果是1，则返回客户端实际IP。

**limit_req_zone** 限流的key不再使用 **$binary_remote_addr**，而是 **$limit_key** 来动态获取值。如果是白名单，limit_req_zone 的限流key则为空字符串，**将不会限流**；若不是白名单，将会对客户端真实IP进行限流。

## 拓展阅读

除限流外，[ngx_http_core_module](http://nginx.org/en/docs/http/ngx_http_core_module.html#limit_rate_after) 还提供了限制数据传输速度的能力(即常说的下载速度)。

例如：

```
    location /flv/ {
        flv;
        limit_rate_after 20m;
        limit_rate       100k;
    }
```

这个限制是针对每个请求的，表示客户端下载前20M时不限速，后续限制100kb/s。

## 参考

- Nginx官方博客：[Rate Limiting with NGINX and NGINX Plus](https://www.nginx.com/blog/rate-limiting-nginx/)
- DZone：[NGINX Rate Limiting](https://dzone.com/articles/nginx-rate-limiting)
- [Nginx 中如何获取真实IP？](https://chenyongjun.vip/articles/78#如何获取真实IP？)
- [浏览器连接限制](https://chenyongjun.vip/articles/80)
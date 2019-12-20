---
title: nginx反向代理的路径问题和负载均衡配置
date: 2019-12-20 09:03:28
categories: 学习
tags: [nginx]
---
## 最近一次项目上线过程中，配负载均衡的同时，反向代理路径搞错了，特此记录！！！ ##

### proxy_pass 路径问题
`proxy_pass http://localhost:8080`和`proxy_pass http://localhost:8080/`(多了末尾的/)是不同的的处理方式，而proxy_pass `http://localhost:8080/`和`proxy_pass http://localhost:8080/abc`是相同的处理方式。

**总结一下:**
```
server {
  listen       80;
  server_name  localhost;

  location /api1/ {
    proxy_pass http://localhost:8080;
  }
  # http://localhost/api1/xxx -> http://localhost:8080/api1/xxx

  location /api2/ {
    proxy_pass http://localhost:8080/;
  }
  # http://localhost/api2/xxx -> http://localhost:8080/xxx

  location /api3 {
    proxy_pass http://localhost:8080;
  }
  # http://localhost/api3/xxx -> http://localhost:8080/api3/xxx

  location /api4 {
    proxy_pass http://localhost:8080/;
  }
  # http://localhost/api4/xxx -> http://localhost:8080//xxx，请注意这里的双斜线，好好分析一下。

  location /api5/ {
    proxy_pass http://localhost:8080/haha;
  }
  # http://localhost/api5/xxx -> http://localhost:8080/hahaxxx，请注意这里的haha和xxx之间没有斜杠，分析一下原因。

  location /api6/ {
    proxy_pass http://localhost:8080/haha/;
  }
  # http://localhost/api6/xxx -> http://localhost:8080/haha/xxx

  location /api7 {
    proxy_pass http://localhost:8080/haha;
  }
  # http://localhost/api7/xxx -> http://localhost:8080/haha/xxx

  location /api8 {
    proxy_pass http://localhost:8080/haha/;
  }
  # http://localhost/api8/xxx -> http://localhost:8080/haha//xxx，请注意这里的双斜杠。
}
```
### 负载均衡配置
1、upstream是关键字必须要有，后面的server_pool为一个Upstream集群组的名字，可以自定义；
2、server是关键字固定，后面可以接域名或IP。如果不指定端口，默认是80。结尾有分号。
```
http{
  upstream server_pool {
    server 192.68.0.14:80;
    server 192.68.0.15:80;
  }

  server {
    listen       80;
    server_name  localhost;

    location /api {
      proxy_pass http://backserver;
    }
  }
}
```
#### 1.轮询（默认）
#### 2.权重
weight和访问比率成正比，用于后端服务器性能不均的情况。权重越高，在被访问的概率越大
```
upstream server_pool {
  server 192.68.0.14:80 weight=5;
  server 192.68.0.15:80 weight=10;
}
```
#### 3.ip_hash
如果客户已经访问了某个服务器，当用户再次访问时，会将该请求通过哈希算法，自动定位到该服务器。每个请求按访问ip的hash结果分配，这样每个访客固定访问一个后端服务器，可以解决session的问题。
**注意：当负载调度算法为ip_hash时，后端服务器在负载均衡调度中的状态不能是weight和backup。**
```
upstream server_pool {
  ip_hash;
  server 192.68.0.14:80;
  server 192.68.0.15:80;
}
```
#### 4.fair
根据后端服务器的响应时间进行分配，响应快的优先分配请求
```
upstream server_pool {
  server 192.68.0.14:80;
  server 192.68.0.15:80;
  fair;
}
```
#### 较完整的一份配置
```
upstream server_pool {
    server 10.0.0.6：80 weight=1 max_fails=1  fails_timeout=10s;
    server 10.0.0.7：80 weight=1 max_fails=2  fails_timeout=10s backup;
    server 10.0.0.8：80 weight=1 max_fails=3  fails_timeout=20s backup;
}
```
1. server 10.0.0.6:80:  
  负载均衡后面的RS配置，可以是IP或域名，如果不写端口，默认是80端口。高并发场景下，IP可换成域名，通过DNS做负载均衡
2. weight=1
  代表服务器的权重，默认值是1。权重数字越大表示接受的请求比例越大。
3. max_fails=1
  Nginx尝试连接后端主机失败的次数，max_fails的默认值是1；企业场景：建议2-3次。
4. backup
  这标志着这个服务器作为备份服务器，若主服务器全部宕机了，就会向他转发请求；注意：当负载调度算法为ip_hash时，后端服务器在负载均衡调度中的状态不能是weight和backup。
5. fail_timeout=10s
  在max_fails定义的失败次数后，距离下次检查的间隔时间，默认是10s；如果max_fails是5，他就检测5次。如果5次都是502，那么他就会根据fail_timeout的值，等待10s再去检查，还是只检查一次，如果持续502，在不重新加载nginx配置的情况下，每隔10s都只检测一次。常规业务：2-3秒比较合理。
6. down
  这标识着服务器永远不可用，这个参数可配合ip_hash使用。
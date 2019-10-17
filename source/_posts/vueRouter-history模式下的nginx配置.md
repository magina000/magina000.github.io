---
title: vueRouter history模式下的nginx配置
date: 2019-10-16 15:55:51
categories: 学习
tags: [vue]
---
# 主要是Vue路由history模式的踩坑记录，记录下来。

## hash模式下的配置
```
//html访问路径
location / {
    root  /web/project/**;  // 本地项目的路径
    index index.html index.htm;
}
//api接口代理
location /api/ {
    proxy_pass http://192.168.1.1:8080/;（后端接口地址,默认端口号80可以不加）
}
```
<font size=3>**注意:**</font>
<font size=2 color=#2b91af>*如果出现404*</font>
1.proxy_pass 地址后面要不要加/取决于匹配的 /api/ 作不作为你uri的一部分，如果 /api/ 是其中一部分,则不需要带上 / ； 反之带上。加了 / 相当于是绝对根路径，nginx 不会把location 中匹配的路径 /api/ 带上。
2.proxy_pass的地址记得在hosts文件做ip映射，建议直接使用域名对应的ip地址。
3.location 中 ~ （区分大小写）与 ~* （不区分大小写）标识均为正则匹配。如果你不确定，请在location后面加上 location ~* /api/ { }这样的配置 不区分 api三个字母的大小写。

## history模式下的配置
### 问题
在进行项目的主页的时候，一切正常，可以访问，但是当刷新页面或者直接访问路径的时候就会返回404，因为在history模式下，只是动态的通过js操作window.history来改变浏览器地址栏里的路径，并没有发起http请求，但是直接在浏览器里输入这个地址的时候，就一定要对服务器发起http请求，但是这个目标在服务器上又不存在，所以会返回404

**解决方案**
如果项目是根路径,补充try_files $uri $uri/ /index.html;
```
location / {
　　root   /web/project/**;  // 本地项目的路径
　　index  index.html index.htm;
　　try_files $uri $uri/ /index.html;
}
```
如果项目还带项目名称目录,比如项目访问地址 http://192.168.1.1:8080/project/
要注意项目打包后静态资源的路径，修改index.js中build的`assetsPublicPath:"/project/"`
```
location /project {
  root   /web/project/**;  // 本地项目的路径
　index  index.html index.htm;
  if (!-e $request_filename) {
    rewrite ^(.*) /index.html last;
    break;
  }
}
```




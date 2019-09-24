---
title: vue中axios请求封装
date: 2019-09-23 16:45:19
categories: 学习
tags: [vue]
---

## axios的封装和api接口的统一管理，主要目的是简化代码和利于后期的更新维护。

默认已经安装axios

### 引入
在项目的src目录中，新建一个api文件夹，然后在里面新建一个http.js，index.js文件。http.js文件用来封装axios，index.js用来统一管理接口
```
// 在http.js中引入axios
import axios from 'axios'; // 引入axios
import QS from 'qs'; // 引入qs模块，用来序列化post类型的数据，后面会提到
// ElementUI的message提示框组件，可根据使用的ui组件更改。
import { Message } from 'element-ui'; 
```

### 环境切换
通过node的环境变量来匹配默认的接口url前缀。axios.defaults.baseURL设置axios的默认请求地址。
```
//环境切换(主要是开发和生产环境) e.g.
if (process.env.NODE_ENV == 'development') {    
  axios.defaults.baseURL = '/api';} 
else if (process.env.NODE_ENV == 'production') {    
  axios.defaults.baseURL = '';
}
```

### 创建实例并设置请求超时
```
var instance = axios.create({timeout: 1000 * 12});
```

### POST请求头的设置
```
instance.defaults.headers.post['Content-Type'] = 'application/x-www-form-urlencoded;charset=UTF-8';
```

### 请求拦截
在发送请求前可以进行一个请求的拦截，为什么要拦截呢，拦截请求是用来做什么的呢？比如，有些请求是需要用户登录之后才能访问的，或者post请求的时候，需要序列化我们提交的数据。这时候，可以在请求被发送之前进行一个拦截，从而进行想要的操作。
```
// 如果用到store缓存登录信息,先导入vuex
import store from '@/store/index';
instance.interceptors.request.use(    
  config => {
    // 每次发送请求之前判断vuex中是否存在token        
    // 如果存在，则统一在http请求的header都加上token，这样后台根据token判断你的登录情况
    // 即使本地存在token，也有可能token是过期的，所以在响应拦截器中要对返回状态进行判断
    const token = store.state.token;        
    token && (config.headers.Authorization = token);        
    return config;    
  },    
  error => {        
    return Promise.error(error);    
  })
```
补充：说一下token，一般是在登录完成之后，将用户的token通过localStorage或者cookie存在本地，然后用户每次在进入页面的时候（即在main.js中），会首先从本地存储中读取token，如果token存在说明用户已经登陆过，则更新vuex中的token状态。然后，在每次请求接口的时候，都会在请求的header中携带token，后台人员就可以根据你携带的token来判断你的登录是否过期，如果没有携带，则说明没有登录过。

### 响应拦截
将服务器返回给我们的数据进行处理。如果后台返回的状态码是200，则正常返回数据，否则根据错误的状态码类型进行一些需要的错误捕捉提示，同时包括没登录或登录过期后重定向登录页。
*报错提示*
```
const tip = msg => {    
  Message({        
      message: msg,        
      duration: 1000  
  });
}
```
*跳转登录页*
```
const toLogin = () => {
  router.replace({
    path: '/login',        
    query: {
        redirect: router.currentRoute.fullPath
    }
  });
}
```
*错误状态码捕捉*
```
const errorHandle = (status, message) => {
  // 状态码判断
  switch (status) {
    // 401: 未登录状态，跳转登录页
    case 401:
      toLogin();
      break;
    // 403 token过期
    // 清除token并跳转登录页
    case 403:
      tip('登录过期，请重新登录');
      localStorage.removeItem('token');
      store.commit('loginSuccess', null);
      setTimeout(() => {
          toLogin();
      }, 1000);
      break;
    // 404请求不存在
    case 404:
      tip('请求的资源不存在'); 
      break;
    default:
      console.log(message);   
  }
}
```
<font size=4 >**完整拦截**</font>
```
// 响应拦截器
instance.interceptors.response.use(    
  // 请求成功
  res => res.status === 200 ? Promise.resolve(res) : Promise.reject(res),    
  // 请求失败
  err => {
    const { response } = err;
    if (response) {
      // 请求已发出，但是不在2xx的范围 
      errorHandle(response.status, response.data.message);
      return Promise.reject(response);
    } else {
      // 处理断网的情况
      // eg:请求超时或断网时，更新state的network状态
      // network状态在app.vue中控制着一个全局的断网提示组件的显示隐藏
      // 关于断网组件中的刷新重新获取数据，会在断网组件中说明
      if (!window.navigator.onLine) {
          store.commit('changeNetwork', false);
      } else {
          return Promise.reject(err);
      }
    }
  }
)
//拦截器完成后即可抛出axios
export default instance;
```

### api管理
index.js中
```
import axios from '@/api/http'; // 导入http中创建的axios实例
import qs from 'qs'; // 导入qs模块进行请求参数格式化，或者根据需求自定义
//get，post本就是promise，不需要过度封装直接使用
//e.g.
const getDemo = (params) {        
  return axios.get(`url`, {
    params: params
  }).then(
    res => res.data;
  );    
}, 
const postDemo = (params) {
  return axios.post(`url`, 
    qs.stringify(params)
  ).then(
    res => res.data;
  )
}
export default{
  getDemo,
  postDemo
}
```

### 页面中调用
为了方便api的调用，需要将其挂载到vue的原型上。在main.js中：
```
import api from './api' // 导入api接口
Vue.prototype.$http = api; // 将api挂载到vue的原型上
```
页面中调用
```
methods: {    
  func() {      
    this.$http.getDemo({        
      param: 123      
    }).then(res=> {
      //执行某些操作      
    })    
  }  
}
```
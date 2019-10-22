---
title: vue+axios实现登录拦截
date: 2019-10-15 13:50:22
categories: 学习
tags: [vue]
---

# vue+axios 前端实现登录拦截（包括路由拦截、http拦截）

<font size=3>**整体思路：**</font>
1. 登录时，判断登录状态（从session中拿登录信息），（若没有登录信息）发送登录请求，成功后将后台返回的登录用户信息（token）存在sessionStorage中。
2. 接口请求时，在axios请求拦截器中给http头携带token发送给后台，用于验证登录状态是否过期（request interceptors）。
3. 请求时token如果过期，清除sessionstorage的用户信息，并重定向到登录页。
4. 路由跳转时，利用vue-router提供的钩子函数beforeEach()对路由进行判断，非登录页且存在token就next()，不存在token便跳转到登录页面。

## vuex
*先把要管理的登录状态写完整(store->index.js)*
```
import Vue from "vue";
import Vuex from "vuex";

Vue.use(Vuex);
export default new Vuex.Store({
  state: {
    token: null，    //登录信息(也可以是登录用户信息，由后台定)
    isLogin: false,  //登录状态
  },
  getters:{
    //获取登录状态
    isLogin(state){
      return state.isLogin
    }
  },
  mutations: {
    Login(state, data){
      sessionStorage.setItem('token',data);
      state.isLogin = true;
      state.token = data;
    },
    Logout(state){
      sessionStorage.removeItem('token');
      state.isLogin = false;
      state.token = null;
    }
  }
})
```

## 登录
<font size=3> **Login.vue** </font>
```
//判断是否登录
methods:{
  isLogin(){
    //判断是否由登录信息
    if(session.getItem('token')){
      this.$store.commit('Login', session.getItem('token'));
    }else{
      this.$store.commit('Logout');
    }
    return this.$store.getters.isLogin;
  },
  //login接口请求登录信息,存储token
  login(){
    this.$http.post('/api/login').then(res => {
      this.$store.commit('Login', res.token);
      // index为首页，或者重定向传过来的跳转页面
      let redirect = decodeURIComponent(this.$route.query.redirect || '/index');
      this.$router.push({
        path: redirect
      })
    }).catch(err=>{
      //登出
      this.$store.commit('Logout');
    })
  }
  })
}
```
## http拦截
<font size=3> **axios封装** </font>
```
//请求拦截
axios.interceptors.request.use(config => {
  //存在token，写入请求头
  if(store.state.token){
    config.headers.Authorization = `${store.state.token}`
  }
  return config
})
//返回拦截
axios.interceptors.response.use(
  response => {
    return response;
  },
  err => {
    if(err.response){
      switch (error.response.status) {
        // 401: 未登录
        // 未登录则跳转登录页面，并携带当前页面的路径
        case 401:
            router.replace({                        
                path: '/login',                        
                query: { 
                    redirect: router.currentRoute.fullPath 
                }
            });
            break;
        // 403 token过期
        // 登录过期对用户进行提示
        // 清除本地token和清空vuex中token对象
        // 跳转登录页面
        case 403:
            store.commit('Logout');
            router.replace({                            
                path: '/login',                            
                query: { 
                    redirect: router.currentRoute.fullPath 
                }                        
            });     
      }
      return Promise.reject(error.response.data)   // 返回接口返回的错误信息
    }
})
```
## 路由拦截
<font size=3> **router>index.js** </font>
```
//路由跳转之前
//如果token不存在且不是登录页，跳转登录页
//如果有跳转地址要默认登录，就将路由当参数传到登录页
router.beforeEach((to, from, next) => {
  if (to.path !== '/login' && !store.state.token) {
    next({                            
      path: '/login',                            
      query: { 
          redirect: to.fullPath 
      }                        
    })
  }
   next()
})
```
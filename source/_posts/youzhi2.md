---
title: youzhi2
date: 2019-07-19 18:47:32
tags:
---

### 通过一个实际案例彻底理解 promise

- [前言](#_1)
- [案例](#_9)
- [案例分析](#_16)
- [通过 Promise 对案例进行优化](#_Promise__70)
- [总结](#_157)

# 前言

JavaScrip 是单线程，前端在大部分情况下写出来的 JS 代码都是同步执行。偶尔会通过回调函数达到异步执行的目的。后来有了 promise 对象解决异步问题，被纳入到 ES6 标准中。

但是很多人都不理解 promise，或者一知半解，感觉懂又好像不懂，最主要的是在实际项目中用得少，不能够学以致用，融会贯通。但也正是因为不能够真正理解所以也不敢贸然用在项目中，于是两者陷入循环，导致很多人对这个既能够提高代码质量，又在一定程度上提升性能的对象弃之不用。

我从前对 promise 的印象也是停留在以上阶段，直到今天，在一个实际开发案例中运用到了 promise ，让我对 promise 有了深刻的理解，在此分享给大家。

# 案例

![在这里插入图片描述](https://img-blog.csdnimg.cn/20190719154134495.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3dlaXhpbl80NDEzNTEyMQ==,size_16,color_FFFFFF,t_70)

在以上的页面中，有一个点赞功能，是上级看过记录之后进行评价的反馈。
**上级打开下属的记录详情之后，涉及到的逻辑如下**：

1. 调登陆接口，拿到上级userId。
2. 调点赞列表接口。
3. 通关上级的 userId，与点赞列表接口里面对应的点赞  id 比较，如果上级点过赞，则点赞的图标是实心。否则是空心。

# 案例分析

实现以上功能需要调两个接口。首先调登陆接口,拿到 userId，然后在回调中再调点赞列表接口，循环列表，拿到点赞 id ，与 userId 做比较。代码如下：

第一步：调用登陆接口，拿到 userId

    ```
        function getUserId(cb){
        axios.post(Config.serverUrl + "/login/login", 
        {code: result.code,
        url:url,}, {
        headers: {'Content-Type': 'application/json'}
        }).then(res => {
        if (res.data.isSuc == true) {
        sessionStorage.setItem(Config.SESSION_DING_USER,JSON.stringify(res.data.result));
        let s = sessionStorage.getItem(Config.SESSION_DING_USER);
        let user = JSON.parse(s);
        let userId = user && user.userid;
        if(cb){
        cb(userId);
        }
        } else {
              alert(登录失败);}
        })
        }
        
    
    ```

第二步，调用点赞列表接口

    ```
        function getPraiseList(cb){
                    axios.post(Config.serverUrl + "/visitAndSign/getpraiseInfo", {
                        "signId": Number(self.id),
                    }, {
                        headers: {'Content-Type': 'application/json', 'DING_TOKEN': self.token}
                    }).then(function (res) {
                        if (res.data.isSuc) {
                            let list = res.data.result;
                            if(cb){
                                cb(list);
                            }
                        }
                    })
                }
        
    
    ```

第三步，根据回调结果，判断是否点过赞

    ```
        getUserId(function (userId){
          getPraiseList(function(list){
           for (var i=0;i<list.length;i++){
                    if(list[i].praiseUserid == userId){
                      self.isPraise = 0;
                      break;
                  }
            }
        })
        })
        
    
    ```

# 通过 Promise 对案例进行优化

以上通过两层回调可以实现业务逻辑，咋一看没什么大问题，但是如果业务还没完，假设又增加一个需求，根据点赞状态判断是否调用另一个接口。这样就会陷入回调地狱，回调地狱主要是指回调太多，代码冗长，到最后很有可能你都不知道那个回调对应的那个参数了。

为了优雅的解决以上问题，Promise 对象闪亮登场，现在使用 Promise 实现以上功能。并且假设得到是否点赞状态之后还要调第三个接口。

**第一步优化，使用 Promise 对象和 .then() 方法**

    ```
        var promise = new Promise(function(resolve,reject){
        axios.post(Config.serverUrl + "/login/login", 
        {code: result.code,
        url:url,}, {
        headers: {'Content-Type': 'application/json'}
        }).then(res => {
        if (res.data.isSuc == true) {
        sessionStorage.setItem(Config.SESSION_DING_USER,JSON.stringify(res.data.result));
        let s = sessionStorage.getItem(Config.SESSION_DING_USER);
        let user = JSON.parse(s);
        let userId = user && user.userid;
        resolve(userId);
        }})
        })
        
        promise.then(function(userId){
        axios.post(Config.serverUrl + "/visitAndSign/getpraiseInfo", {
                        "signId": Number(self.id),
        }, {
        headers: {'Content-Type': 'application/json', 'DING_TOKEN': self.token}
                    }).then(function (res) {
                        if (res.data.isSuc) {
                            let list = res.data.result;
                            for (var i=0;i<list.length;i++){
                           if(list[i].praiseUserid == userId){
                           self.isPraise = 0;
                           break;
                           }
                          }
                          return self.isPraise;
                        }
                    })
        })
        .then(function(res){
        console.log(res)//点赞状态实现结果，这里可以继续调第三个接口
        })
        //后面如果还有回调继续用.then()即可。
        
    
    ```

**第二步优化，使用 Promise.all 优化性能**

虽然通过 Promise 对象解决了代码展示问题。还有一个问题，等到登陆接口调成功之后再调点赞列表接口。但是实际上点赞列表接口并不依赖于登陆接口。所以登陆接口和点赞列表接口可以同时调，加快接口返回速度，但问题是，做最后的对比处理（点赞列表编号与用户编号对比来判断用户是否点过赞）需要在两个接口都成功返回才可。所以这时候就用上了 Promise.all。

Promise.all 接收一个数组，而数组中就是每一个 Promise 对象。假设 Promise.all 的返回结果定义为 pro。pro.then()就是两个接口都调成功之后的回调。

对以上代码进一步改造如下：

    ```
        var promise1 = new Promise(function(resolve,rejuect){
        axios.post(Config.serverUrl + "/login/login", 
        {code: result.code,
        url:url,}, {
        headers: {'Content-Type': 'application/json'}
        }).then(res => {
        if (res.data.isSuc == true) {
        sessionStorage.setItem(Config.SESSION_DING_USER,JSON.stringify(res.data.result));
        let s = sessionStorage.getItem(Config.SESSION_DING_USER);
        let user = JSON.parse(s);
        let userId = user && user.userid;
        resolve(userId);
        }})
        })
        var promise2 = new Promise(function(resolve,rejuect){
        axios.post(Config.serverUrl + "/visitAndSign/getpraiseInfo", {
                        "signId": Number(self.id),
        }, {
        headers: {'Content-Type': 'application/json', 'DING_TOKEN': self.token}
                    }).then(function (res) {
                        if (res.data.isSuc) {               
                          resolve(res.data.result);
                        }
                    })
        })
        })
        var pro = Promise.all([promise1,promise2]);
        pro.then(function(userId,list){
             for (var i=0;i<list.length;i++){
            if(list[i].praiseUserid == userId){
            self.isPraise = 0;
                break;
             }
         }
        })
        
    
    ```

# 总结

1. Promise 对象用法

    ```
        var  promise = new Promise(function(resolve,rejuct){
        if(){
        //异步方法调用成功
           resolve();
        }else{
        //异步方法调用失败
           rejuct();
        }
        })
        promise.then(function(){
        
        })
        
    
    ```

举例：

    ```
            var  promise = new Promise(function(resolve,reject){
                setTimeout(function (res) {
                    resolve(res);
                },2000,1);
            })
            promise.then(function(id){
                console.log(id)//1
            })
        
    
    ```

1. Promise.all() 用法

    ```
        var  promise1 = new Promise(function(resolve,rejuct){
        if(){
        //异步方法调用成功
           resolve();
        }else{
        //异步方法调用失败
           rejuct();
        }
        })
        var  promise2 = new Promise(function(resolve,rejuct){
        if(){
        //异步方法调用成功
           resolve();
        }else{
        //异步方法调用失败
           rejuct();
        }
        })
        var pro = Promise.all([promise1,promise2])
        pro.then(function([a,b]){
           //在这里a和b都可以拿到
        })
        
    
    ```

举例：

    ```
        var  promise1 = new Promise(function(resolve,reject){
                setTimeout(function (res) {
                    resolve(res);
                },2000,1);
            })
            var  promise2 = new Promise(function(resolve,reject){
                setTimeout(function (res) {
                    resolve(res);
                },3000,2);
            })
            var pro = Promise.all([promise1,promise2]);
            pro.then(function([a,b]){
                console.log(a+b)//3
            })
        
    
    ```

1. 虽然不使用一些新语法和高阶用法也能实现一些功能，但是新语法和用法的诞生必定是解决了某个问题或提升了性能。所以在平时项目中还是要经常将一些高阶用法运用进去，如果不去实践，有些概念很难真正理解。


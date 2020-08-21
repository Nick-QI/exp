---
title: 日常记录
---


## 禁止字体跟随系统字体大小

### 即，固定字体大小

1. IOS

   ```css
   body {
     -webkit-text-size-adjust: 100% ;
   }
   ```

   

2. Android 微信端

   ```js
   (function() {
       if (typeof WeixinJSBridge == "object" && typeof WeixinJSBridge.invoke == "function"){
           handleFontSize();
       } else {
           if (document.addEventListener) {
               document.addEventListener("WeixinJSBridgeReady", handleFontSize, false);
           } else if (document.attachEvent) {
               document.attachEvent("WeixinJSBridgeReady", handleFontSize);
               document.attachEvent("onWeixinJSBridgeReady", handleFontSize);
           }
       }
       function handleFontSize() {
           WeixinJSBridge.invoke('setFontSizeCallback', { 'fontSize' : 0 });
           WeixinJSBridge.on('menu:setfont', function() {
               WeixinJSBridge.invoke('setFontSizeCallback', { 'fontSize' : 0 });
           });
       }
   })();
   ```




## 软键盘处理 

1. ios 下，input onblur后，软键盘收起，但是界面无法回弹至100vh

   ```js
   // 监听onblur事件，塞入400ms定时器.(一般软键盘收起 耗时400ms)
   input.onblur = () => {
     setTimeout(()=> {
       window.scrollTo(0, 0)
   	}, 400)  
   }
   ```

   :::tip 

   通过scrollTo 模拟用户滚动，让页面重归正常

   :::



## app和webview 交互

交互方式主要有**两种**：[点我了解更多](https://mp.weixin.qq.com/s/9YkBGfOMvSHr_alpMJUY2Q)

1. webview通过创建一个iframe，通过修改iframe的src 传递指令和参数；app则通过监听iframe的路由变化来调用原生指令

   [详情了解](https://github.com/lzyzsd/JsBridge)

   [部分坑点](https://github.com/lzyzsd/JsBridge/issues/119)

   

2. android和ios分别通过window中埋入函数来调用

   :::details 实例

   ```js
   function callApp(command,data){
     // 统一传递数据类型为 string，java，oc 强类型语言。让客户端自己序列化
      try {
       data = JSON.stringify(data)
     } catch (e) {}
     /* android 和 ios 调用方法是不一样的
    		android: 在window中写入一个独有变量，将指令存入其中
    		ios：通过window.webkit 来调用
    */
     if (window.webkit) {
       window.webkit.messageHandlers[command].postMessage(data)
     } else {
       window.android[command](data)
     }
   }
   ```

   

   :::

|  方式  |                             弊端                             |                解决                |
| :----: | :----------------------------------------------------------: | :--------------------------------: |
| iframe | 1.监听冲突，app必须通过路由监听来处理，<br />在某些特殊情况下（app内webview需要跳转app页面，window.location.href = '/login' 等app内部页面时）引起冲突<br />2.iframe注册回调有延迟<br />3.小程序中加载该webview前需要将该iframe禁用（非业务域名） | 改window；跳转原生页面用指令来跳转 |
| window |         无法回调，调取app指令，app无法return数据回来         |              改iframe              |



### 思考下来：iframe 的方式有很大局限性。决定依据window 重新写一套

暂时的思路： 

​		window目前发现的局限性就是无法回调。那么我主动注册一个回调函数不就行了，在每次app收到指令后，指令执行完成，在执行window上我注册的回调。

::: details 简单理想实现

```js
// 简单实现
import uuid from 'uuid'
const APP = 'app'  // window中存app指令的key
const RECEIVE_APP_DATA = 'h5_callback'  // 注册的回调函数 key

function callApp({command, data, cbKey, cb}) {
  regsiterCommand(cbKey, cb) // 注册回调函数
 
  // 固定格式传参格式，data为数据，cbKey为执行回调函数name
	let body = {
		data,
    cbKey
  }
  try {
    body = JSON.stringify(body)
  }catch(e) {}
  
   if (window.webkit) {
    window.webkit.messageHandlers[command].postMessage(body)
  } else {
    window.android[command](body)
  }
}

// 保持注册指令唯一性，防止指令注册重复
function regsiterCommand(command, cb) {
  const _uuid = uuid()
  command = command ? command + _uuid : _uuid
  window[RECEIVE_APP_DATA][command] = cb
}

/*-----------  预想中的执行 --------------*/

const receiveUserInfo = data => {
  console.log('接收到的用户信息'， data)
}

const info = {
  command: 'getUserInfo',
  data: '',
  cbKey: '',
  cb: receiveUserInfo
}

callApp(info)

```
:::


## 微信ios端底部导航栏

> <img src="https://raw.githubusercontent.com/Nick-QI/pics/master/1597832370146-1597832370128.png" alt="图" style="zoom:20%;" />

**出现原因：** 当存在历史记录栈的时候, ios 底部会出现导航bar

**导航栏隐藏条件：** 微信端会监听整个 scrollview (**整个 html**)的滚动,当导航 bar 存在的时候,用户上滑 scrollview 的时候, 导航 bar 会隐藏, 下滑的时候, 显示导航 bar

**解决方案：**

1. 既然是存在历史记录的时候存在,那么我们每次跳转直接 replace 即可解决
2. 所有的页面高度由dom 将 html 撑高, 不要将页面设置为固定宽高, 这样在导航bar存在的时候, 上滑下滑会触发导航 bar 的自动显隐

​	

## 微信js-sdk，慢慢啃出来的经验

> [自己写的工具类，里面整合了微信js配置，打开即食](https://www.npmjs.com/package/utils94)
>
> [微信js-sdk文档](https://developers.weixin.qq.com/doc/offiaccount/OA_Web_Apps/JS-SDK.html)

![重点](https://raw.githubusercontent.com/Nick-QI/pics/master/1597832346227-1597832346222.png)

**根据上面的信息和踩过的坑，总结出：** 

1. **所有需要使用JS-SDK的页面必须先注入配置信息，否则将无法调用**
2. **同一个url仅需调用一次，对于变化url的SPA的web app可在每次url变化时进行调用,目前Android微信客户端不支持pushState的H5新特性，所以使用pushState来实现web app的页面会导致签名失败，此问题会在Android6.2中修复**
3. **<font color='red'>在ios中,初始配置一次之后即可通用使用(ios 中请求的url需要初次进入的url来获取签名,否则会出错)</font>**
4. **<font color='red'>在安卓中,需要在每次路由变化时重新配置</font>**

:::danger 谨记！！！

ios 在首次进入的时候，要保存初次进入的url，用这个url去请求jsconfig，不然永远都配置不成功；

之前我在写angular的时候，因为初次进入登录会将token附加在url上，然后我用了个守卫将url上的token过滤掉，然后用过滤掉的url去请求jsconfig，永远都成功不了。

:::



## IOS中new Date的坑

![safari表现](https://raw.githubusercontent.com/Nick-QI/pics/master/1597832322393-1597832322384.png)

```js
new Date('2020-12-11 08:09') // 这种带时间的在safari中是无法解析的
new Date('2020-12-11')  // 这样的就可以
```



## 获取url参数的三种方式

1.**qs**

```js
import qs from 'qs'
const search = window.location.href.split('?')[1]
qs.parse(search) // {key:value}
```

---

2.**url**

```js
import url from 'url'
const res = url.parse(window.location.href, true)
/* 
	res = { 
    auth: null
    hash: null
    host: "www.baidu.com"
    hostname: "www.baidu.com"
    href: "https://www.baidu.com/?a=1&a=2&b=2"
    path: "/?a=1&a=2&b=2"
    pathname: "/"
    port: null
    protocol: "https:"
    query: {a: Array(2), b: "2"}
    search: "?a=1&a=2&b=2"
    slashes: true
  }
*/

```

---

3.**URLSearchParams**

```js
 const res = new URLSearchParams(window.location.search)
 const query = {}
 for (let key of res.keys()) {
   query[key] = res.getAll(key)
 }
```



## webview 跳转 app

:::tip

ios android 均支持scheme协议跳转；

微信中禁止了scheme协议和ios universal link，除非加入腾讯白名单，才能直接打开。

最近微信平台开放了网页跳转app功能，[点我知道](https://developers.weixin.qq.com/doc/oplatform/Mobile_App/WeChat_H5_Launch_APP.html)

> 2020-8-5

:::

1. **通过scheme url 协议跳转**

   <font color='red'> ios 最新safari浏览器转入后台后js还会继续执行，所以我们初始方案（先跳scheme协议，2s后跳app store 方案失败），会出现跳转了app后，再次跳转app store。<br>ios 推荐使用universal link跳转，直接跳转，不需要确认框 </font>

   ```js
   window.location.href = 'weixin://'
   ```

   

2. **ios 通过 universal link 跳转（ios 9以上）**

   ![流程图](https://raw.githubusercontent.com/Nick-QI/pics/master/1597832281067-appjump.jpeg)



## 老生常谈，事件循环 EventLoop

![event loop](https://raw.githubusercontent.com/Nick-QI/pics/master/1597832224340-eventloop.jpeg)



## html2canvas 坑点

1. ios 13 版本中无法生成图片，android 和 pc端均正常，将html2canvas版本降级为 "html2canvas": "1.0.0-rc.4"


## AMD 和 CMD



## Sentry 前端报错监控



## SSR 给我们带来了什么

优化白屏渲染，增加服务器成本





## 函数式编程，是个人都能看懂的函数式编程

::: detail like this

```js
man().eat(food)
```

:::



## 啊哈，微前端，微服务，我究竟是个啥？



## GRAPHQL，前端后端不吵架



## 我是真的懒，单元测试写不写


## PWA香的呀！





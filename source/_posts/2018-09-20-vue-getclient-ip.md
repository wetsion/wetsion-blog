---
title: JS获取客户端本地IP
description: vue前后端分离后，后端Java在使用request.getRemoteAddr获取的永远都是前端服务器的IP
categories:
 - 前端
tags:
 - js
 - vue
---

> vue前后端分离后，后端Java在使用request.getRemoteAddr获取的永远都是前端服务器的IP

在使用vue前后端分离之后，发现后端（Java）在使用request.getRemoteAddr之后获取的永远都是前端所部署的服务器的IP，在翻墙之后找到一段获取客户端浏览器本地的IP

```js
export function getUserIp (onNewIP) {
  // compatibility for firefox and chrome
 var myPeerConnection = window.RTCPeerConnection || window.mozRTCPeerConnection || window.webkitRTCPeerConnection
  var pc = new myPeerConnection({
    iceServers: []
  })
  var noop = function () {}
  var localIPs = {}
  var ipRegex = /([0-9]{1,3}(\.[0-9]{1,3}){3}|[a-f0-9]{1,4}(:[a-f0-9]{1,4}){7})/g

 function iterateIP (ip) {
    if (!localIPs[ip]) {
      onNewIP(ip)
    }
    localIPs[ip] = true
 }
  pc.createDataChannel('')
  // create offer and set local description
 pc.createOffer().then(function (sdp) {
    sdp.sdp.split('\n').forEach(function (line) {
      if (line.indexOf('candidate') < 0) {
        return
 }
      line.match(ipRegex).forEach(iterateIP)
    })
    pc.setLocalDescription(sdp, noop, noop)
  }).catch(function (reason) {
    // An error occurred, so handle the failure to connect
 })
  pc.onicecandidate = function (ice) {
    if (!ice || !ice.candidate || !ice.candidate.candidate || !ice.candidate.candidate.match(ipRegex)) {
      return
 }
    ice.candidate.candidate.match(ipRegex).forEach(iterateIP)
  }
}
```

定义一个工具JS，将方法getUserIp导出使用，参数onNewIP是一个function，这里我在登录的页面中调用该方法，首先定义了个方法，设置IP

```js
setUserIp (ip) {
  this.ip = ip
}
```

再调用getUserIp方法：
```js
getUserIp(this.setUserIp)
```

此时，会调用setUserIp将ip设置为客户端IP，至此就获得了客户端IP，再将IP请求发送给后端
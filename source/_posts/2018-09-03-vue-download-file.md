---
title: Vue下载导出文件
description: Vue下载，将字符串导出成txt或pdf等文本文件
categories:
 - 前端
tags:
 - js
 - vue
---

> Vue下载，将字符串导出成txt或pdf等文本文件

将字符串导出成txt或pdf等文本文件，只需要下面几行代码即可
```js
let aTag = document.createElement('a')
let content = 'this is test'
let blob = new Blob([content])
aTag.download = 'a.txt'
aTag.href = URL.createObjectURL(blob)
aTag.click();
URL.revokeObjectURL(blob)
```
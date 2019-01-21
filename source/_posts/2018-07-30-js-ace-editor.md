---
title: ace-editor前端代码编辑器使用
description: 在前端页面使用ace-editor
categories:
 - 前端
tags:
 - js
---

> 如何在页面使用代码编辑器

在git上将项目下载或者clone下来

进入项目根目录

npm install

node ./Makefile.dryice.js

此时再将项目中build/src文件夹下的ace.js拷进项目中

「ext-language_tools.js」文件是必要的

「mode-」开头的文件是支持的代码类型文件，将需要用到的拷进项目，与ace.js同一级

「theme-」开头的文件是编辑器样式文件，同样将需要的拷进与ace.js同一级

「snippets」文件夹下文件使用的话，在项目中ace.js同级创建「snippets」文件夹，再将需要的拷进去

在html中使用：
> 首先要记得先引入ace.js和ext-language_tool.js

```html
<style>
    #editor {
        /*position: absolute;*/
 width: 100%;
        height: 400px;
    }
</style>
<div id="editor"></div>
var editorInit = function(editorType, contentText) {
    ace.require("ace/ext/language_tools");
    window.editor = ace.edit("editor");
    // editor.$blockScrolling = Infinity;
 editor.getSession().setMode("ace/mode/" + editorType);
    editor.setOptions({
        enableBasicAutocompletion: true,
        enableSnippets: true,
        enableLiveAutocompletion: true
 });
    editor.setTheme("ace/theme/twilight");
    editor.setValue(contentText);
}
```
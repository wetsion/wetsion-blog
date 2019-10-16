---
title: Vue：iView和elementUI嵌套弹窗问题
description: 整理一下遇到的弹窗嵌套的问题
categories:
 - 前端
tags:
 - js
 - vue
---


最近在写一个使用elementUI的Vue前端项目时，有一个需求是在弹窗显示了关联的列表信息后，再点击列表中某条记录，进行弹窗编辑，也就是嵌套弹窗。

其实前阵子在写另一个使用iView的Vue前端项目时，也碰到了类似的问题，这一次就一起整理做下记录。


#### elementUI

对于elementUI，提供的弹窗组件是`<el-dialog></el-dialog>`，想要解决嵌套的问题，即在外层父级`el-dialog`加上属性`:modal-append-to-body="false"`和`append-to-body="true"`，而在内层子级`el-dialog`加上属性`append-to-body="true"`，样例代码如下：


```html
<el-dialog title="父级弹窗" :visible.sync="showParent" :modal-append-to-body="false" append-to-body>
    <el-dialog :title="子级弹窗" :visible.sync="showChild" append-to-body>
    </el-dialog>
</el-dialog>
```


#### iView

对于iView呢，提供的弹窗组件是`<Modal></Modal>`，想要解决嵌套的问题，比elementUI要复杂一些。首先需要知道的是，iView中的Modal默认的`z-index`是1000，而想要嵌套，则后面子级弹窗的`z-index`需要配置大于1000，同时父级弹窗需要配置属性`:transfer="false"`，这样后面子级的弹窗将会覆盖父级弹窗，样例代码如下：


```html
<template>
    <Modal v-model="showParent" :transfer="false" title="父级弹窗">
        <Modal v-model="showChild" title="子级弹窗" class-name="child-modal"></Modal>
    </Modal>
</template>
<style>
    .child-modal {
        z-index: 1002;
    }
</style>
```
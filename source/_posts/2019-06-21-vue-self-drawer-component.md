---
title: Vue：自定义抽屉组件
description: 自定义一个抽屉组件
categories:
 - 前端
tags:
 - js
 - vue
---



这两天在开发某个前端系统时，想要做一个抽屉弹出的功能，但由于当前系统是使用的element-ui，并没有提供抽屉组件，而之前使用的iView则提供了抽屉组件，于是便着手仿照iView的抽屉组件自定义了一个简易的抽屉组件。

 - 定义组件 `dwui-drawer.vue` :

```js
<template>
  <div class="dwui-drawer">
    <div v-if="open"
         :class="computeMode"
         @click="maskClick">
    </div>
    <div  :class="computeContent">
      <slot></slot>
    </div>
  </div>
</template>

<script>
export default {
  name: 'dwui-drawer',
  props: {
    /**
     * 是否点击遮罩层关闭，默认可以
     */
    maskCloseable: {
      type: Boolean,
      default: true
    },
    /**
     * 抽屉出来位置
     */
    position: {
      type: String,
      default: 'right'
    },
    /**
     * 是否打开
     */
    open: {
      type: Boolean
    }
  },
  computed: {
    computeMode () {
      return this.position === 'left' ? (this.open ? 'dwui-drawer-open-mode': 'dwui-drawer-close-mode') : ( this.open ? 'dwui-drawer-open-mode-right': 'dwui-drawer-close-mode-right')
    },
    computeContent () {
      return this.position === 'left' ? (this.open ? 'dwui-drawer-open-content' : 'dwui-drawer-close-content') : (this.open ? 'dwui-drawer-open-content-right' :'dwui-drawer-close-content-right')
    }
  },
  methods: {
    maskClick() {
      if (this.maskCloseable) {
        this.$emit('update:open',false);
        this.$emit('change');
      }
    }
  }
}
</script>

<style scoped lang="less">
  @import "./dwui-drawer";
  .dwui-drawer-open-content,.dwui-drawer-close-content{
    padding: 30px 0px 0px 0px;
  }
  .dwui-drawer-open-content-right,.dwui-drawer-close-content-right{
    padding: 30px 0px 0px 0px;
  }
</style>

```

 - 定义样式文件`dwui-drawer.less`

```css
.dwui-drawer{
  &-close-content,&-open-content, &-close-content-right,&-open-content-right, &-close-mode ,&-open-mode,&-close-mode-right,&-open-mode-right{
    position: fixed;
    top: 0;
    height: 100%;
  }
  &-close-content,&-open-content,&-close-content-right,&-open-content-right{
    width: 10%;
    background: #fff;
    z-index: 99
  }
  &-open-mode,&-close-mode,&-close-mode-right,&-open-mode-right{
    background: #222222;
    opacity: 0.8;
    z-index: 9;
    width: 100%;
    right: 0;
    left: 0;
  }
  &-close-content{
    opacity:0;
    left: 0%;
    transform: translate3d(-100%, 0, 0);
    transition: all .2s;
  }
  &-open-content{
    opacity:1;
    left: 0%;
    transform: translate3d(0, 0, 0);
    transition:all  .2s;
  }
  &-close-content-right{
    right: -50%;
    transition: right .2s;
  }
  &-open-content-right{
    right: 0;
    transition: right .2s;
  }
  &-close-mode{
    opacity:0;
    transition: opacity .2s;
  }
  &-open-mode{
    opacity: 0.7;
    transition: opacity .2s;
  }
  &-close-mode-right{
    opacity:0;
    transition: opacity .2s;
  }
  &-open-mode-right{
    opacity: 0.7;
    transition: opacity .2s;
  }
}
```

 - 使用
 
```js
<dwui-drawer :open.sync="showRight" :mask-closeable="true" position="right">
    <p>xxxx</p>
</dwui-drawer>
```


这样一个抽屉组件就定义好了，记录一下～

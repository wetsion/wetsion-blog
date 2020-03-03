---
title: Electron学习和小试牛刀
description: Electron学习和小试牛刀
categories:
 - 前端
tags:
 - vue
 - electron
---



&emsp;&emsp;前段时间发现了一个好玩的东西：「Electron」，electron是什么呢，electron让前端开发者使用js、html、css就可以构建跨平台的桌面应用程序，electron框架解决了底层原生比较难搞的部分，提供了丰富的API，以及打包生成安装程序，让前端开发者能更专注在应用本身上。

&emsp;&emsp;简言之，通过electron，写js也可以开发桌面app了。

&emsp;&emsp;当我刚看到这个东西，心里还是挺兴奋的，为现在JS的强大而感慨。然后立马着手学习，比葫芦画瓢，上手开发了个桌面MarkDown编辑器。

&emsp;&emsp;首先还是先了解一些基本概念，以及electron提供的API，赋予了我们哪些能力。

&emsp;&emsp;electron本质上是将Chromium和Node合并在了同一个运行环境中，也可以理解为它相当于封装了一个浏览器，我们开发的其实还是web页面，只不过运行在了内置浏览器中，而Node则支持了与操作系统交互。所以，electron的应用架构主要是基于「主进程」和「渲染进程」。

#### 主进程

&emsp;&emsp;主进程，从目录结构和代码来说，就是运行`package.json`和`src/main.js`脚本的进程。主进程负责创建窗口，创建web页面，包括与操作系统交互的操作。

#### 渲染器进程

&emsp;&emsp;上面说到了主进程是创建窗口，而渲染器进程则是负责在创建好的窗口中渲染页面，例如vue代码将会在渲染器进程中解析运行。


&emsp;&emsp;再看一些基本API概念，至少在我的桌面markdown编辑器涉及到的：

#### ipcMain

&emsp;&emsp;主进程的通信器，在主进程中使用，用于处理从渲染器进程发送的消息以及向渲染器进程发送消息

> 接收消息： ipcMain.on(msgName, handler)  
> 发送消息： 需要注意的是ipcMain并不像ipcRenderer那样提供了send方法，想要向渲染器进程发消息，需要换种方式：BrowserWindow.getFocusedWindow().webContents.send(msgName, data)

#### ipcRenderer

&emsp;&emsp;渲染器进程的通信器，在渲染器进程中使用，向主进程发送消息和接收主进程消息。

> 接收消息： ipcRenderer.on(msgName, handler)  
> 发送消息： ipcRenderer.send(msgName)

#### BrowserWindow

&emsp;&emsp;窗口，在主进程中创建

#### app

&emsp;&emsp;在主进程中，控制应用程序的事件生命周期，提供了诸如`ready`、`window-all-closed`等事件监听，以及例如`quit()`这样的方法

#### dialog

&emsp;&emsp;对话框，主进程中执行，弹出例如打开文件、保存文件、警告这样的**操作系统对话框**

> 举两个常用的方法，在我的项目里也有涉及的：  
> dialog.showSaveDialog(options, callback) 弹出保存文件对话框，可在options参数中指定保存文件类型  
> dialog.showOpenDialog(options, callback) 弹出打开文件对话框，可在options参数中指定诸如是否能多选文件

#### Menu

&emsp;&emsp;主进程中执行，创建操作系统原生应用菜单、上下文菜单，（包括Mac系统下的dock菜单）

> 通常调用`Menu.buildFromTemplate(template)`方法指定模版创建菜单对象，再在应用窗口创建完成后，通过`Menu.setApplicationMenu(menus)`方法设置应用的系统菜单，针对Mac系统，通过`app.dock.setMenu(menu)`设置dock菜单


&emsp;&emsp;上述就是项目组用到的基本API，markdown编辑器的UI主要是通过vue + elementUI实现，也就是`electron-vue`，并引用了一个第三方开源的MarkDown组件：`mavonEditor`，总体还是挺简单的，实现的功能包括窗口最大化、窗口最小化、关闭窗口、自定义窗口顶部标题栏、自定义系统菜单、自定义dock菜单、打开文件、保存文件、拖拽打开文件。

&emsp;&emsp;项目版本：`vue 2.5.16`、`vue-electron 1.0.6`、`electron 2.0.4`、`element-ui 2.13.0`

主进程执行的脚本，`src/main.js`：

```js
import { app, BrowserWindow, Menu, ipcMain, dialog } from 'electron'

if (process.env.NODE_ENV !== 'development') {
  global.__static = require('path').join(__dirname, '/static').replace(/\\/g, '\\\\')
}

let mainWindow
const winURL = process.env.NODE_ENV === 'development'
  ? `http://localhost:9080`
  : `file://${__dirname}/index.html`

let aboutWindow
const aboutWindowUrl = process.env.NODE_ENV === 'development'
  ? `http://localhost:9080/#about`
  : `file://${__dirname}/index.html#/about`

let windowCount = 0

function createWindow () {
  /**
   * 初始化主窗口
   */
  mainWindow = new BrowserWindow({
    title: '秋月编辑器',
    height: 600,
    useContentSize: true,
    width: 1000,
    frame: false, // 默认标题栏去掉
    // resizable: false, // 是否窗口可调整大小
    webPreferences: {
      nodeIntegration: true,
      nodeIntegrationInWorker: true,
      webSecurity: false,
      devTools: true
    }
  })

  mainWindow.loadURL(winURL)
  windowCount ++

  mainWindow.on('closed', () => {
    mainWindow = null
  })

  // 去除原生顶部菜单栏
  mainWindow.setMenu(null)

  // 自定义Mac 的 dock菜单
  const dockMenu = Menu.buildFromTemplate([
    {
      label: '新窗口',
      click () {
        createWindow()
      }
    }
  ])
  app.dock.setMenu(dockMenu)

  return mainWindow
}

// 主进程监听渲染器进程发来的事件
// 关闭窗口
ipcMain.on('close', () => {
  console.log('window count: ', windowCount)
  BrowserWindow.getFocusedWindow().close()
})

ipcMain.on('min', () => {
  BrowserWindow.getFocusedWindow().minimize()
})

ipcMain.on('max', () => {
  BrowserWindow.getFocusedWindow().maximize()
})

ipcMain.on('unmax', () => {
  BrowserWindow.getFocusedWindow().unmaximize()
})

ipcMain.on('ondragstart', (event, filePath) => {
  event.sender.startDrag({
    file: filePath,
    icon: `file://${__dirname}/title/max.png`
  })
})

ipcMain.on('render-process-open-file', () => {
  dialog.showSaveDialog(
    {
      filters: [
        { name: 'MarkDown', extensions: ['md'] }
      ]
    },
    (filename, bookmark) => {
      console.log('render-process-open-file:', filename)
      if (filename) {
        BrowserWindow.getFocusedWindow().webContents.send('main-process-save-to-another', filename)
      }
    }
  )
})

app.on('ready', () => {
  createWindow()
	let menu = Menu.buildFromTemplate([
    {
      submenu: [
        {
          label: '关于秋月编辑器',
          click (event, focusedWindow, focusedWebContents) {
            if (aboutWindow) {
              aboutWindow.show()
            } else {
              aboutWindow = new BrowserWindow({
                title: '关于秋月编辑器',
                height: 400,
                useContentSize: true,
                width: 300,
                resizable: false,
                frame: true,
                webPreferences: {
                  nodeIntegration: true,
                  nodeIntegrationInWorker: true,
                  webSecurity: false,
                  devTools: true
                }
              })
              aboutWindow.loadURL(aboutWindowUrl)
              aboutWindow.on('closed', () => {
                aboutWindow = null
              })
            }
          }
        },
        {
          label: '关闭当前',
          role: 'close'
        },
        {
          label: '退出',
          role: 'quit'
        }
      ]
    },
		{
			label: '文件',
			submenu: [
        {
          label: '打开..',
          click (event, focusedWindow, focusedWebContents) {
            dialog.showOpenDialog(
              {
                properties: ['openFile'],
                filters: [
                  { name: 'MarkDown', extensions: ['md'] }
                ]
              },
              (filePaths, bookmarks) => {
                console.log(filePaths)
                if (filePaths) {
                  focusedWindow.webContents.send('main-process-open-file', filePaths)
                }
              }
            )
          }
        },
        {
          label: '保存',
          click (event, focusedWindow, focusedWebContents) {
            focusedWindow.webContents.send('main-process-save', 'save')
          }
        },
        {
          label: '另存为',
          click (event, focusedWindow, focusedWebContents) {
            dialog.showSaveDialog(
              {
                filters: [
                  { name: 'MarkDown', extensions: ['md'] }
                ]
              },
              (filename, bookmark) => {
                console.log(filename)
                console.log(bookmark)
                if (filename) {
                  focusedWindow.webContents.send('main-process-save-to-another', filename)
                }
              }
            )
          }
        }
      ]
		}
	])
	Menu.setApplicationMenu(menu)
})

app.on('window-all-closed', () => {
  if (process.platform !== 'darwin') {
    app.quit()
  }
})

app.on('activate', () => {
  if (mainWindow === null) {
    createWindow()
  }
})

```

主要创建窗口，以及监听处理各类事件。

自定义的窗口顶部标题栏：

**title.vue**

```js
<template>
	<div id="newTitle">
		<div class="left">
			<title-btn type="max"/>
			<title-btn type="min"/>
			<title-btn type="close"/>
		</div>
		<div class="center">
			<span>{{ label }}</span>
		</div>
		<div class="right">
			<span class="name">秋月编辑器-MarkDown神器</span>
		</div>
	</div>
</template>

<script>
	import TitleBtn from './TitleBtn/title-btn'
	import { getMdFilePath } from '@/utils/storerage'

  export default {
		name: 'title',
		components: { TitleBtn },
		props: ['label']
	}
</script>
```

**title-btn.vue**：

```js
<template>
	<div class="title-btn" @click="click">
		<img v-if="type === 'max' && !isMax" id="maxLogo" src="~@/assets/title/max.png" alt="electron-vue">
		<img v-if="type === 'max' && isMax" id="maxLogo" src="~@/assets/title/unmax.png" alt="electron-vue">
		<img v-if="type === 'min'" id="minLogo" src="~@/assets/title/min.png" alt="electron-vue">
		<img v-if="type === 'close'" id="closeLogo" src="~@/assets/title/close.png" alt="electron-vue">
	</div>
</template>

<script>
	const style = {
		min: {
			backgroundColor: 'red',
			right: '100px'
		},
		max: {
			// backgroundColor: 'yellow',
			backgroundImage: '~@/assets/title/max.png',
			right: '60px'
		},
		close: {
			backgroundColor: 'black',
			right: '20px'
		}
	}
	export default {
		name: 'title-btn',
		props: ['type'],
		data () {
			return {
				isMax: false
			}
		},
		computed: {
			style () {
				return style[this.type]
			}
		},
		methods: {
			click () {
				console.log(this.type)
				switch (this.type) {
					case 'max':
						if (!this.isMax) {
							this.$electron.ipcRenderer.send('max')
							this.isMax = true
						} else {
							this.$electron.ipcRenderer.send('unmax')
							this.isMax = false
						}
						break;
					case 'min':
						this.$electron.ipcRenderer.send('min')
						break;
					case 'close':
						this.$electron.ipcRenderer.send('close')
						break;
				}

			}
		}
	}
</script>
```

markdown编辑主页面：

`edit.vue`

```js
<template>
  <div id="editorDiv">
		<mavon-editor v-model="markdownText" :toolbars="toolbarConfig" :style="editorStyle" @save="save"/>
	</div>
</template>

<script>
  import { saveMdFilePath } from '@/utils/storerage'
  const fs = require('fs')
  export default {
    name: 'edit',
    data () {
      return {
        markdownText: '',
        sourceFilePath: undefined,
        toolbarConfig: {
          bold: true, // 粗体
          italic: true, // 斜体
          header: true, // 标题
          underline: true, // 下划线
          strikethrough: true, // 中划线
          mark: true, // 标记
          superscript: true, // 上角标
          subscript: true, // 下角标
          quote: true, // 引用
          ol: true, // 有序列表
          ul: true, // 无序列表
          link: true, // 链接
          imagelink: true, // 图片链接
          code: true, // code
          table: true, // 表格
          fullscreen: false, // 全屏编辑
          readmodel: false, // 沉浸式阅读
          htmlcode: true, // 展示html源码
          help: true, // 帮助
          /* 1.3.5 */
          undo: true, // 上一步
          redo: true, // 下一步
          trash: true, // 清空
          save: true, // 保存（触发events中的save事件）
          /* 1.4.2 */
          navigation: true, // 导航目录
          /* 2.1.8 */
          alignleft: true, // 左对齐
          aligncenter: true, // 居中
          alignright: true, // 右对齐
          /* 2.2.1 */
          subfield: true, // 单双栏模式
          preview: true
        }
      }
    },
    computed: {
      editorStyle () {
        console.log(window.innerHeight)
        return {
            height: '530px'
        }
      }
    },
    mounted () {
      const editorDiv = document.getElementById('editorDiv')

      editorDiv.addEventListener('drop', (event) => {
        event.preventDefault()
        const file = event.dataTransfer.files
        const path = file[0].path
        console.log('drag file:', path)
        this.openMd(path)
      })

      editorDiv.addEventListener('dragover', (event) => {
        event.preventDefault()
      })

      // 渲染器监听主进程点击保存文件后发送的事件
      this.$electron.ipcRenderer.on('main-process-save', () => {
        this.save()
      })

      // 渲染器监听主进程点击另存为后发送的事件
      this.$electron.ipcRenderer.on('main-process-save-to-another', (event, data) => {
        console.log(data)
        this.saveAnother(data)
      })

      // 渲染器监听主进程打开文件后发送的事件
      this.$electron.ipcRenderer.on('main-process-open-file', (event, data) => {
        console.log(data)
        this.openMd(data[0])
      })
    },
    methods: {
      /**
       * 保存
       */
      save () {
        if (this.sourceFilePath) {
          // 如果是已经打开的文档，直接保存
          this.saveMd(this.sourceFilePath)
        } else {
          // 如果是空白文档，先选择存储地址,然后后续按照「另存为」方式
          this.$electron.ipcRenderer.send('render-process-open-file')
        }
      },
      /**
       * 另存为
       */
      saveAnother (path) {
        console.log('save another:', path)
        if (this.sourceFilePath) {
          // 如果已经是打开的文件，则直接另存为
          this.saveMd(path)
        } else {
          // 如果是空白文件，另存时则保存并打开文件
          this.sourceFilePath = path
          this.saveMd(path)
          this.openMd(path)
        }
      },
      saveMd (path) {
        fs.writeFileSync(path, this.markdownText)
        this.$message.success('保存成功！')
      },
      openMd (path) {
        this.sourceFilePath = path
        const content = fs.readFileSync(path)
        const cttStr = content.toString()
        console.log(cttStr)
        this.markdownText = cttStr
        // this.$electron.ipcRenderer.send('ondragstart', path)
        // 告知主界面文件路径
        this.$emit('set-file-path', this.sourceFilePath)
      }
    }
  }
</script>
```

效果如下：

![WX20200303-160511@2x.png](https://i.loli.net/2020/03/03/GhM9dUtk8mZ7w2z.png)

源码地址：https://github.com/wetsion/electron-vue-demo

electron文档地址：https://www.electronjs.org/docs

electron还有很多功能，还需要通过实践去学习探索。
---
title: 使用electron+vue搭建一个桌面应用
---

&emsp;[electron](https://electronjs.org/) 是一个可以使用 web 技术来创建跨平台原生桌面应用的框架。借助 [electron](https://electronjs.org/) ，我们可以使用纯 JavaScript 来调用丰富的原生 APIs。优点：可以开发跨平台应用；成熟的社区；图形化的开发。缺点：应用体积过大；重型项目性能问题。

&emsp;[electron](https://electronjs.org/) 核心的部分就是两个进程之间的协作------主进程和渲染进程。进程之间通过 ipcMain 和 ipcRenderer 来进行通信。<!--more-->

>主进程：在 electron 里面，运行main 脚本的进程成为主进程。主进程控制整个应用的生命周期，在主进程中可以创建 Web 形式的 GUI，而且整个 Node API 是内置其中。
>
>渲染进程：每个 electron 的页面都运行着自己的进程。

&emsp;与其他桌面应用开发框架，例如qt，c#语言等相比，electron的开发速度更快，跨平台能力更强大。

----

#### 搭建基础的electron+vue项目

1. 搭建基础vue项目

```shell
vue new desktop-app
```

2. 引入electron-builder

```shell
vue add electron-builder
```

3. 运行APP

```shell
vue run electron:serve
```

运行效果如下图：

![browser](https://raw.githubusercontent.com/shiroJin/image-storage/master/electron_1.jpg)

#### 配置background.js

1. 设置窗口大小

```js
win = new BrowserWindow({
  width: 800, // 设置窗口的宽
  height: 600, // 设置窗口的高
  // fullscreen: true, 是否设置全屏显示
  webPreferences: {
    nodeIntegration: true // 是否完整支持node
  }
})
```

2. 跨域安全设置

```js
win = new BrowserWindow({
  webviewTag: true, // 是否使用webview标签
  fullscreen: true, // 使用全屏
  webPreferences: {
    webSecurity: false, // 是否禁用浏览器的跨域安全特性
  }
})
```

3. 开发工具配置

window的webContents属性调用openDevTools方法就能唤起开发者工具，webview标签的开发者工具也同理。

```js
if (!process.env.IS_TEST) win.webContents.openDevTools()
```

#### GUI开发

​	界面开发的流程跟vue项目一致，electron将编译好的资源文件打包进了自己的APP，所以一些资源的路径设置，例如public目录下的资源访问，图片资源的路径等等。electron也提供了加载外部h5页面的标签——webview。

​	简单的做一个浏览器的示例，在APP.vue中布局基本的导航页，然后内嵌一个webview。

#### webview标签

​	使用 `webview` 标签在Electron 应用中嵌入 "外来" 内容 (如 网页)。外来"内容包含在 `webview` 容器中。 应用中的嵌入页面可以控制外来内容的布局和重绘。

​	与 `iframe` 不同, `webview` 在与应用程序不同的进程中运行。它与您的网页没有相同的权限, 应用程序和嵌入内容之间的所有交互都将是异步的。 这将保证你的应用对于嵌入的内容的安全性。 **注意:** 从宿主页上调用 webview 的方法大多数都需要对主进程进行同步调用。

​	`webview`标签使用示例如下:

```vue
<webview id="foo" src="https://www.github.com/" style="display:inline-flex; width:640px; height:480px"></webview>
```

​	在script写用于侦听 `webview` 事件的 JavaScript, 并使用 `webview` 方法响应这些事件。 下面是包含两个事件侦听器的示例代码: 一个侦听网页开始加载, 另一个用于网页停止加载, 并在加载时显示 "loading..." 消息:

```vue
<script>
  onload = () => {
    const webview = document.querySelector('webview')
    const indicator = document.querySelector('.indicator')
    const loadstart = () => {
      indicator.innerText = 'loading...'
    }
    const loadstop = () => {
      indicator.innerText = ''
    }
    webview.addEventListener('did-start-loading', loadstart)
    webview.addEventListener('did-stop-loading', loadstop)
  }
</script>
```

​	webview处理openUrl、a标签等打开新标签页的处理不是很完善。需要自行拦截新窗口事件然后自行打开窗口或者在当前标签页内加载。

```js
webview.addEventListener('new-window', (event) => {
  webview.loadURL(event.url)
})
```

具体api请参考https://www.electronjs.org/docs/api/webview-tag

效果如图：

![browser](https://raw.githubusercontent.com/shiroJin/image-storage/master/electron_2.jpg)

#### 打包

​	执行完以下命令后，打包文件在项目根目录下**dist_electron**目录中，需要注意的是，mac系统只能打dmg文件、win系统只能打exe文件。

```shell
$ npm run electron:build
```


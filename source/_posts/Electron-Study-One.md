---
title: Electron学习| Electron的安装和HelloWorld
date: 2018-03-26 10:33:45
tags: Electron
categories:
  - 码农学习日常
---
由于需要对最近的一个项目进行可视化，就像趁着这个机会学习一下Electron和前端，于是就有了这个系列

## Electron简介
Electron官方教程 [点我](https://electronjs.org/docs/tutorial/quick-start) 已经介绍了Electron是可以通过JS来调用原生操作系统的API，进而可以很容易的创建一个跨平台的应用，是一种Node.js应用的变体

## Electron安装
首先需要安装好nodejs，具体安装教程不再赘述。  
安装好之后，初始化一个应用
```bash
node init
```
跟随引导创建一个packet.json，创建好之后大概是这个样子
```json
{
  "name": "project",
  "version": "1.0.0",
  "description": "hello world",
  "main": "main.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC",
}
```
其中"main"字段默认为"index.js",Electron的主要逻辑都会在这个js里面，根据需要可以更改为不同的名称，这里改为"main.js"  

### npm安装
然后在工程所在文件夹打开终端，安装electron
```bash
npm install --save-dev electron
```
也可以全局安装
```bash
npm install electron -g
```
由于安装的时候会从Github上下载对应的预编译文件，如果网络速度比较慢，可以选择使用淘宝的cnpm进行安装  

### cnpm安装
首先安装cnpm
```bash
 npm install -g cnpm --registry=https://registry.npm.taobao.org
 ```
 然后使用cnpm安装electron，使用方法和npm完全相同
 ```bash
 cnpm install --save-dev electron
 ```

 ## Electron的Hello World
 安装完之后，就可以新建一个应用了。在项目文件夹中创建一个main.js (根据packet.json中main字段对应的名称创建)

 然后写入以下代码
 ```javascript
 const {app, BrowserWindow} = require('electron')
  const path = require('path')
  const url = require('url')
  
  function createWindow () {
    // Create the browser window.
    win = new BrowserWindow({width: 800, height: 600})
  
    // 然后加载应用的 index.html。
    win.loadURL(url.format({
      pathname: path.join(__dirname, 'index.html'),
      protocol: 'file:',
      slashes: true
    }))
  }
  
  app.on('ready', createWindow)
 ```

上述代码创建了一个宽800，高600的窗口，然后加载同文件夹下的index.html

index.html代码如下
```html
<!DOCTYPE html>
  <html>
    <head>
      <meta charset="UTF-8">
      <title>Hello World!</title>
    </head>
    <body>
      <h1>Hello World!</h1>
      We are using node <script>document.write(process.versions.node)</script>,
      Chrome <script>document.write(process.versions.chrome)</script>,
      and Electron <script>document.write(process.versions.electron)</script>.
    </body>
  </html>
```

### 启动Electron
在packet.json中加入start字段
```json
{
  "name": "project",
  "version": "1.0.0",
  "description": "hello world",
  "main": "main.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1",
    "start": "electron ."
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    "electron": "^1.8.4"
  }
}
```
然后在终端中执行
```bash
npm start
```
即可以运行Electron。界面如下
![](http://p3jggzq4i.bkt.clouddn.com/electron-helloworld.png)

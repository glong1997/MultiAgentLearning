## HTML





## CSS





## JavaScript





## Vue

### Electron-Vue

安装electron

```
npm install -g electron
```

创建项目

```
vue create electron-demo
```

cd到项目electron-demo

```
vue add electron-builder
```



```
npm install

npm run electron:serve
```

打包

```
npm run electron:build
```

生成`dist_electron`文件夹，在`win-unpacked`里面找到后缀为`.exe`的可执行文件。



```
├─public
	├─index.html	渲染进程
└─src
    ├─assets
    └─components
    └─App.vue	负责渲染
    └─main.js	导包
```




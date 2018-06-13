---
layout: post
title: "ts-loader 与 Vue SFC 编译错误"
date: 2018-06-13
---

开始一个新的启用 TypeScript 的 Vuejs 项目，写完基本内容第一次启动就碰见错误：
```
Module build failed: TypeError: Cannot read property 'afterCompile' of undefined
......
```

挠头半天发现，原来是 ts-loader 新版本的问题。解决办法：
* ``` npm i --save-dev ts-loader@^3.5.0 ```
* 在 ``` webpack.config.js ``` 里用 ``` vue-ts-loader ``` 替换 ``` ts-loader ```
    ```javascript
    { 
        test: /\.ts(x?)$/,
        exclude: /node_modules/,
        loader: 'vue-ts-loader'
    }
    ```
  在 xxxx.vue 的 ``` script ``` 节点添加 ``` lang="vue-ts" ```

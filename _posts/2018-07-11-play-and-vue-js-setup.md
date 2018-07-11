---
layout: post
title: Vue.js + Play 混合开发
date: 2018-07-11
---
Play 是个相对简单的 Java/Scala 框架，后端开发过程中可以主要跟 Java 打交道，最多附带一点 Scala，不会变成 XML 开发。

但 Play 有一点不够好，它用的是 [Twirl](https://github.com/playframework/twirl) 模板引擎，而主流的界面框架像 Angular/React/Vue.js 能够更快速的拼装出页面。除此之外，前端开发有一些成熟的工具和集成开发环境，像 webpack 和 Visual Studio Code，能够极大程度解放生产力。其实，Play 本身支持流行的前端框架，只是不在默认配置内。以 Vue.js 为例，几个小步骤即可实现前后端分离开发，生产环境合并部署。

前端用 webpack + Visual Studio Code，以 webpack 模版创建 Vue.js 项目

```bash
npm install -g @vue/cli-init
# vue init now works exactly the same as vue-cli@2.x
vue init webpack my-project
```

以 Play 的 [REST API 项目](https://github.com/playframework/play-java-rest-api-example/tree/2.6.x)作为后端示例。这样，一个前后端分离的项目就成型了，Vue.js 项目仅负责界面逻辑，包括页面组织，导航跳转，数据展现等等，而后台专注于 REST API 的数据服务。

开发环境下，界面的 web 服务由 [webpack-dev-server](https://webpack.js.org/guides/development/#using-webpack-dev-server) 提供，它包含了一系列旨在提高开发效率的特性。而且，webpack-dev-server 还可以作为后台数据服务的代理，在 config/index.js 中增加：

```Javascript
module.exports = {
  // ...
  dev: {
    proxyTable: {
      '/api': {
        target: 'http://localhost:9000',
        changeOrigin: true
      }
    }
  }
}
```

wepack-dev-server 就将任何以 /api 起始的链接转发到后端的数据服务上，而对于 Play 框架来说，这些请求与直接的请求没有区别。

部署到生产环境，[自然无法再使用这种方式](https://en.wikipedia.org/wiki/Cross-site_scripting)。在正式的生产环境下，前后端最好由统一的 web 服务器提供服务，降低部署复杂度。直接的做法就是把 webpack 生成的 dist 文件夹整体作为后台的静态资源。

```
GET     /                           controllers.HomeController.index

->      /api                        endpoints.Routes

# 这是入手点
GET     /assets/*file               controllers.Assets.versioned(path="/public", file: Asset)
```
依照文档描述，配置 [asset controller](https://www.playframework.com/documentation/2.6.x/AssetsOverview#the-assets-controller)。首先，在 routes 里修改静态资源的路由信息
```
GET  /assets/*file  controllers.Assets.versioned(path="/public", file: Asset)
```
接下来，就是修改 Play 示例代码的 views/index.scala.html，让它变成 Vue.js 的 index.html。这里，我们用的是把生成的 index.html 直接拿来微调的办法，当然，也可以在前端项目中增加 index.html.template 来实现。
```html
<!-- index.scala.html -->

<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset=utf-8>
  <meta name=viewport content="width=device-width,initial-scale=1">
  <title>
    ui
  </title>
  <link href='@routes.Assets.versioned("assetcss/app.7a27261b57054469e5aa6196ff2a634a0e36368a.css")' rel=stylesheet>
</head>
<body>
<div id=app></div>
<script type=text/javascript src='@routes.Assets.versioned("js/manifest.fb1c0051aa2af0e62d3de62c6c374b87838ecf9b.js")'></script>
<script type=text/javascript src='@routes.Assets.versioned("js/vendor.31e2f39332bf57cfb205484cc04b7d2f30ab6107.js")'></script>
<script type=text/javascript src='@routes.Assets.versioned("js/app.48022844b9ea394376dd0906cc890dd6269c3ade.js")'></script>
</body>
</html>
```
这个方案的问题在于，webpack 生成的是抽取合并混淆过的资源文件，文件名中带了 hash 编码以更新浏览器缓存。这样每次更新前端内容后都需要调整 index.scala.html 的内容，太繁琐。

为处理这个问题，使用 [AssertFinder](https://www.playframework.com/documentation/2.6.x/AssetsOverview#using-configuration-and-assetsfinder) 来自动替换文件名。回到配置 asset controller 的步骤，在 application.conf 里添加静态资源的位置信息
```Scala
# setup asset path
play.assets {
  path = "/public"
  urlPrefix = "/static"
}
```
然后，修改 routes 中静态资源路由
```
GET     /static/*file               controllers.Assets.versioned(file)
```
注意，这里不再使用 file: Asset 而是用 file，同时也去掉了 path 参数。

接着，调整 index.scala.html
```html
<!-- index.scala.html -->
@(assetsFinder: AssetsFinder)

<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset=utf-8>
  <meta name=viewport content="width=device-width,initial-scale=1">
  <title>
    ui
  </title>
  <link href='@assetsFinder.path("css/app.css")' rel=stylesheet>
</head>
<body>
<div id=app></div>
<script type=text/javascript src='@assetsFinder.path("js/manifest.js")'></script>
<script type=text/javascript src='@assetsFinder.path("js/vendor.js")'></script>
<script type=text/javascript src='@assetsFinder.path("js/app.js")'></script>
</body>
</html>
```
新的 index.scala.html 中，不再指定具体的资源名，使用的是具体的文件名，需要 AssetFinder 的辅助，才能获取真正的带 hash 的 webpack 输出结果文件。

为此，需要增加一个 VueAssetFinder 类
``` Java
package controllers;

import java.io.File;

public class VueAssetFinder implements AssetsFinder {
    @Override
    public String assetsBasePath() {
        return "static";
    }

    @Override
    public String assetsUrlPrefix() {
        return "";
    }

    @Override
    public String findAssetPath(String basePath, String rawPath) {
        String assetName = null, ext = null, dirName = null;
        int index = rawPath.lastIndexOf("/");
        if (index != -1) {
            String fileName = rawPath.substring(index + 1);
            String[] paths = fileName.split("\\.");
            if (paths.length < 2) return null;

            assetName = paths[0];
            ext = paths[1];
            dirName = rawPath.substring(0, index);
        }

        String searchPath = "public";
        if (dirName != null) {
            searchPath = dirName.replace(basePath, searchPath);
        }

        File[] files = new File(searchPath).listFiles();
        for (File file : files) {
            // 查找目标文件，匹配前缀和扩展名
            if (file.getName().startsWith(assetName) && file.getName().endsWith(ext)) {
                String path = dirName + File.separatorChar + file.getName();
                if (File.separatorChar != '/')
                    path = path.replace(File.separatorChar, '/');
                return path;
            }
        }
        return null;
    }

    @Override
    public String path(String rawPath) {
        return AssetsFinder.super.path(rawPath);
    }

    @Override
    public AssetsFinder withUrlPrefix(String newPrefix) {
        return AssetsFinder.super.withUrlPrefix(newPrefix);
    }

    @Override
    public AssetsFinder withAssetsPath(String newPath) {
        return AssetsFinder.super.withAssetsPath(newPath);
    }

    @Override
    public AssetsFinder unprefixed() {
        return this.withUrlPrefix("");
    }
}
```

最后，在 HomeController 中使用自定义的 AssetFinder, 从而输出正确的最终 index.html 页面。
```Java
public class HomeController extends Controller {
    public Result index() {
        return ok(views.html.index.render(new VueAssetFinder()));
    }
}
```
所有设置完成后，部署生产环境就变成常规的 Play 项目部署之前，增加 webpack prod build 以及拷贝 webpack 生成结果的步骤。新增的步骤可以通过前后台整体的编译脚本完成，也可以直接在前端的 webpack.conf.js 中指定生成目录解决。

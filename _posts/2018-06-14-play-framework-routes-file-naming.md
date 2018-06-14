---
layout: post
title: Play 框架里 routes 文件的命名
date: 2018-06-14
---

如果你在用 [play](https://www.playframework.com/) for Java，肯定会用到 routes 以及分层的子 routes 配置。

`routes`:

~~~
GET     /                           controllers.HomeController.index

->     /api                         api.Routes
~~~

`api.routes`:

~~~
GET    /                     v1.post.PostController.list
POST   /                     v1.post.PostController.create

GET    /:id                 v1.post.PostController.show(id)
PUT    /:id                 v1.post.PostController.update(id)
~~~

要真这么写，遇到的错误会让人莫名其妙，明明存在的类忽然就无法找到。

```Java
play.api.UnexpectedException: Unexpected exception[NoClassDefFoundError: api/Routes(wrong name: api/routes)]
    at play.core.server.DevServerStart$$anon$1.reload(DevServerStart.scala:190)
    at play.core.server.DevServerStart$$anon$1.get(DevServerStart.scala:124)
    at play.core.server.AkkaHttpServer.handleRequest(AkkaHttpServer.scala:222)
    ...
Caused by: java.lang.NoClassDefFoundError: api/Routes (wrong name: api/routes)
    at java.lang.ClassLoader.defineClass1(Native Method)
    at java.lang.ClassLoader.defineClass(ClassLoader.java:763)
    at java.security.SecureClassLoader.defineClass(SecureClassLoader.java:142)
    ...
```

问题在于 routes 文件的生成机制。框架会为 routes 文件们生成各自的类，

- api.routes 是入口的 routes 文件对应的 Java 类名， 文件名 `api/routes.java`
- 而应用程序自定义的 api.Routes 则对应到 Scala 类 api.Routes，文件名 `api/Routes.scala`


本来二者以不同的后缀名存在于磁盘上互不干扰，然而，Scala 和 Java [联合编译](http://www.codecommit.com/blog/scala/joint-compilation-of-scala-and-java-sources)时的字节码文件在文件名大小写不敏感的操作系统上（例如 Windows），就会变成同一个 routes.class 文件。

由于 api/Routes.class 被 api/routes.class 覆盖， api.Routes 类不存在，框架也就无法构造签名中包含 api.Routes 的 router.Routes 类实例，只能抛出异常。

```scala
class Routes(
    override val errorHandler: play.api.http.HttpErrorHandler,
    HomeController_0: controllers.HomeController,
    api_Routes_0: api.Routes,
    Assets_1: controllers.Assets,
    val prefix: String
) extends GeneratedRouter {
    ......
```

解决方法简单直接，换个名字，让应用程序的 routes 生成的 Scala 类落到另一个不同于 api 的目录。

`routes`:

~~~
GET     /                           controllers.HomeController.index

->      /api                        endpoints.Routes

GET     /assets/*file               controllers.Assets.versioned(path="/public", file: Asset)
~~~

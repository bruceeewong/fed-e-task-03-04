# 构建自己的Vue SSR项目

>  Vue SSR指南: [传送门](https://ssr.vuejs.org/zh/guide/structure.html#%E4%BD%BF%E7%94%A8-webpack-%E7%9A%84%E6%BA%90%E7%A0%81%E7%BB%93%E6%9E%84)

## 构建源码结构

## Webpack 打包同构应用

### 安装生产依赖

```
npm i vue vue-server-renderer express cross-env
```

### 安装开发依赖

```
npm i -D webpack webpack-cli webpack-merge webpack-node-externals @babel/core @babel/plugin-transform-runtime @babel/preset-env babel-loader css-loader url-loader file-loader rimraf vue-loader vue-template-compiler friendly-errors-webpack-plugin
```

| 包                                                           | 说明                                       |
| ------------------------------------------------------------ | ------------------------------------------ |
| webpack                                                      | webpack核心包                              |
| webpack-cli                                                  | webpack的命令行工具                        |
| webpack-merge                                                | webpack配置信息合并工具                    |
| webpack-node-externals                                       | 排除webpack中的node模块                    |
| friendly-errors-webpack-plugin                               | 友好错误提示插件                           |
| rimraf                                                       | 基于nodefengzhaung的一个跨平台`rm -rf`工具 |
| @babel/code<br />@babel/plugin-transform-runtime<br />@babel/preset-env<br />babel-loader | Babel相关工具                              |
| vue-loader<br />vue-template-compiler                        | 处理 .vue 资源                             |
| file-loader                                                  | 处理字体资源                               |
| css-loader                                                   | 处理css资源                                |
| url-loader                                                   | 处理图片资源                               |

## Bundle Renderer

基础SSR问题:

- 更改源码,需要手动重启server
- NodeJS不支持source-map

Vue server renderer 提供 `createBundleRenderer` 来解决上述问题. 通过将server bundle生成特殊的JSON文件, vue-server-renderer就能正常使用,并新增以下特性:

- 支持 source-map
- 支持热重载(开发时甚至部署后)
- CSS注入
- 通过`clientManifest`将静态资源注入

> 需要开放静态资源目录，使用express static 中间件

### vue-ssr-server-bundle.json

- entry: 入口
- files: 把模块打包后的代码输出到此
- maps: source-map信息

### vue-ssr-client-manifest.json

客户端打包资源的构建清单

- publicPath：静态资源路径
- all：客户端所有打包构建的文件名
- initial：初始脚本，自动注入到html页面script标签
- async：异步资源信息
- modules：模块信息，值是模块的索引数组

### 客户端激活 (client-side hydration)

所谓客户端激活，指的是 Vue 在浏览器端接管由服务端发送的静态 HTML，使其变为由 Vue 管理的动态 DOM 的过程。

由于服务器已经渲染好了 HTML，我们显然无需将其丢弃再重新创建所有的 DOM 元素。相反，我们需要"激活"这些静态的 HTML，然后使他们成为动态的（能够响应后续的数据变化）。

服务端返回的html应用程序的根元素上添加了一个特殊的属性：

```html
<div id="app" data-server-rendered="true">
```

`data-server-rendered` 特殊属性，让客户端 Vue 知道这部分 HTML 是由 Vue 在服务端渲染的，并且应该以激活模式进行挂载。

还可以向 `$mount` 函数的 `hydrating` 参数位置传入 `true`，来强制使用激活模式(hydration)：

```js
// 强制使用应用程序的激活模式
app.$mount('#app', true)
```

在开发模式下，Vue 将推断客户端生成的虚拟 DOM 树 (virtual DOM tree)，是否与从服务器渲染的 DOM 结构 (DOM structure) 匹配。如果无法匹配，它将退出混合模式，丢弃现有的 DOM 并从头开始渲染。**在生产模式下，此检测会被跳过，以避免性能损耗。**

## 如何设置开发模式热更新？

开发模式下，监视源代码变动，打包构建，重新生成 Renderer 渲染器

**思路: 开发模式下, 监听构建产物变动, 动态更新 bundleRenderer** 

### 监听模板文件变动

使用 [chokidar](https://github.com/paulmillr/chokidar) 包(基于`fs`模块的`watch`, `watchFile`)

### 监听webpack打包产物变动

使用`webpack`将配置文件传入得到`compiler`, 再调用`watch`方法监听构建变化.

以 serverBundle.json 文件为例, 在构建变动后, 可以调用回调函数, 然后读取最新的文件配置, 最后更新renderer

```js
  // serverBundle
  const serverConfig = require("./webpack.server.config");
  const serverCompiler = webpack(serverConfig);
  serverCompiler.watch({}, (err, stats) => {
    if (err) throw err; // webpack 错误
    if (stats.hasErrors()) return;

    serverBundle = JSON.parse(
      fs.readFileSync(resolve("../dist/vue-ssr-server-bundle.json"), "utf-8")
    );
    console.log(serverBundle);
    update();
  });
```

## 服务端打包产物写入内存中

> webpack Custome File Systems: [传送门](https://webpack.js.org/api/node/#custom-file-systems)
>
> webpack-dev-middleware: [传送门](https://github.com/webpack/webpack-dev-middleware)

使用 `webpack-dev-middleware` 将打包产物写入内存:

```js
const serverDevMiddleware = devMiddleware(serverCompiler, {
  logLevel: "silent",
});
// 绑定编译结束后钩子,注册读取内存中打包产物的逻辑
serverCompiler.hooks.done.tap("server", () => {
  serverBundle = JSON.parse(
    serverDevMiddleware.fileSystem.readFileSync(
      resolve("../dist/vue-ssr-server-bundle.json"),
      "utf-8"
    )
  );
  onsole.log(serverBundle);
  update();
});
```

### 客户端 manifest 打包产物写入内存

逻辑与上面服务端的打包逻辑相似,不同有两点:

1. 使用devMiddleware需要传递`publicPath`路径,值为webpack中配置的值
2. 需要为express的实例将客户端打包的devMiddleware应用为中间件, 提供对其内部内存数据的访问

```js
// clientManifest
  const clientConfig = require("./webpack.client.config");
  const clientCompiler = webpack(clientConfig);
  const clientDevMiddleware = devMiddleware(clientCompiler, {
    publicPath: clientConfig.output.publicPath, // 指定配置文件中的公共路径
    logLevel: "silent",
  });
  clientCompiler.hooks.done.tap("client", () => {
    clientManifest = JSON.parse(
      clientDevMiddleware.fileSystem.readFileSync(
        resolve("../dist/vue-ssr-client-manifest.json"),
        "utf-8"
      )
    );
    update();
  });

  // 只有客户端: 将 clientDevMiddleware 挂在到 Express 服务中, 提供对其内部内存数据的访问
  server.use(clientDevMiddleware);
```

## Webpack热更新

> Webpack hot middleware: [传送门](https://github.com/webpack-contrib/webpack-hot-middleware)

开启热更新需要以下几步:

- webpack启用插件 `HotModuleReplacementPlugin`

```js
clientConfig.plugins.push(new webpack.HotModuleReplacementPlugin()); // 开启热更新插件
```

- webpack打包入口添加热更新脚本:  `"webpack-hot-middleware/client`

```js
clientConfig.entry.app = [
	"webpack-hot-middleware/client?quiet=true&reload=true", // 和服务端交互的一个客户端脚本
	clientConfig.entry.app,
];
```

- 注册热更新中间件: `webpack-hot-middleware`

```js
server.use(
	hotMiddleware(clientCompiler, {
		log: false, // 关闭热更新日志
	})
);
```

> 注意: 热更新模式下,chunk hash会影响中间件, 因为不一致导致不可使用.

## 编写通用代码

> Vue SSR 指南: [传送门](ttps://ssr.vuejs.org/zh/guide/universal.html) 

### 服务器上的数据响应

每个请求应该都是全新的、独立的应用程序实例，以便不会有交叉请求造成的状态污染 (cross-request state pollution)。

### 组件生命周期钩子函数

由于没有动态更新，所有的生命周期钩子函数中，只有 `beforeCreate` 和 `created` 会在服务器端渲染 (SSR) 过程中被调用。这就是说任何其他生命周期钩子函数中的代码（例如 `beforeMount` 或 `mounted`），只会在客户端执行。

避免在 `beforeCreate` 和 `created` 生命周期时产生全局副作用的代码. 将副作用代码移动到 `beforeMount` 或 `mounted` 生命周期中

### 访问特定平台(Platform-Specific) API

通用代码不可接受特定平台的 API, 直接使用了像 `window` 或 `document`，这种仅浏览器可用的全局变量，则会在 Node.js 中执行时抛出错误，反之也是如此。

- 对于仅浏览器可用的 API，通常方式是，在「纯客户端 (client-only)」的生命周期钩子函数中惰性访问 (lazily access) 它们。

- 通过模拟 (mock) 一些全局变量来使其正常运行.

## 自定义指令

大多数自定义指令直接操作 DOM，因此会在服务器端渲染 (SSR) 过程中导致错误。

- 推荐使用组件作为抽象机制，并运行在「虚拟 DOM 层级(Virtual-DOM level)」（例如，使用渲染函数(render function)）
- 如果你有一个自定义指令，但是不是很容易替换为组件，则可以在创建服务器 renderer 时，使用 [`directives`](https://ssr.vuejs.org/zh/api/#directives) 选项所提供"服务器端版本(server-side version)"。

## 路由

> 使用VueRouter

同构应用必须使用`history`模式,不支持`hash`模式

首先配置`router`:

```js
import Vue from "vue";
import VueRouter from "vue-router";
import Home from '@/pages/Home';

Vue.use(VueRouter);

export function createRouter() {
  const router = new VueRouter({
    mode: "history", // 同构应用一定要用history模式
    routes: [
      {
        path: "/",
        name: "home",
        component: Home,
      },
      {
        path: "/about",
        name: "about",
        component: () => import("@/pages/About"),
      },
      {
        path: "*",
        name: "error404",
        component: () => import("@/pages/404"),
      },
    ],
  });
}
```

然后在服务端入口, 拿到router实例,

//...

### 代码分割与预取

异步import会在dist中生成分块的js

在html中主chunk js通过 `<link rel="preload" href="xxx.js" as="script" />` 获取, 异步chunk 通过 `<link rel="prefetch" href="xxx.js" />`

> preload vs prefetch: https://blog.csdn.net/luofeng457/article/details/88409903

#### preload

link标签的preload是一种声明式的资源获取请求方式，用于提前加载一些需要的依赖，并且不会影响页面的onload事件

- rel属性值为preload
- as属性用于规定资源的类型，并根据资源类型设置Accep请求头，以便能够使用正常的策略去请求对应的资源
- href为资源请求地址
- onload和onerror则分别是资源加载成功和失败后的回调函数；

preload特点

- preload加载的资源是在浏览器渲染机制之前进行处理的，并且不会阻塞onload事件；
- preload可以支持加载多种类型的资源，并且可以加载跨域资源；
- preload加载的js脚本其加载和执行的过程是分离的。即preload会预加载相应的脚本代码，待到需要时自行调用；

数据预取, 在body底部通过script获取

#### prefetch

prefetch是一种利用浏览器的空闲时间加载页面将来可能用到的资源的一种机制；通常可以用于加载非首页的其他页面所需要的资源，以便加快后续页面的首屏速度；

##### prefetch特点

prefetch加载的资源可以获取非当前页面所需要的资源，并且将其放入缓存至少5分钟（无论资源是否可以缓存）；并且，当页面跳转时，未完成的prefetch请求不会被中断；

数据预取, 在head的底部通过插入script获取

Chrome有四种缓存：http cache、memory cache、Service Worker cache和Push
cache。在preload或prefetch的资源加载时，两者均存储在http
cache。当资源加载完成后，如果资源是可以被缓存的，那么其被存储在http
cache中等待后续使用；如果资源不可被缓存，那么其在被使用前均存储在memory cache

#### 对比

- preload主要用于预加载当前页面需要的资源；而prefetch主要用于加载将来页面可能需要的资源
- preload和prefetch都没有同域名的限制；
- preload主要用于预加载当前页面需要的资源；而prefetch主要用于加载将来页面可能需要的资源；
  不论资源是否可以缓存，prefecth会存储在net-stack cache中至少5分钟；
- preload需要使用as属性指定特定的资源类型以便浏览器为其分配一定的优先级，并能够正确加载资源；
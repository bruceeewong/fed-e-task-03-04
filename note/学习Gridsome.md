# 学习Gridsome 

## 什么是Gridsome

[Gridsome](https://gridsome.org/): 一个免费、开基于Vue.js技术栈的静态网站生成器

### 静态网站生成器

静态网站生成器是使用一系列配置、模板以及数据，生成静态html文件以及相关资源的工具

也叫做预渲染，不需要PHP等服务器，只需要把资源放到支持静态资源的Web Server或者CDN即可

好处：

- 省钱：无需专业服务器，能托管静态文件即可
- 快速：不经过后端服务器的处理，只传输内容
- 安全：没有后端程序的执行，自然更安全

### JAMStack

JavaScript、API&Markup的结合，静态网站的生成器

## 适用场景

- 不适合有大量路由页面的应用：预渲染太慢
- 不适合有大量动态内容的应用

## 安装

>  中国区存在网络问题，以及sharp包包含C++文件

[sharp](https://github.com/lovell/sharp)包是Nodejs的图片处理库，需设置环境变量才能下载成功。

```
npm config set sharp_binary_host "https://npm.taobao.org/mirrors/sharp"
npm config set sharp_libvips_binary_host "https://npm.taobao.org/mirrors/sharp-libvips"
npm install sharp
```

>  如果安装出现 `pngpaunt`包连接github连不上，mac用户考虑使用`brew install libpng`来下载C++模块的包。

编译Nodejs中C++扩展包：[Node-gyp](https://github.com/nodejs/node-gyp)

下载脚手架

```
npm install --global @gridsome/cli
```

创建项目

```
gridsome create my-gridsome-site
```

## 配置

### 项目配置

> https://gridsome.org/docs/config/

### 路由配置

> https://gridsome.org/docs/pages/

- 文件约定式生成路由
- 编程式路由：gridsome.server.js

## 页面配置

- meta
- 定制404页面

## Collection 集合

> A collection is a group of nodes and each node contains fields with custom data. Collections are useful if you are going to have blog posts, tags, products etc. on your site

将不同数据源获取的数据，通过 gridsome 的api创建集合，并且将数据项作为该集合的节点存放：

```js
module.exports = function(api) {
  api.loadSource(async ({ addCollection }) => {
    // Use the Data Store API here: https://gridsome.org/docs/data-store-api/
    const collection = addCollection("Post");
    const { data } = await axios.get(
      "https://jsonplaceholder.typicode.com/posts"
    );
    for (const item of data) {
      collection.addNode({
        id: item.id,
        title: item.title,
        content: item.body,
      });
    }
  });
  //...
};

```

有了节点之后，就可以使用graphql的方式进行查询，开发环境访问 `http://localhost:8080/___explore`即可通过可视化方式访问。

在vue模板文件里，使用`<page-query />`标签编写`graphql`语句

```vue
<page-query>
query {
  posts: allPost {
    edges {
      node {
        id
        title
      }
    }
  }
}
</page-query>
```

### 模板 for 集合

可以为特定集合的节点定制模板，首先在`gridsome.config.js`里配置模板，跟配置vue路由很像：

```js
module.exports = {
  // ...
  templates: {
    Post: [
      {
        path: "/posts/:id",
        component: "./src/templates/Post.vue",
      },
    ],
  },
};
```

创建对应模板vue文件，在文件内部编写`graphql`查询语句，注意这里使用变量的方式，动态拿到路由参数

```vue
<page-query>
query ($id: ID!) {
  post (id: $id) {
    id
    title
    content
  }
}
</page-query>
```

然后在页面中以 `$page.post` 获取数据

```vue
<template>
  <div>
    <Layout>
      <h1>
        {{ $page.post.title }}
      </h1>
      <main>
        <p>
          {{ $page.post.content }}
        </p>
      </main>
    </Layout>
  </div>
</template>
```

如果想要以数据作为meta title，需要将metaInfo改写为函数，动态设置：

```vue
<script>
export default {
  name: 'Post',
  metaInfo() {
    return {
      title: this.$page.post.title
    }
  }
}
</script>
```

最后打包，预览产物，可以看见数据都被预先加载进来：

![image-20210401173621113](/Users/bruskiwang/Library/Application Support/typora-user-images/image-20210401173621113.png)


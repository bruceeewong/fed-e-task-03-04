# 使用Gridsome搭建博客

## 将博客模板迁移至gridsome

> 基于[startbootstrap-clean-blog](https://github.com/StartBootstrap/startbootstrap-clean-blog)为博客的样式模板

- 将依赖CSS通过npm安装，在main.js中引入全局样式
- 将项目的全局样式迁移至`src/assets/css/index.css`中，其中字体文件通过`@import`方式引入
- 将`nav`,`footer`添加到`Layout`
- 将`home`,`post`,`contact`,`about`页迁至对应vue组件中
- 将图片文件迁移至`static`，页面中通过`/img/xxx`基于根路径访问资源

## 支持markdown生成博客

安装文件转换插件：[@gridsome/source-filesystem](https://gridsome.org/plugins/@gridsome/source-filesystem)

注册插件：

```js
plugins: [
    {
      use: '@gridsome/source-filesystem',
      options: {
        typeName: 'BlogPost',
        path: './content/blog/**/*.md',
      }
    }
  ]
```

要支持markdown文件，还要安装 [@gridsome/transformer-remark](https://gridsome.org/plugins/@gridsome/transformer-remark) 将markdown转为html

此时在根目录下 `content/blog`创建markdown文件，插件就能把文本转为html文本存到graphql层

### MarkdownIt 转html

```
npm i markdown-it -D
```

```
import MarkdownIt from 'markdown-it'
const md = new MarkdownIt()
md.render('# title') // <h1>title</h1>
```

## Strapi - Headless CMS

### 安装

```bash
npx create-strapi-app my-project --quickstart
```

## 配置内容实体

![image-20210402144039690](/Users/bruskiwang/Library/Application Support/typora-user-images/image-20210402144039690.png)

创建内容实体，即可按描述自动生成对应实体与restful api

### 用户与角色

![image-20210402144237656](/Users/bruskiwang/Library/Application Support/typora-user-images/image-20210402144237656.png)

可以使用角色配置路由的权限

![image-20210402144309377](/Users/bruskiwang/Library/Application Support/typora-user-images/image-20210402144309377.png)

再创建用户、绑定角色，即可。

访问受保护的接口时，需要访问 `/auth/local` 接口获取 `jwt` 令牌，加到请求头的`Authorization`中，作为`Bearer token`使用，即可访问被保护路由。

### GraphQL支持

```
npm run strapi install graphql
```

启动后可以访问: `/graphql` 路径即可

## Gridsome集成Strapi

```
npm i @gridsome/source-strapi
```

Gridsome会将Strapi中的数据预取到本地graphql，所有的前缀会加上 `strapi` 如 `Post` -> `strapiPost`。

在`gridsome.config.js`中注册插件，指定graphql的数据项：

```js
plugins: [
	{
      use: '@gridsome/source-strapi',
      options: {
        apiURL: 'http://localhost:1337',
        queryLimit: 1000,
        contentTypes: ['post', 'tag'],
        // singleTypes: ['impressum'],
        loginData: {
          identifier: '',
          password: ''
        }
      }
    }
]
```

### 获取文章列表

```vue
<page-query>
query ($page: Int) {
  posts: allStrapiPost (perPage: 2, page: $page) @paginate {
    pageInfo {
      totalPages
      currentPage
    }
    edges {
      node {
        id 
        title
        created_at
        tags {
          id
          title
        }
      }
    }
  }
}
</page-query>

<script>
import { Pager } from 'gridsome'

export default {
  components: {
    Pager
  }
}
</script>
```

其中 `@paginate` 是gridsome提供的指令，用于当前页码传给graphql查询。

`Pager`组件是gridsome提供的基于graphql的pageInfo的分页器，与gridsome项目集成。

## 博客部署

需要部署Gridsome与Strapi

### Strapi 部署

需要支持NodeJS的服务器

> 问题：webpack Build时卡在90%，查询[issue](https://github.com/strapi/strapi/issues/3512)发现需要部署机器 RAM > 2G .... 

### gridsome 部署

#### 环境变量

添加环境变量文件 `.env.development` 与 `.env.production` 设置前端所需的环境变量

#### 使用 vercel 快速部署静态资源

> https://vercel.com/

授权vercel获取gitub权限，选择gridsome模板，快速部署、

如何让strapi更新后，gridsome实时更新？

使用webhook!

在vercel设置deploy hook，拿到构建api:

![image-20210402200714707](/Users/bruskiwang/Library/Application Support/typora-user-images/image-20210402200714707.png)在strapi中设置webhook，写入vercel的hook api，即可触发自动构建&部署！

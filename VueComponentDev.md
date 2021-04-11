# Vue组件库开发

## vue跨组件通信

### $root

获取根组件实例

对于小型项目的全局状态管理OK

没有vuex的时光旅行能力

### $parent

可以获取直接父组件的实例

### $children

数组的方式获取子组件

可读性不强,需要用下标来索引

### $refs

需要在组件上标记`ref="name"`来获取组件实例

当组件为html标签,则获取到对应DOM元素

当组件为vue组件,则获取对应vue组件实例

### provide-inject

通过在父组件提供 `provide`, 任意层级的子组件都可以通过 `inject` 注入

provide是函数,返回一个对象

```javascript
provide() {
  return {
    title: 'Title',
    handle() { console.log(this.title) }
  }
}```

inject语法

数组: `inject: ['title', 'handle']`

优点

任意层级都可以获取

缺点

组件间耦合

注意点

不可以直接修改`inject`的值!

对于其他也注入了该项的组件,  直接对`inject`值修改不是响应式的!

$attrs / $listeners

Vue Props知识

如果是class/style,会合并到最外层的标签的class/style上

不可以设置 `class` `style` 保留字段作为 props

如果子组件没有声明props接收,外部传入的props会自动设置到子组件最外层的标签上

​```html
<template>
  <input />
</template>

<script>
  export default {}
</script>```

$attrs

把父组件中非prop属性绑定到内部

如果设置了props, 则$attrs不包含 Props 声明的属性

注意：class/style还是只合并到最外层标签

示例

​```html
<div>
	<input v-bind="$attrs" />
</div>```

如果子组件中不想继承父组件传入的非prop属性，可以使用 inheritAttrs 禁用继承; 然后通过 v-bind="$attrs" 把外部传入的非prop属性设置给希望的标签上. 但不会透传 class 与 style

​```html
<template>
  <div>
  	<input v-bind="$attrs" />
  </div>
</template>

<script>
  export default {
    inheritAttrs: false
  }
</script>```

Vue event知识

子组件抛出自定义事件: $emit, 外部通过@来监听时间

​```html
// myinput.vue
<input @focus="$emit('focus', $event)" />

// parent.vue
<myinput @focus="(e) => console.log(e)" />```

$listeners

把父组件中的DOM对象的原生事件展开到组件内部

事件是DOM原生事件

​```html
<input v-on="$listeners" />```

### 快速原型开发

VueCLI提供一个插件可以进行原型快速开发,需要全局安装扩展

npm install -g @vue/cli-service-global

使用 vue serve 快速查看组件运行效果

基于ElementUI二次开发

安装

初始化: npm init -y

安装 ElementUI: vue add element

加载 ElementUI: 使用 Vue.use() 安装插件

### 组件分类

第三方组件

基础组件

业务组件

### 组件

表单组件

Form

表单容器组件

model: 接收表单对象

rules: 接收校验规则

功能

提供provide

validate: 表单验证

调用所有子类item的validate方法

FormItem

表单项容器

label: 接收表单项名称

展示校验错误信息

校验本控件的值,并提示错误信息

使用`async-validator`

使用依赖注入关联所有FormItem

model: 表单值

rules: 表单规则

根据校验结果,展示错误信息

Input

输入控件

实现v-model

:value

@input

使用 inheritAttrs: false 与 v-bind="$attrs" 透传input原生属性

功能

触发form父组件的表单验证(但不能强依赖form)

Button

### 项目的组织方式

Multirepo

每一个包对应一个项目

### Monorepo

一个项目仓库中管理多个模块/包, 可单独测试,发布等

项目结构

根目录放置 `packages`

每个子项都包含完整的

`__test__`

`src`

`dist`

`index.js`

`LISCENCE`

`package.json`

### Storybook

介绍

可视化的组件展示平台

在各离得开发环境中,以交互式的方式展示组件

独立开发组键

支持主流前端框架

实操

安装storybook

​```javascript
npx -p @storybook/cli sb init --type vue```

安装vue及其依赖

​```javascript
yarn add vue```

​```javascript
yarn add vue-template-compiler vue-loader --dev```

### Yarn workspaces

处理monorepo的各子包的异同依赖

给工作区下面的相同依赖提取到根目录

给重名但不同版本的包,安装到各自的子包目录

开启

​```javascript
// package.json
{
  "priavte": true,
  "workspaces": [
    "packages/*"
  ]
}```

使用

给指定工作区安装依赖

yarn workspace lg-button add lodash@4

Lerna

优化使用git和npm管理多包仓库的工作流工具

管理多个包的JavaScript项目

一键把代码提交到git

安装

yarn global add lerna

初始化

lerna init

发布

lerna: publish

给每个包package.json加上`gitHead`: 唯一hash

当前项目必须有修改才能发布

单元测试

依赖

VueTestUtils

Jest

vue-jest: 将SFC转为JS给jest

babel-jest: 对测试代码进行降级处理

​```javascript
yarn add jest @vue/test-utils vue-jest babel-jest -D -W```

Babel桥接

项目使用babel7, 而vue-test用的是babel6,需要安装桥接

​```javascript
yarn add babel-core@bridge -D -W  // -W 安装到工作区根目录```

配置

package.json配置测试命令

jest配置

测试点:

组件渲染dom是否包含预期标签

组件内部状态

组件快照

Rollup

介绍

是一个模块打包器

支持Tree-shaking

打包结果比webpack小,适合js-lib项目

安装

rollup

rollup-plugin-terser

rollup-plugin-vue@5.1.9(for vue2)

vue-template-compiler

配置文件

​```javascript
import { terser } from "rollup-plugin-terser";
import vue from "rollup-plugin-vue";
module.exports = [
  {
    input: "index.js",
    output: [
      {
        file: "dist/index.js",
        format: "es",
      },
    ],
    plugins: [
      vue({
        css: true,
        compileTemplate: true,
      }),
      terser(),
    ],
  },
];```

清理每个包的dist: 使用rimraf

清理每个包的node_modules: 使用lerna clean

基于模板生成项目结构

​```javascript
yarn add plop -D -W```


```
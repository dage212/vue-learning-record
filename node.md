ast就是由根节点构建的，RootNode定义的类型，根节点的children挂载子节点，子节点的children挂载孙子节点，以此类推
```javascript
export interface RootNode extends Node {
  type: NodeTypes.ROOT //节点类型,节点类型在NodeTypes枚举里
  source: string //节点源码
  children: TemplateChildNode[] //节点子节点
  helpers: Set<symbol> //这里放import需要引入的方法例如import {createElementVNode as _createElementVNode, resolveComponent as _resolveComponent} from 'vue'里面的createElementVNode,resolveComponent
  components: string[] //这里放引入的组件信息
  directives: string[] //指令信息
  hoists: (JSChildNode | null)[] //这里放静态节点，这里存放的是静态节点，不需要比较
  imports: ImportItem[] 
  //template包含下面tags里面的属性同名组件，就会被添加到imports里面
  // tags: {
  //   video: ['src', 'poster'],
  //   source: ['src'],
  //   img: ['src'],
  //   image: ['xlink:href', 'href'],
  //   use: ['xlink:href', 'href'],
  // },
  cached: number
  temps: number
  ssrHelpers?: symbol[] //这里存放的是ssr模式import需要引入的方法
  codegenNode?: TemplateChildNode | JSChildNode | BlockStatement //
  transformed?: boolean //是否被转换过

  // v2 compat only
  filters?: string[] //vue2才支持
}
```
## generate(ast, resolvedOptions)对ast进行递归处理从根节点开始。

* 1.对helpers遍历，将组件import内容添加到context.code中，非ssr从helprs中添加，
    ssr从ssrHelpers中添加

* 2.对imports遍历，将import内容添加到context.code //非module模式下,没有这步

* 3.对hoists遍历，将静态节点添加到context.code中

* 4.添加 export default render|ssrRender 函数和参数

* 5.useBlock

* 6.添加components里面

* 7.添加directives里面

* 8.添加filters里面
* 9.添加temps
* 10.添加子节点（<template>）里面内容，递归遍历
* 11.收尾添加'}'括号
  

title: vue中父子组件交互的场景
date: 2018-03-16 17:31:18
tags:
---

vue中父子组件的常见交互场景

# 父组件的data改变控制子组件的渲染改变

场景：

子组件暴露props，父组件将data传递到子组件的props上。

当data发生改变，子组件是否会跟着刷新渲染？

通过v-bind

子组件代码

```vue
<template>
<div>
  <p>String</p>
  <span>
    {{childText}}
  </span>
  <p>Number</p>
  <span>
    {{childNumber}}
  </span>
  <p>object</p>
  <span>
    {{childObj.text}}
  </span>
  <p>Array</p>
  <span v-for="i in childArray" :key="i">{{i}}</span>


</div>
</template>
<style scoped>
p {
  font-weight:bolder;
}
</style>

<script>
export default {
  props: {
    childText: {
      type: String
    },
    childNumber: {
      type: Number
    },
    childObj: {
      type: Object
    },
    childArray: {
      type: Array
    }
  }
};
</script>

```

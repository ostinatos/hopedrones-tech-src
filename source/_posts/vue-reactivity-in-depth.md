title: vue-reactivity-in-depth
date: 2018-04-06 20:38:22
tags:
---

# vue的响应式实现原理

# 实现vue的响应式原型

## 细节

### why target stack?

# vue的响应式机制的“坑”

vue的官方文档中，专门花了一个章节讲述vue的响应式机制的原理和注意事项：

https://cn.vuejs.org/v2/guide/reactivity.html

当中提到的要点：

* **不能动态添加根级响应式属性 (root-level reactive property)**

* **对于提早声明了的响应式属性，若需要添加删除属性，需要使用vue提供的set API**

这两点，本质都是由于vue要实现属性的响应式，依赖的是给属性注入的reactiveGetter和reactiveSetter。

对于根级响应式属性，reactiveGetter和reactiveSetter的注入是通过在初始化vue组件实例的时候，遍历data来实现的。而当实例被初始化后，就没有时机再去监控根级响应式属性的增加。因此就有了第一条要点。

而对于已经存在的响应式对象，若要添加删除属性，则需要通过vue提供的API来实现getter和setter的注入，这就有了第二条要点。


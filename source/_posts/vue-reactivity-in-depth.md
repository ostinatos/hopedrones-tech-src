title: 深入vue的响应式实现
date: 2018-04-06 20:38:22
tags: [vue, frontend, reactivity, javascript, framework, 前端, 响应式]
---



使用vue有一段时间，对于响应式（reactivity）这个特性，也是踩过不少坑，自然就想了解一下vue源码中是如何实现的，以求更好的理解和使用这个框架。

在google上搜到了一篇很好的文章，通俗易懂的讲清楚了vue源码中关于响应式的核心实现代码原理。文章在此：

[How to Build Your Own Reactivity System](https://hackernoon.com/how-to-build-your-own-reactivity-system-fc48863a1b7c)

基于此就顺手写了个小原型，感受一下这个实现，代码在此：

https://github.com/ostinatos/reactivity-prototype

这篇文章也就简单总结一下实现这个原型的关键点，以及写了这个原型后本人对vue中提及的响应式的“坑”的一些理解。

# 实现vue的响应式原型的关键点

原型的关键图示如下：

{% asset_img vue-reactivity.png vue-reactivity-prototype %}

## 关键点

### 使用defineProperty为data注入getter/setter

参考reactify.js的内容。这里深度遍历输入的data对象，对每一个属性都使用defineProperty注入一个getter和setter。

### 观察者模式 (observer pattern)

vue的响应式机制主要使用了观察者模式，Dep.js定义的Dep类可以理解为一个observable。

在遍历data的时候，会为每个属性都创建一个Dep实例，也就是observable实例。然后通过defineProperty注入的getter/setter中会调用observable实例的方法。

具体来看看Dep类都有哪些关键属性和方法：

一个**观察者清单（subscriber list/ observer list）**。若有watcher和当前属性关联，则会被加入此观察者清单。注意，虽然说是“清单”，但实际实现的时候用的是**集合（Set）**，目的是重复调用也不会导致重复。

**notify()**做的就是循环调用观察者清单的update()方法。这是观察者模式的典型套路了。

**depend()**方法的作用是将当前被估值的watcher加入观察者清单。

### watcher

vue中模版内的表达式，或者计算属性，底层对应的应该就是watcher了。

### Dep.target
这个Dep类的静态变量，作用是纪录当前正在被估值的watcher（们）。

注意，在[实际的vue代码](https://github.com/vuejs/vue/blob/v2.5.17-beta.0/src/core/observer/dep.js#L49)（本文参考的源码版本是2.5.17）中，这个变量还额外用了一个辅助的stack来管理。而如果只是实现响应式的原型，可以不用stack也可以的。

至于为什么要用stack，segmentfault这个问答有分析，还是比较有道理的：

[Vue Dep.target为什么需要targetStack来管理？](https://segmentfault.com/q/1010000010095427)

# 对vue的响应式机制的“坑”的理解

vue的官方文档中，专门花了一个章节讲述vue的响应式机制的原理和注意事项：

https://cn.vuejs.org/v2/guide/reactivity.html

当中提到的要点：

* **不能动态添加根级响应式属性 (root-level reactive property)**

* **对于提早声明了的响应式属性，若需要添加删除属性，需要使用vue提供的set API**

这两点，本质都是由于vue要实现属性的响应式，依赖的是给属性注入的reactiveGetter和reactiveSetter。

## 不能动态添加根级响应式属性

例子，引用自vue的官方文档：

```
var vm = new Vue({
  data:{
  a:1
  }
})

// vm.a 是响应的

vm.b = 2
// vm.b 是非响应的
```

原因在于，对于根级响应式属性，reactiveGetter和reactiveSetter的注入是通过在初始化vue组件实例的时候，遍历data来实现的。而当实例被初始化后，就没有时机再去监控根级响应式属性的增加。因此就有了第一条要点。

## 使用vue提供的API添加删除属性

例子：
```
var vm = new Vue ({
    data: {
        someObj:{
            a:"foo"
        }
    }
})

//vm.someObj.a是响应式的。

Vue.set(vm.someObj, 'b', 'bar');
// vm.someObj.b 也是响应式的。

vm.someObj.c = "no";
// vm.someObj.c 是非响应式的。
```

这条规则背后的原因是，若不通过vue提供的API来进行添加属性，则vue框架无法给新增的属性添加reactiveGetter和reactiveSetter，也就无法实现响应式。


title: vue中的Object/Array响应式陷阱
date: 2018-05-25 22:05:58
tags: [vue, reactivity, object, array, javascript, 前端, 响应式]
---

Vue中的响应式(reactivity)是个好东西，它简化了代码，实现了前端的数据驱动(data-driven)，提升了代码的可维护性。

然而，这个响应式特性也是有不少[要注意的地方](https://cn.vuejs.org/v2/guide/reactivity.html)，特别是对于对象和数组，一不留神就容易掉进“陷阱”：为什么数据变了，界面却没有变化？

这篇文章尝试列举一些针对对象和数组的常见响应式场景，试着分析为什么某些场景下响应式“失效”了，背后的原理是什么。

# 例子源码
请看这里：[vue-playground](https://github.com/ostinatos/vue-playground)

基于vue-cli 3生成的vue demo工程。安装完依赖后 npm run serve 就可以运行。

代码结构：
```
└── src
    ├── App.vue
    ├── assets
    │   └── logo.png
    ├── components
    │   └── HelloWorld.vue
    ├── main.js
    └── object-reactivity
        ├── components
        │   ├── nested-component.vue
        │   └── nested-readonly-component.vue
        └── object-reactivity.vue

```


object-reactivity.vue是入口组件。

nested-component.vue是其中一个子组件，nested-readonly-component.vue是另一个子组件。

两者的区别是nested-readonly-component内直接绑定props进行展示。而nested-component的界面绑定的是一个内部data。

# 场景描述

要测试的场景并不复杂。在入口组件object-reactivity中定义了data，分别有一个类型为对象的basket，和类型为数组的list.

*object-reactivity.vue*

```javascript
data(){
    return {
        foods:{
            basket:{
                apple: "apple in a basket"
            },
            list:[
                "potato","tomato", "pizza"
            ]
        }
    }
}
```

两者都作为参数（props），分别传入到nested-component以及nested-readonly-component中。

```html
<nested-component :basket="foods.basket" :foods="foods.list"></nested-component>
<nested-readonly-component :basket="foods.basket" :foods="foods.list"></nested-readonly-component>
```

然后我们尝试用不同的方式**改变** object-reactivity中的这两个数据，看两个组件实例是否在界面上发生相应的变化。

为了便于在console中进行试验，我们将object-reactivity的foods赋给一个window下的全局变量。

```javascript
mounted(){
    window.foods = this.foods;
}
```

执行 npm run serve 将工程跑起来，然后在控制台分别尝试如下语句，你能预见界面会发生什么变化吗？

```javascript
// 1
window.foods.basket.apple = "new apple in the basket."
// 2
Vue.set(foods.basket, "apple", "new apple");
// 3
window.foods.basket = {apple: "new apple in the basket"};
// 4
Vue.set(foods, "basket", {apple:"new apple"});

```


# 原理解释

## 初始状态

{% asset_img object-reactivity.png object-reactivity %}

上图展示的是初始情况下3个组件和监控的数据（foods.basket）的关系。请结合源码进行理解。

可以看到三个组件都分别有界面的watcher。watcher负责的是监控数据的变化，按需更新界面（DOM）。

值得注意的是，三个watcher所监控的对象并不尽相同：

- object-reactivity.vue以及nested-readonly-component的界面watcher所监控的是橙色的数据对象。
- nested-component的界面watcher所监控的是则是自己内部的data: innerBasket。

## 可以维持响应式的情况

经过测试，你会发现如下两个语句执行后，界面中三处都发生了相应的变化。

```javascript
// 1
window.foods.basket.apple = "new apple in the basket."
// 2
Vue.set(foods.basket, "apple", "new apple");
```

| 界面 | 响应式维持？|
|---|----|
| object-reactivity.vue | yes |
| nested-component.vue | yes |
| nested-readonly-component.vue | yes |

原因是什么？

可以结合上面的图来看，三个界面watcher实际上监控的都是中间橙色的那个对象。

一旦这个对象的属性被修改，就会触发这个对象属性的**reactive setter** （vue给数据对象注入的，如不了解可以参看我的这篇文章: [深入vue的响应式实现](http://ostinatos.github.io/hopedrones-tech/2018/04/06/vue-reactivity-in-depth/)），从而通知三个界面watcher对各自的界面进行修改。

## 丢失响应式的情况

然而，执行以下两个语句中的任意一个，你会发现，nested-component.vue的响应式丢失了。

```javascript
// 3
window.foods.basket = {apple: "new apple in the basket"};
// 4
Vue.set(foods, "basket", {apple:"new apple"});
```

| 界面 | 响应式维持？ |
|---|---|
| object-reactivity.vue | yes |
| nested-component.vue | no |
| nested-readonly-component.vue | yes |

 怎么解释呢？还是通过一张图解释一下。

{% asset_img object-reactivity-2.png object-reactivity %}

从示意图上可以看到，当执行以上语句后：

**object-reactivity.vue**

object-reactivity.vue的界面watcher以及data的指向都发生了变化，指向了一个新的对象；


**nested-readonly-component.vue**

nested-readonly-component.vue的界面watcher指向的还是自己的basket属性，然而basket属性指向的是那个新的对象；
这里值得留意的一点是，object-reactivity.vue中的foods.basket发生了变化，因此**触发了对nested-readonly-component.vue的属性的重新赋值**。而由于nested-readonly-component的界面渲染（也就是render函数）是依赖于basket属性，因此basket属性的变化又触发了render()函数的重新执行，从而界面就得到了更新。

**nested-component.vue**

nested-component.vue的basket属性也发生了变化（就像nested-readonly-component的情况），也指向了新的对象，然而**innerBasket指向的还是原来的对象，而其界面watcher指向的又是innerBasket**。这也就可以解释为什么nested-component的响应式丢失了，因为其界面watcher所监控的还是旧的那个对象。

{% asset_img nested-component.png nested-component %}

# lesson learned

## 属性对应的数据发生变化，vue会给属性重新赋值

其实本质就是**监控数据变化 + 执行render函数**

object-reactivity.vue的数据（foods）发生变化，触发object-reactivity.vue的render函数重新执行，自然也就把新的数据作为props赋值给了子组件。

## 理解界面watcher的指向

响应式的核心是通过给数据注入getter/setter从而让界面作为观察者（watcher）监控到数据的变化，自动更新界面。响应式“丢失”的情况，往往就是界面watcher的指向与预期的数据对象不匹配造成。
title: vue中对象作为属性的响应式行为
date: 2019-03-02 12:02:20
tags: [vue, reactivity, object, javascript, 前端, 响应式]
---

# 场景：

1. 更改对象属性的引用
2. 更改对象属性内的属性

# 需要考虑的因素

对象中的属性是否在传递props前存在

# 考察：

- watcher的行为
- computed 属性的行为

# 演示代码

请看这里：[vue-playground](https://github.com/ostinatos/vue-playground)

找到main.js，将以下这句取消注释，将其它App的import 注释掉

```javascript
import App from './object-property/main.vue'
```

# 结论总结

{% asset_img object-prop-reactivity.png object-prop-reactivity %}
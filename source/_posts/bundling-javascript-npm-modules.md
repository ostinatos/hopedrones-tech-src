title: 打包发布js库时对于异步依赖的处理
date: 2020-01-24 18:26:54
tags:
---

# 背景

本人在公司内主要负责一些前端公共组件的设计开发和维护，免不了涉及将javascript代码打包为模块，并将其发布到npm registry的工作。

在这当中，其实会遇到不少的细节问题，值得总结分享一下，其中一个最近碰到的比较共性的问题，就是**对于js库的异步依赖如何处理的问题**。

场景是这样的：

假设你负责维护的，是一个叫 `foo` 的公共组件。

随着时间推移，你维护的这个公共组件foo功能越来越丰富，能够满足越来越多的业务场景，越来越多的应用依赖了你维护的这个组件。

但慢慢的，使用你的组件的开发团队对你的组件提出了这些意见：

> 我们的应用中90%的用户并没有用到你组件的a功能，能否默认不加载？否则对我们应用的初始性能会造成影响。

这种情况，对于维护组件的你应该如何实现呢？下面我们来一起看看。

# 场景和期望

假设你维护的组件叫 `async-dep-module-demo` ，当中提供了一个复杂的计算功能，叫 `asyncAdd` 

为了完成此功能，需要引入一个体积比较大的代码。为了性能，我们使用es6的异步import语法。

``` javascript
// 引入一些同步依赖
import {
    syncDep
} from './sync-dep'

function asyncAdd(x, y) {
    // 调用同步依赖
    syncDep();
    // 复杂的逻辑，代码体积较大，通过异步import引入
    return import('./async-dep').then(({
        asyncDep
    }) => {
        asyncDep();
        return x + y;
    })

}

export {
    asyncAdd
}
```

使用你的组件库的应用，会用类似如下的代码来使用：

``` javascript
import {
    asyncAdd
} from 'async-dep-module-demo'

asyncAdd(2, 3).then(res => {
    console.log("invoke asyncAdd() ", res)

})
```

现在使用你的组件库的应用，对你的组件库的要求是，如果应用还没有调用aysncAdd方法，则不要加载那部分代码量大的复杂逻辑（ `async-dep` ）。

# 实现

## 使用webpack打包你的组件库？

说起前端的打包工具（bundler），大家最熟悉的可能就是 `webpack` 。那么webpack能达到我们的期望吗？

webpack支持打包js库，输出的模块规范有好几种，可以参考这里：
https://webpack.js.org/configuration/output/#outputlibrarytarget

常用的可能有 `umd` ， `amd` ， `commonjs` 等。

假设我们采用umd格式输出，对应的webpack配置文件大概长这个样子：

``` javascript
// webpack.config.js
const path = require('path');

module.exports = {
    entry: './src/index.js',
    output: {
        filename: 'demo.umd.js',
        path: path.resolve(__dirname, 'dist'),
        library: "moduleDemo",
        // 指定输出的模块规范
        libraryTarget: "umd"
    },
    // for better code inspection, we use "development" mode here.
    mode: "development"
};
```

用以上配置文件执行编译后，查看一下编译输出的文件，对于我们所关心的异步import的语法，webpack是编译成如下的代码：

```javascript
function jsonpScriptSrc(chunkId) {
	return __webpack_require__.p + "" + chunkId + ".demo.umd.js"
}
```

这个方法在哪里用呢？可以找到，是在下面这个方法`__webpack_require__`使用：


``` javascript
// The require function
function __webpack_require__(moduleId) {
    // Check if module is in cache
    if (installedModules[moduleId]) {
        return installedModules[moduleId].exports;
    }
    // Create a new module (and put it into the cache)
    var module = installedModules[moduleId] = {
        i: moduleId,
        l: false,
        exports: {}
    };
    // Execute the module function
    modules[moduleId].call(module.exports, module, module.exports, __webpack_require__)
    // Flag the module as loaded
    module.l = true;
    // Return the exports of the module
    return module.exports;
}
// This file contains only the entry chunk.
// The chunk loading function for additional chunks
__webpack_require__.e = function requireEnsure(chunkId) {
        var promises = [];
        // JSONP chunk loading for javascript
        var installedChunkData = installedChunks[chunkId];
        if (installedChunkData !== 0) { // 0 means "already installed".
            // a Promise means "currently loading".
            if (installedChunkData) {
                promises.push(installedChunkData[2]);
            } else {
                // setup Promise in chunk cache
                var promise = new Promise(function(resolve, reject) {
                    installedChunkData = installedChunks[chunkId] = [resolve, reject];
                });
                promises.push(installedChunkData[2] = promise);
                // start chunk loading
                var script = document.createElement('script');
                var onScriptComplete;
                script.charset = 'utf-8';
                script.timeout = 120;
                if (__webpack_require__.nc) {
                    script.setAttribute("nonce", __webpack_require__.nc);
                }
                script.src = jsonpScriptSrc(chunkId);
                // create error before stack unwound to get useful stacktrace later
                var error = new Error();
                onScriptComplete = function(event) {
                    // avoid mem leaks in IE.
                    script.onerror = script.onload = null;
                    clearTimeout(timeout);
                    var chunk = installedChunks[chunkId];
                    if (chunk !== 0) {
                        if (chunk) {
                            var errorType = event && (event.type === 'load' ? 'missing' : e
                                var realSrc = event && event.target && event.target.src; error.message = 'Loading chunk ' + chunkId + ' failed.\n(' + er error.name = 'ChunkLoadError'; error.type = errorType; error.request = realSrc; chunk[1](error);
                            }
                            installedChunks[chunkId] = undefined;
                        }
                    };
                    var timeout = setTimeout(function() {
                        onScriptComplete({
                            type: 'timeout',
                            target: script
                        });
                    }, 120000);
                    script.onerror = script.onload = onScriptComplete;
                    document.head.appendChild(script);
                }
            }
            return Promise.all(promises);
        };
```

可以看到，webpack编译出来的代码，对于异步import的模块，是编译为动态创建script标签，加载对应分割出来的文件。

如果我们的组件输出的是这种代码，当应用引入我们的依赖时，会出现什么情况？

{% asset_img error-loading-chunk.png error-loading-chunk %}

可以看到加载分割文件失败了，原因是：

- 所希望加载的分割文件，在宿主应用中并没有

要解决这个问题，可以通过在宿主应用中，使用类似`copy-webpack-plugin`这种插件，将node_modules下的依赖文件拷贝到运行目录中解决。但是这样的方案并不优雅：使用个第三方依赖，还要在编译构建的脚本中增加配置，也太麻烦啰。

## rollup

rollup 和 webpack的最大不同之处在于，rollup更适合于用来打包js库[2]。

rollup可以支持将js库打包为es module（esm）格式的js。

下面我们来试试使用rollup来打包我们的js库。

rollup配置文件大概长这个样子：

```javascript
//rollup.config.js
module.exports = {
    input: 'src/index.js',
    output: {
        // 设置输出格式为esm
        format: 'esm',
        dir: "dist-esm"
    }
};
```

看看rollup打包后的文件长什么样子：

```javascript
function syncDep() {
    console.log("[sync-dep] do some job.");
}

// sync import

function asyncAdd(x, y) {
    syncDep();
    return import('./async-dep-c29b934b.js').then(({ asyncDep }) => {
        asyncDep();
        return x + y;
    })
    
}

export { asyncAdd };

```

啥？初步一看，和我们写的源码没什么区别呀？

这就对了。

因为我们编写源码的时候，用的就是es module的语法（import, export）。

编译目标是es module，那当然就变化不大啰。

细看一下，其实还是有变化的，异步import的文件名变成了编译后的文件名，带上了hash值。

### 改变组件的入口

使用rollup编译完，在 `package.json` 中设置main指向rollup打包出来的 `es module` 格式的文件。

``` json
{
    "main": "dist-esm/index.js"
}
```

或者，除了指定main外，也可以指定module字段，也有同样的效果。

``` json
{
    "module": "dist-esm/index.js"
}
```

### 运行效果

{% asset_img loading-chunk.png loading-chunk %}

这次没有报错了，从网络请求中，也可以看到有一个正常的分割文件的请求，经过查看，这个分割文件正是我们的js库中希望分割的那部分代码。

使用[webpack bundle analyzer](https://github.com/webpack-contrib/webpack-bundle-analyzer)分析一下宿主应用的模块组成：

{% asset_img bundle.png bundle %}

可以看到最右边的 `0.app.js`正是我们希望分割的`async-dep`模块啰。大功告成。

# 总结

若希望你开发的js模块，包含按需异步加载的代码，使得宿主应用使用时，能够按需加载，请注意如下几点：

- js库的源码中使用动态import语法（dynamic import）加载需要按需加载的代码

```javascript
import('./xxx.js').then()
```

- 不能使用webpack打包js库
    
    因为使用webpack打包js库时，**对于需要懒加载的资源，webpack是直接生成了动态拼接script标签的代码**，而对于library来说，这样的代码对于宿主应用的使用是很麻烦的

- 使用rollup打包js库（js library）

    使用rollup打包js library的好处是，对于library中需要懒加载的资源，rollup编译为了异步加载chunk的代码，而且用的不是动态拼接script标签的方式，这样有助于宿主应用使用js library时进行分割。

    具体例子，可以看看rollup官网的这个例子：

    [rollup dynamic import compilation REPL example](https://rollupjs.org/repl/?version=1.30.0&shareable=JTdCJTIybW9kdWxlcyUyMiUzQSU1QiU3QiUyMm5hbWUlMjIlM0ElMjJtYWluLmpzJTIyJTJDJTIyY29kZSUyMiUzQSUyMiUyRiolMjBEWU5BTUlDJTIwSU1QT1JUUyU1Q24lMjAlMjAlMjBSb2xsdXAlMjBzdXBwb3J0cyUyMGF1dG9tYXRpYyUyMGNodW5raW5nJTIwYW5kJTIwbGF6eS1sb2FkaW5nJTVDbiUyMCUyMCUyMHZpYSUyMGR5bmFtaWMlMjBpbXBvcnRzJTIwdXRpbGl6aW5nJTIwdGhlJTIwaW1wb3J0JTIwbWVjaGFuaXNtJTVDbiUyMCUyMCUyMG9mJTIwdGhlJTIwaG9zdCUyMHN5c3RlbS4lMjAqJTJGJTVDbmlmJTIwKGRpc3BsYXlNYXRoKSUyMCU3QiU1Q24lNUN0aW1wb3J0KCcuJTJGbWF0aHMuanMnKS50aGVuKGZ1bmN0aW9uJTIwKG1hdGhzKSUyMCU3QiU1Q24lNUN0JTVDdGNvbnNvbGUubG9nKG1hdGhzLnNxdWFyZSg1KSklM0IlNUNuJTVDdCU1Q3Rjb25zb2xlLmxvZyhtYXRocy5jdWJlKDUpKSUzQiU1Q24lNUN0JTdEKSUzQiU1Q24lN0QlMjIlMkMlMjJpc0VudHJ5JTIyJTNBdHJ1ZSU3RCUyQyU3QiUyMm5hbWUlMjIlM0ElMjJtYXRocy5qcyUyMiUyQyUyMmNvZGUlMjIlM0ElMjJpbXBvcnQlMjBzcXVhcmUlMjBmcm9tJTIwJy4lMkZzcXVhcmUuanMnJTNCJTVDbiU1Q25leHBvcnQlMjAlN0JkZWZhdWx0JTIwYXMlMjBzcXVhcmUlN0QlMjBmcm9tJTIwJy4lMkZzcXVhcmUuanMnJTNCJTVDbiU1Q25leHBvcnQlMjBmdW5jdGlvbiUyMGN1YmUlMjAoeCUyMCklMjAlN0IlNUNuJTVDdHJldHVybiUyMHNxdWFyZSh4KSUyMColMjB4JTNCJTVDbiU3RCUyMiUyQyUyMmlzRW50cnklMjIlM0FmYWxzZSU3RCUyQyU3QiUyMm5hbWUlMjIlM0ElMjJzcXVhcmUuanMlMjIlMkMlMjJjb2RlJTIyJTNBJTIyZXhwb3J0JTIwZGVmYXVsdCUyMGZ1bmN0aW9uJTIwc3F1YXJlJTIwKCUyMHglMjApJTIwJTdCJTVDbiU1Q3RyZXR1cm4lMjB4JTIwKiUyMHglM0IlNUNuJTdEJTIyJTJDJTIyaXNFbnRyeSUyMiUzQWZhbHNlJTdEJTVEJTJDJTIyb3B0aW9ucyUyMiUzQSU3QiUyMmZvcm1hdCUyMiUzQSUyMmVzbSUyMiUyQyUyMm5hbWUlMjIlM0ElMjJteUJ1bmRsZSUyMiUyQyUyMmFtZCUyMiUzQSU3QiUyMmlkJTIyJTNBJTIyJTIyJTdEJTJDJTIyZ2xvYmFscyUyMiUzQSU3QiU3RCU3RCUyQyUyMmV4YW1wbGUlMjIlM0ElMjIwMCUyMiU3RA==)

- 设置正确的模块入口

    package.json 设置入口为es module 标准的js。可以通过指定main或者module字段都可以

- 宿主应用可以正常使用webpack来编译使用js库的代码

    从实践结果来看，宿主应用使用webpack编译，能够正确的解析出js库中的异步加载依赖，并将其作为宿主应用的分割文件进行异步懒加载，有效提升加载性能

# 完整参考代码

模块代码参考：

https://github.com/ostinatos/js-playground/tree/master/packages/async-dep-module-demo

宿主应用参考：

https://github.com/ostinatos/js-playground/tree/master/packages/module-host-demo

# 参考资料

[1] [Code-splitting for libraries—bundling for npm with Rollup 1.0](https://levelup.gitconnected.com/code-splitting-for-libraries-bundling-for-npm-with-rollup-1-0-2522c7437697)

讲解如何使用rollup打包出能按需使用的npm模块

[2] [Webpack and Rollup: the same but different](https://medium.com/webpack/webpack-and-rollup-the-same-but-different-a41ad427058c)

比较了webpack和rollup的不同，以及何时该使用哪一个
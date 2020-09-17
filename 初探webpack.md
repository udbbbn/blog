### 初探 webpack

本文所有代码基于 `5.0.0-beta.29` 版本!!


#### 1. 启动 webpack
当我们把webpack的配置文件 作为 **fn** 传入 `webpack` 主执行函数 会执行 `webpack` 方法。(请先往下看)
![20200905004025](https://raw.githubusercontent.com/udbbbn/Img/master/20200905004025.png)

有同学会疑惑，这里不是声明了一个 **fn** 的变量了吗？那这里应该是用 **fn** 变量传入这个`mergeExports`方法才对。
别急，让我们来看看这个 **fn** 方法。

![20200905005627](https://raw.githubusercontent.com/udbbbn/Img/master/20200905005627.png)
![20200905005613](https://raw.githubusercontent.com/udbbbn/Img/master/20200905005613.png)

通过源码我们可以得知，此处使用了一个 **缓存函数** 将 `webpack.js` 给导入并缓存下来作为 **工厂函数** 接收用户传入的 `config`。然后我们来看看 `webpack` 是如何处理这份 `config` 文件的。

#### 2. webpack 初始化

`webpack` 接收到 `config` 文件后，就会执行下面的代码。
```js
const webpack = (( options, callback ) => {
    // ...缩减代码
    compiler = createCompiler(options);
    watch = options.watch;
    watchOptions = options.watchOptions || {};
    // 回调存在的话 直接启动编译
    if (callback) {
        if (watch) {
            compiler.watch(watchOptions, callback);
        } else {
            compiler.run((err, stats) => {
                compiler.close(err2 => {
                    callback(err || err2, stats);
                });
            });
        }
    }
    return compiler;
});
```

通过上面的代码，我们可以看到 `webpack` 返回了一个 `compiler`对象。这个对象从始至终都会存在于整个编译过程的上下文当中。而此处重点便是`createCompiler` 这个方法。下面，让我们来看看它的源码。
![20200908171103](https://raw.githubusercontent.com/udbbbn/Img/master/20200908171103.png)
查看源码可得知，该方法主要做了这些事情：
1. 初始化 `webpack` 对象属性
2. 挂载 `hooks`、挂载第三方插件
3. 创建文件缓存系统、创建文件监视系统
4. 根据编译对象、`devtools` 模式的不同挂载不同的插件处理。

到这里 初始化工作就以及基本完成了

需要注意的是 上图中的 
```js
new WebpackOptionsApply().process(options, compiler)
```
 在这个函数内部 挂载了 `webpack` 编译的核心操作 即 `compilation` 跟 `make` hooks。

![20200911182552](https://raw.githubusercontent.com/udbbbn/Img/master/20200911182552.png)

#### 编译过程

流程： `compiler.run -> compiler.compile`

`compiler.run` 方法简化代码如下

```js
run(callback) {
    const run = () => {
        this.hooks.beforeRun.callAsync(this, err => {
            if (err) return finalCallback(err);

            this.hooks.run.callAsync(this, err => {
                if (err) return finalCallback(err);

                this.readRecords(err => {
                    if (err) return finalCallback(err);

                    this.compile(onCompiled);
                });
            });
        });
    };
    const onCompiled = (err, compilation) => {
        if (err) return finalCallback(err);

        if (this.hooks.shouldEmit.call(compilation) === false) {
            // ... 简化代码
            this.hooks.done.callAsync(stats, err => {
                if (err) return finalCallback(err);
                return finalCallback(null, stats);
            });
            return;
        }

        process.nextTick(() => {
            // ...
            this.emitAssets(compilation, err => {
                if (err) return finalCallback(err);
                // ...
                this.emitRecords(err => {
                    if (err) return final
                    // ...
                    this.hooks.done.callAsync(stats, err => {
                        // ...
                    });
                });
            });
        });
    };
}
```

`compiler.compile` 的简化代码如下

```js
compile(callback) {
    const params = this.newCompilationParams();
    this.hooks.beforeCompile.callAsync(params, err => {
        if (err) return callback(err);

        this.hooks.compile.call(params);
        // 根据参数 创建一个 compilation
        const compilation = this.newCompilation(params);

        this.hooks.make.callAsync(compilation, err => {
            if (err) return callback(err);

            this.hooks.finishMake.callAsync(compilation, err => {
                if (err) return callback(err);

                process.nextTick(() => {
                    compilation.finish(err => {
                        if (err) return callback(err);

                        compilation.seal(err => {
                            if (err) return callback(err);

                            this.hooks.afterCompile.callAsync(
                                compilation, err => {
                                    logger.timeEnd(
                                        "afterCompile hook"
                                    );
                                    if (err) return callback(
                                        err);

                                    return callback(null,
                                        compilation);
                                });
                        });
                    });
                });
            });
        });
    });
}
```

`compiler.run`方法触发了以下 `hooks`:
```js
beforeRun => run => needAdditionalPass => additionalPass => done => afterDone
```
`compiler.compile`方法触发了以下 `hooks`:
```js
beforeCompile => compile => make => finishMake => afterCompile
```
由源码可见，这两个方法调用了大量的 `hooks`，这些 `hooks`的挂载得益于 `tapable` 这个库。没有这个库的加持，回调地狱会很恐怖。(虽然现在各种回调已经很不方便阅读了)。

我们主要来看 `make` 这个钩子具体做了些什么。

```js
compiler.hooks.make.tapAsync("EntryPlugin", (compilation, callback) => {
    const {
        entry,
        options,
        context
    } = this;

    const dep = EntryPlugin.createDependency(entry, options);
    compilation.addEntry(context, dep, options, err => {
        callback(err);
    });
});
```
主要是调用了 `compilation.addEntry` 方法，功能是初始化模块一些信息并传递给 **模块链** 。一系列的函数最后将配置文件中的入口文件都 "编译" 成模块(即 `normalModule` 实例)。函数调用顺序如下：

```js

这里将 this 指向 compilation
这里将 aq 指向 AsyncQueue 实例

this.addEntry => this._addEntryItem => 
this.addModuleChain  => this.handleModuleCreation => 
this.factorizeModule => this.factorizeQueue.add =>
// 这里需要注意 异步队列是由 AsyncQueue 这个类去维护的。
// factorizeQueue.add 方法里存在异步执行队列中的任务
// 执行顺序：aq._ensureProcessing => aq._startProcessing => aq._processor
// 此处的 aq._processor 即 this._factorizeModule
this._factorizeModule => factory.create =>
// 此处的 factory 指的是 NormalModuleFactory
// factory.create 触发了 NormalModuleFactory 在 constructor 就挂载的一个 hooks
// 代码如下
// this.hooks.factorize.callAsync(resolveData, (err, module) => { // ... }
// 这里是 new normalModule 实例的地方
this.addModule => this.buildModule => this.buildQueue.add => 
// 这里跟刚刚一样 也是一个 异步队列
this._buildModule => module.needBuild => 
// 此处的 module 实际上为 normalModule 实例
module.build => module.doBuild => runLoaders
// const { runLoaders } = require("loader-runner");
```

这里还需要注意的是 `compilation` 中不同队列的 `processor` 是不同函数。

![20200911190925](https://raw.githubusercontent.com/udbbbn/Img/master/20200911190925.png)
>图片中的代码位置在 `compilation` 文件中。

因为 `webpack` 中大量使用了**异步**、**hooks** 等方式，比较乱。建议配合着源码调试阅读。 
到这一步，代表我们的代码已经经过 `loader` 编译成标准的 `js` 代码了。接下来就是解析标准 `js` 代码。在 `normalModule` 中有如下代码：

![20200915130719](https://raw.githubusercontent.com/udbbbn/Img/master/20200915130719.png)

当前的 `this.parser` 为 `JavascriptParser` 实例。再查看了 `JavascriptParser` 类中的方法可以得知， 该类引入了 `acorn` 去对 `js` 进行语法解析。

处理完语法解析后，会将 `chunk` 添加到 `compliation.chunks`。
```js
seal(callback) {
    // ...
    entry.unshiftChunk(chunk);
    chunk.addGroup(entry);
    entry.setRuntimeChunk(chunk);
    // ...
}
```

再设置 `assets`。

```js
// 将静态资源保存到 compilation.assets
emitAssets(compilation, callback) {
    let outputPath;

    const emitFiles = err => {
        // ...
        if (err) return callback(err);

        const assets = compilation.getAssets();
        compilation.assets = { ...compilation.assets };
        // ...
    }
```

`emitAssets` 该方法还会将文件输出到指定目录。

![20200915135616](https://raw.githubusercontent.com/udbbbn/Img/master/20200915135616.png)

本文对 `webpack` 编译过程仅初步分析。后续还需要继续深入理解！

#### 从 webpack 编译结果看 模块之间的关系
源代码：
```js
import is from "object.is";
const a = 2133;
```
webpack 编译后的代码：
```js
(() => {
    // 用于存储 模块与模块实际代码的 键值对对象
    var __webpack_modules__ = {
        "./src/index.js": (
            __unused_webpack_module,
            __webpack_exports__,
            __webpack_require__
        ) => {
            "use strict";
            __webpack_require__.r(__webpack_exports__);
            var object_is__WEBPACK_IMPORTED_MODULE_0__ = __webpack_require__(
                "../node_modules/object.is/index.js"
            );
            // 这里将模块加载并保存了一份 暂时不知道具体作用
            var object_is__WEBPACK_IMPORTED_MODULE_0___default = /*#__PURE__*/ __webpack_require__.n(
                object_is__WEBPACK_IMPORTED_MODULE_0__
            );

            const a = 2133;
        },

        "../node_modules/object.is/index.js": (
            module,
            __unused_webpack_exports,
            __webpack_require__
        ) => {
            module.exports =
                typeof Object.is === "function" ?
                Object.is :
                __webpack_require__( /*! ./is */ "../node_modules/object.is/is.js");
        },

        "../node_modules/object.is/is.js": module => {
            var is = function (x, y) {
                if (x === y) {
                    return x !== 0 || 1 / x === 1 / y;
                } else {
                    return x !== x && y !== y;
                }
            };

            module.exports = is;
        }
    };

    // 模块缓存器
    var __webpack_module_cache__ = {};

    // 模块加载器
    // 每次先判断当前模块是否加载过 并存放在 cache 中
    // 若存在直接返回 否则加载模块并保存 cache 中
    function __webpack_require__(moduleId) {
        if (__webpack_module_cache__[moduleId]) {
            return __webpack_module_cache__[moduleId].exports;
        }

        var module = (__webpack_module_cache__[moduleId] = {
            exports: {}
        });

        __webpack_modules__[moduleId](module, module.exports, __webpack_require__);

        return module.exports;
    }

    // 返回一份导出函数
    (() => {
        __webpack_require__.n = module => {
            var getter =
                module && module.__esModule ? () => module["default"] : () => module;
            __webpack_require__.d(getter, {
                a: getter
            });
            return getter;
        };
    })();

    // 将 definition 的属性复制给 __webpack_require__.n 中的 getter
    (() => {
        __webpack_require__.d = (exports, definition) => {
            for (var key in definition) {
                if (
                    __webpack_require__.o(definition, key) &&
                    !__webpack_require__.o(exports, key)
                ) {
                    Object.defineProperty(exports, key, {
                        enumerable: true,
                        get: definition[key]
                    });
                }
            }
        };
    })();

    // 判断 obj 是否有 prop 属性
    (() => {
        __webpack_require__.o = (obj, prop) =>
            Object.prototype.hasOwnProperty.call(obj, prop);
    })();

    // 加载模块时 给模块添加一个标示 __esModule 若有 Symbol.toStringTag 则一并设置
    // 作用是让 Object.prototype.toString.call(exports) 时返回 [Object Module] 的结果
    (() => {
        __webpack_require__.r = exports => {
            if (typeof Symbol !== "undefined" && Symbol.toStringTag) {
                Object.defineProperty(exports, Symbol.toStringTag, {
                    value: "Module"
                });
            }
            Object.defineProperty(exports, "__esModule", {
                value: true
            });
        };
    })();

    __webpack_require__("./src/index.js");
})();
```

从上面的代码，我们不难看出。`webpack` 是将各个模块都打包到 `js` 对象中，用文件路径来当 **索引** ，值则是模块本身的 js 代码。需要加载哪些模块就调用 `__webpack_require__` 方法加载并缓存到 `cache` 对象中。

#### 最后
有趣的是你甚至可以在 `webpack 5` 的版本里找到一些 `webpack 6` 将会删除的代码。如下：
![20200911162731](https://raw.githubusercontent.com/udbbbn/Img/master/20200911162731.png)

#### 参考
[loader pitch的参考](https://www.jianshu.com/p/9dfb8e18e76d)
[了解 chunk 的概念](https://juejin.im/post/6844903889393680392)
[编译流程参考](https://juejin.im/post/6844903987129352206#heading-7)
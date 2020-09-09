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
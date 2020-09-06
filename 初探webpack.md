### 初探 webpack

本文所有代码基于 `5.0.0-beta.29` 版本!!


#### 1. 启动 webpack
当我们把webpack的配置文件 作为 **fn** 传入 webpack主执行函数 会执行 `webpack` 方法。(请先往下看)
![20200905004025](https://raw.githubusercontent.com/udbbbn/Img/master/20200905004025.png)
> 文件名： webpack/lib/index.js

有同学会疑惑，这里不是声明了一个 **fn** 的变量了吗？那这里应该是用 **fn** 变量传入这个`mergeExports`方法才对。
别急，让我们来看看这个 **fn** 方法。

![20200905005627](https://raw.githubusercontent.com/udbbbn/Img/master/20200905005627.png)
![20200905005613](https://raw.githubusercontent.com/udbbbn/Img/master/20200905005613.png)

通过源码我们可以得知，此处使用了一个 **缓存函数** 将 `webpack.js` 给导入并缓存下来
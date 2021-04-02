前言：笔者计划读 `react` 很久了，却一直没有完整的实施。后来偶然间发现 `fre` 框架，并了解到作者似乎并不是科班出身，但技术十分了得(个人认为)。

查看了 `fre` 早期的 `commit`，发现那时候其作者也是一边学习他人代码一边写的该库。

于是，笔者也决定开始尝试写一个属于自己的 `react-like` 库！

--

## 虚拟DOM 与 渲染

### 准备工作

使用零配置打包工具 parcel

` yarn add parcel-bundler@1.12.3 -D`

这里需要注意添加的版本号 **@1.12.3**，不指定版本号的话在 **1.12.4** 在启动项目时会报
`Invalid Version: undefined` 的错误。还有另一种方式就是使用 `parcel@next`。

这里我们使用 `parcel@1.12.3` 版本。

建立 `index.tsx` 和 `index.html` 文件，并在 `index.html` 文件中引入 `index.tsx`。

需要注意 babel 的配置

**.babelrc**

```js
{
    "presets": ["env"],
    "plugins": [
        ["transform-react-jsx", {
            "pragma": "h"
        }]
    ]
}
```
这里 `transform-react-jsx` 就是将 **jsx** 转换为 **js** 的插件， `pragma` 属性可以指定转换 **jsx** 时的调用方法。

准备工作完成后就可以开始启动项目了。

命令行输入 `npx parcel index.html`。

若为 **parcel@next**，命令行 `npx parcel serve index.html` 启动项目。

### 生成虚拟DOM

![](https://raw.githubusercontent.com/udbbbn/Img/master/20210308220650.png)

根据上图 **babel** 编辑结果我们可以知道 **渲染函数** 的参数大致为 

    h(tag, props, children)

所以我们编写一个最简单的渲染函数

```js
// ...children 表示子级可能会有多个
function h(tag, attrs, ...children) {
    return {
        tag,
        attrs,
        children
    }
}
```
文中 `element` 编译执行结果如下，基本跟我们预想的相同

![](https://raw.githubusercontent.com/udbbbn/Img/master/20210308221932.png)

这种结构也就是我们说的 **虚拟DOM**。

### 渲染

既然已经生成了虚拟DOM，下一步我们就该生成真实DOM并渲染在页面上。

上文中我们已经知道了，babel 会将 jsx 编译，于是乎
```js
ReactDom.render(<div> react-like </div>, document.querySelect('#root'))
```
等同于
```js
ReactDom.render(h('div', null, ' react-like '), document.querySelect('#root'))
```

也就是说 `ReactDom.render` 方法接收两个参数，一个为 **虚拟DOM**，另一个为 渲染容器。

```tsx
const ReactDom = {
    render: (vnode, container: HTMLElement) => {
        // 需要清空容器中的内容
        container.innerHTML = ''
        return render(vnode, container)
    }
}
```

```tsx
function render(vnode, container) {
    // 文字类型 创建返回文字节点
    if (typeof vnode === 'string') {
        const textNode = document.createTextNode(vnode)
        return container.appendChild(textNode)
    }

    const dom = document.createElement(vnode.tag)

    if (vnode.attrs) {
        // 遍历设置属性
        Object.keys(vnode.attrs).forEach(key => {
            setAttribute(dom, key, vnode.attrs[key])
        })
    }

    vnode.children.forEach(child => render(child, dom))

    return container.appendChild(dom)
}
```

因为属性存在多种情况需要处理，故独立成一个函数。

```tsx
function setAttribute(dom, key, val) {
    // 将 className 转 class
    if (key === 'className') key = 'class'

    // 注册事件
    if (/on\w+/.test(key)) {
        const method = key.toLowerCase()
        dom[method] = val
    } else if (key === 'style') {
        // 文字型 style
        if (!val || typeof val === 'string') {
            dom.style.cssText = val || ''
        } else if (val && typeof val === 'object') {
            // 设置style 且数字自动补充 px 单位
            Object.entries(val).forEach(([styleKey, styleVal]) => {
                dom.style[styleKey] = typeof styleKey === 'number' ? styleVal + 'px' : styleVal
            })
        }
    } else {
        // (1) 对此有疑问的请看文章结尾
        if (Reflect.has(dom, key)) {
            dom[key] = val || undefined
        }
        if (val) {
            dom.setAttribute(key, val)
        } else {
            dom.removeAttribute(key)
        }
    }
}
```

到这一步，最基本渲染功能就已经完成了。

### FAQ

1. 为什么不统一使用 `setAttribute` 设置属性

因为 Dom 的 attribute 跟 property 是有区别的，且 setAttribute 参数皆为 **string**，**property** 则可以设置其他数据类型


下图为 `style` 两种方式的结果。
![](https://raw.githubusercontent.com/udbbbn/Img/master/20210309100924.png)

2. 笔者发现，当我们把 **render** 函数放在其他文件导入时，会发现运行报错。

![](https://raw.githubusercontent.com/udbbbn/Img/master/20210308084757.png)

![](https://raw.githubusercontent.com/udbbbn/Img/master/20210308084955.png)

但是根据 `transform-react-jsx` 插件的文档来看，应该不存在这样的问题才对。

当笔者尝试定位后，觉得应该是 `ts` 版本跟某个插件有冲突，同一套代码，使用 `js` 便不会有问题。

再次查询资料得知：

好像是 `babel 7.x` 的 `tree shaking` 将 指定的 `h` 函数给清除了。([babel issues](https://github.com/babel/babel/issues/12585)) 得到的回复是会在 `babel 8` 中处理。

以及 [Parcel-Bug: React not defined (with TypeScript)](https://github.com/parcel-bundler/parcel/issues/1199)

最后暂时的解决方法就是 将 `transform-react-jsx` 的 `pragma` 改为 `React.h` (React对象下的方法)
## 组件 与 生命周期

### 组件基类

回想一下我们使用react的 class组件 是不是都是下面这种形式

`class App extends React.Component { }`

我们就先实现最基础的 `React.Component`

```tsx 
class Component {
    // 这里应该提供范型 但是先 any
    state: Record<string, any>
    props: Record<string, any>

    constructor(props = {}) {
        this.state = {}
        this.props = props
    }

    // 内置 setState 方法
    setState(nextState) {
        this.state = { ...this.state, ...nextState }
        // 渲染组件
        renderComponent(this)
    }
}
```

`renderComponent` 是更新值时渲染的方法，那在这之前还有一次 **初始化**。

### 组件初始化

在上一篇已经介绍了 babel 组件会调用 `render` 方法，所以我们要拓展一下，具体代码如下：

```tsx 
function render(vnode) {
    // ...
    // (1) 对此有疑问的请看文章结尾
    if (typeof vnode.tag === 'function') {
        const component = createComponent(vnode.tag, vnode.attrs)

        setComponentProps(component, vnode.attrs)

        return component.base
    }
    // ...
}
```

`createComponent` 方法为 **创建组件** ，**function** 组件我们构造成 **class** 组件进行初始化，代码如下：
```tsx
function createComponent(component, props) {
    let instance

    // 若为 class 组件 直接 new 并返回实例
    if (component.prototype && component.prototype.render) {
        instance = new component(props)
    } else {
        // function 组件
        instance = new Component(props)
        instance.constructor = component
        // 这里注意不能使用 箭头函数 否则 this 指向会无法指定为 instance
        instance.render = function () {
            return this.constructor(props)
        }
    }

    return instance
}
```

`setComponentProps` 方法作用为设置组件参数以及处理生命周期。
虽然现在 `componentWillMount` `componentReceiveProps` 两个钩子都已经被弃用了，但是我们暂时加上。

```tsx
function setComponentProps(component, props) {
    // 判断是否第一次设置组件入参 调用钩子
    if (!component.base) {
        if (component.componentWillMount) {
            component.componentWillMount()
        }
    } else if (component.componentReceiveProps) {
        component.componentReceiveProps(props)
    }

    component.props = props

    renderComponent(component)
}
```

然后我们来看渲染函数 `renderComponent`，该函数作用为调用组件的 `render` 函数 以及 **将生命周期的函数扔到队列中**，因为形如 `componentDidMount` 的钩子是在 **真实DOM** 渲染后再触发的。

**TODO: 需要注意因为回调是先抛到队列中，所以需要 `bind` 绑定 this 指向，否则 this 指向会丢失**

```tsx
const _renderCallBacks = []

// render 后的回调
export const executeRenderCallBack = () => {
    while (_renderCallBacks.length) {
        const func = _renderCallBacks.shift()
        func()
    }
}
// ...
function renderComponent(component) {
    // 记录组件对应的 DOM
    let base

    const renderer = component.render()

    if (component.base && component.componentWillUpdate) {
        component.componentWillUpdate()
    }

    base = render(renderer)

    // 将注册的事件扔队列中 render 后调用
    // bind 绑定 this 指向
    if (component.base) {
        if (component.componentDidUpdate) _renderCallBacks.push(component.componentDidUpdate.bind(component))
    } else if (component.componentDidMount) {
        _renderCallBacks.push(component.componentDidMount.bind(component))
    }

    if (component.base && component.base.parentNode) {
        component.base.parentNode.replaceChild(base, component.base)
    }

    // component => base
    component.base = base
    // base => component
    base._component = component
}
```

到这一步，组件的初始化、注册生命周期、Dom生成就基本完成了，按上一篇文章的介绍，组件会在 `ReactDom.render` 中被 `appendChlid` 到真实Dom中。

刚刚我们将异步钩子推入队列，所以需要修改一下 `ReactDom.render` 函数

这里把 `render` 分成了两个，主要目的是 将 `container.innerHTML = ''` 这一句代码给分隔开。因为子元素渲染时会遍历渲染，所以只需要在初始化项目时清空容器内容。

```tsx
export const ReactDom = {
    render: (vnode, container: HTMLElement) => {
        container.innerHTML = ''
        _render(vnode, container)
    }
}

const _render = function (vnode, container) {
    container.appendChild(render(vnode))
    // 触发回调
    executeRenderCallBack()
}
```

### 实现效果

![](https://raw.githubusercontent.com/udbbbn/Img/master/20210323145946.png)

![](https://raw.githubusercontent.com/udbbbn/Img/master/20210323145812.gif)

### FAQ
 
1. `vnode.tag` 为什么会等于 `function`

因为是 `class` 组件通过babel编译后，会编译为 `function` 形式，如下：

![](https://raw.githubusercontent.com/udbbbn/Img/master/20210323110428.png)
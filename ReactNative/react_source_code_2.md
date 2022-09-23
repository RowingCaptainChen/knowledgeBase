# React基本原理2
> ## setState、Diff算法简析
## 1. setState过程

  1. Component内部实现，重点关注this.$updater = new Updater(this)，Updater类用来管理state。

```js
export default class Component{
    static isReactComponent = {}
    constructor(props, context){
        this.$updater = new Updater(this)
        this.$cache = { isMounted: false }
        this.props = props
        this.state = {}
        this.refs = {}
        this.context = context
    }
    forceUpdate(callback) {
        // 实际更新组件的函数
        let { $updater, $cache, props, state, context } = this
        if (!$cache.isMounted) {
            return
        }
        if ($updater.isPending) {
            $updater.addState(state)
            return;
        }
        let nextProps = $cache.props || props
        let nextState = $cache.state || state
        let nextContext = $cache.context || context
        let parentContext = $cache.parentContext
        let node = $cache.node
        let vnode = $cache.vnode
        // 缓存 
        $cache.props = $cache.state = $cache.context = null
        $updater.isPending = true
        if (this.componentWillUpdate) {
            this.componentWillUpdate(nextProps, nextState, nextContext)
        }
        this.state = nextState
        this.props = nextProps
        this.context = nextContext

        // 下面才是重点  对比vnode
        //renderComponent 生成新的Vnode
        let newVnode = renderComponent(this)
        //对比新旧node
        let newNode = compareTwoVnodes(vnode, newVnode, node, getChildContext(this, parentContext))
        if (newNode !== node) {
            newNode.cache = newNode.cache || {}
            syncCache(newNode.cache, node.cache, newNode)
        }
        $cache.vnode = newVnode
        $cache.node = newNode
        // 清除pending 执行didmount生命周期
        clearPending()
        if (this.componentDidUpdate) {
            this.componentDidUpdate(props, state, context)
        }
        if (callback) {
            callback.call(this)
        }
        $updater.isPending = false
        $updater.emitUpdate()
        // 更新
    }
    setState(nextState, callback) {
        // 添加异步队列  不是每次都更新
        this.$updater.addCallback(callback)
        this.$updater.addState(nextState)
    }
}
```

  2. 调用setState()刷新状态，调用Updater类的addCallback()和addState()方法。

  3. addState()将新状态加入队列，调用this.emitUpdate()判断做何种更新操作。

  4. emitUpdate()中根据，有无新的props或者更新队列是否挂起执行this.updateComponent()或者updateQueue.add(this)

  5. 当执行this.updateComponent()时，调用shouldUpdate()函数的参数会调用this.getState()，它会根据传入的state的数据类型采取不同的策略，但最后都要合并state。这也是多次setState()后会合并处理的原因。

  6. shouldUpdate()函数根据Component类的生命周期方法shouldComponentUpdate()的返回值判断是否更新state。并调用Component类中的forceUpdate()方法进行实际的state更新，需要注意的是Diff算法就在此类中。这也是可以直接调用forceUpdate()强制更新的原因。
  ```js
    class Updater{
    constructor(instance){
        this.instance = instance
        this.pendingStates = [] // 待处理状态数组
        this.pendingCallbacks = []
        this.isPending = false
        this.nextProps = this.nextContext = null
        this.clearCallbacks = this.clearCallbacks.bind(this)
    }
    emitUpdate(nextProps, nextContext) {
        this.nextProps = nextProps
        this.nextContext = nextContext
        //如果有新的props则立即更新
        //如果更新队列没有被挂起，则立即更新
        //否则都放入更新队列，等待更新
        nextProps || !updateQueue.isPending
        ? this.updateComponent()
        : updateQueue.add(this)
    }
    updateComponent() {
        let { instance, pendingStates, nextProps, nextContext } = this
        if (nextProps || pendingStates.length > 0) {
            nextProps = nextProps || instance.props
            nextContext = nextContext || instance.context
            this.nextProps = this.nextContext = null
            // getState 合并所有的state的数据，一次更新
            //注意传入的this.getState()
            shouldUpdate(instance, nextProps, this.getState(), nextContext, this.clearCallbacks)
        }
    }
    addState(nextState) {
        if (nextState) {
        //放入挂起的状态队列，稍后处理
            this.pendingStates.push(nextState)
            if (!this.isPending) {
            //如果不是挂起状态，则立即更新
                this.emitUpdate()
            }
        }
    }
    getState() {
        let { instance, pendingStates } = this
        let { state, props } = instance
        if (pendingStates.length) {
        //取出原有的state
            state = {...state}
            //遍历队列中的状态
            pendingStates.forEach(nextState => {
                //如果是数组类型，就取出第一个值替换nextState
                let isReplace = _.isArr(nextState)
                if (isReplace) {
                    nextState = nextState[0]
                }
                if (_.isFn(nextState)) {
                    nextState = nextState.call(instance, state, props)
                }
                //取代或者合并之前的state
                //这就是setState()是集合操作的原因
                if (isReplace) {
                    state = {...nextState}
                } else {
                    state = {...state, ...nextState}
                }
            })
            pendingStates.length = 0
        }
        return state
    }
    clearCallbacks() {
        let { pendingCallbacks, instance } = this
        if (pendingCallbacks.length > 0) {
            this.pendingCallbacks = []
            pendingCallbacks.forEach(callback => callback.call(instance))
        }
    }
    addCallback(callback) {
        //如果是函数，则放入回调的挂起队列
        if (_.isFn(callback)) {
            this.pendingCallbacks.push(callback)
        }
    }
}

function shouldUpdate(component, nextProps, nextState, nextContext, callback) {
    // 是否应该更新 判断shouldComponentUpdate生命周期
    let shouldComponentUpdate = true
    if (component.shouldComponentUpdate) {
    //根据component的返回值来判断是否更新
        shouldComponentUpdate = component.shouldComponentUpdate(nextProps, nextState, nextContext)
    }
    if (shouldComponentUpdate === false) {
        component.props = nextProps
        component.state = nextState
        component.context = nextContext || {}
        return
    }
    let cache = component.$cache
    cache.props = nextProps
    cache.state = nextState
    cache.context = nextContext || {}
    //强制更新
    component.forceUpdate(callback)
}
  ```

  7. 当执行updateQueue.add(this)时，将Updater()实例放入updater队列中处理，在render时候会调用batchUpdate()批处理，并执行e，f步骤。
  ```js
export let updateQueue = {
    updaters: [],
    isPending: false,
    add(updater) {
        this.updaters.push(updater)
    },
    batchUpdate() {
       //若为挂起状态则不处理
        if (this.isPending) {
            return
        }
        this.isPending = true
        let { updaters } = this
        let updater
        //取出每个updater实例进行更新
        while (updater = updaters.pop()) {
            updater.updateComponent()
        }
        //更新完成后关闭挂起状态
        this.isPending = false
    }
}
```

## 2.Diff算法
  ### 1. 比较节点：删除、替换、更新
1. forceUpdate()更新的时候调动compareTwoVnodes()主要进行以下操作：
2. newVnode为空时删除虚拟节点，从父级删除Dom节点。
3. newVnode的类型与原有vnode类型不同或者key不同，删除原有虚拟节点，新建虚拟节点，并替换父级原有节点。
4. newVnode的类型与原有vnode类型相同，key相同，就执行更新操作。

```js
export function compareTwoVnodes(vnode, newVnode, node, parentContext) {
    let newNode = node
    if (newVnode == null) {
        // 删除操作
        destroyVnode(vnode, node)
        node.parentNode.removeChild(node)
    } else if (vnode.type !== newVnode.type || vnode.key !== newVnode.key) {
        // 替换操作
        destroyVnode(vnode, node)
        newNode = initVnode(newVnode, parentContext)
        node.parentNode.replaceChild(newNode, node)
    } else if (vnode !== newVnode || parentContext) {
        // 更新操作
        newNode = updateVnode(vnode, newVnode, node, parentContext)
    }
    return newNode
}
```

 ### 2. 销毁节点
 ```js
// 根据组件类型销毁
export function destroyVnode(vnode, node) {
    let { vtype } = vnode
    if (vtype === VELEMENT) {  //element
        destroyVelem(vnode, node)
    } else if (vtype === VCOMPONENT) { // state component
        destroyVcomponent(vnode, node)
    } else if (vtype === VSTATELESS) { //stateless component
        destroyVstateless(vnode, node)
    }
}

function destroyVelem(velem, node) {
    let { vchildren, childNodes } = node
    if (vchildren) {
        for (let i = 0, len = vchildren.length; i < len; i++) {
            destroyVnode(vchildren[i], childNodes[i])
        }
    }
    //删除ref
    detachRef(velem.refs, velem.ref, node)
    node.eventStore = node.vchildren = null
}

function destroyVcomponent(vcomponent, node) {
    let uid = vcomponent.uid
    let component = node.cache[uid]
    let cache = component.$cache
    delete node.cache[uid]
    detachRef(vcomponent.refs, vcomponent.ref, component)
    component.setState = component.forceUpdate = _.noop
    if (component.componentWillUnmount) {
        component.componentWillUnmount()
    }
    //递归调用删除node
    destroyVnode(cache.vnode, node)
    delete component.setState
    cache.isMounted = false
    cache.node = cache.parentContext = cache.vnode = component.refs = component.context = null
}

function destroyVstateless(vstateless, node) {
    let uid = vstateless.uid
    let vnode = node.cache[uid]
    delete node.cache[uid]
     //递归调用删除node
    destroyVnode(vnode, node)
}

function detachRef(refs, refKey, refValue) {
    if (refKey == null) {
        return
    }
    if (_.isFn(refKey)) {
        refKey(null)
    } else if (refs && refs[refKey] === refValue) {
        delete refs[refKey]
    }
}
 ```

### 3. 初始化节点
```js
// 初始化vnode
export function initVnode(vnode, parentContext) {
    // 初始化 不同的vtype 执行不同的函数
    let { vtype } = vnode
    let node = null
    if (!vtype) { // init text
        node = document.createTextNode(vnode)
    } else if (vtype === VELEMENT) { // init element
        node = initVelem(vnode, parentContext)
    } else if (vtype === VCOMPONENT) { // init stateful component
        node = initVcomponent(vnode, parentContext)
    } else if (vtype === VSTATELESS) { // init stateless component
        node = initVstateless(vnode, parentContext)
    }
    return node
}

//***********************************************************************
// 1.初始化element
function initVelem(velem, parentContext) {
    let { type, props } = velem
    //实际上也是创建Dom
    let node = document.createElement(type)
     // 初始化childrend
    initVchildren(velem, node, parentContext)

    let isCustomComponent = type.indexOf('-') >= 0 || props.is != null
    _.setProps(node, props, isCustomComponent)

    if (velem.ref != null) {
        pendingRefs.push(velem)
        pendingRefs.push(node)
    }

    return node
}
// 遍历children 循环调用initVnode
function initVchildren(velem, node, parentContext) {
    let vchildren = node.vchildren = getFlattenChildren(velem)
    for (let i = 0, len = vchildren.length; i < len; i++) {
        node.appendChild(initVnode(vchildren[i], parentContext))
    }
}
//填充子元素
function getFlattenChildren(vnode) {
    let { children } = vnode.props
    let vchildren = []
    if (_.isArr(children)) {
        _.flatEach(children, collectChild, vchildren)
    } else {
        collectChild(children, vchildren)
    }
    return vchildren
}

function collectChild(child, children) {
    if (child != null && typeof child !== 'boolean') {
        children[children.length] = child
    }
}

//***********************************************************************
//2.初始化component
function initVcomponent(vcomponent, parentContext) {
    // 初始化
    let { type: Component, props, uid } = vcomponent
    let componentContext = getContextByTypes(parentContext, Component.contextTypes)
    // 初始化组件class
    let component = new Component(props, componentContext)
    // 获取updater和cache
    let { $updater: updater, $cache: cache } = component
    cache.parentContext = parentContext
    // 设置pending
    updater.isPending = true
    component.props = component.props || props
    component.context = component.context || componentContext
    // 生命周期
    if (component.componentWillMount) {
        component.componentWillMount()
        component.state = updater.getState()
    }
    //调用componet的render方法获取node
    let vnode = renderComponent(component)
    let node = initVnode(vnode, getChildContext(component, parentContext))
    node.cache = node.cache || {}
    node.cache[uid] = component
    cache.vnode = vnode
    cache.node = node
    cache.isMounted = true
    pendingComponents.push(component)

    if (vcomponent.ref != null) {
        pendingRefs.push(vcomponent)
        pendingRefs.push(component)
    }
    return node
}

export function renderComponent(component, parentContext) {
    refs = component.refs
    //调用componet的render方法获取node
    let vnode = component.render()
    if (!vnode || !vnode.vtype) {
        throw new Error(`@${component.constructor.name}#render:You may have returned undefined, an array or some other invalid object`)
    }
    refs = null
    return vnode
}

//***********************************************************************
//3.初始化function
function initVstateless(vstateless, parentContext) {
    let vnode = renderVstateless(vstateless, parentContext)
    let node = initVnode(vnode, parentContext)
    node.cache = node.cache || {}
    node.cache[vstateless.uid] = vnode
    return node
}

function renderVstateless(vstateless, parentContext) {
    // 函数组件
    let { type: factory, props } = vstateless
    let componentContext = getContextByTypes(parentContext, factory.contextTypes)
    let vnode = factory(props, componentContext)
    if (vnode && vnode.render) {
        vnode = vnode.render()
    }
    if (!vnode || !vnode.vtype) {
        throw new Error(`@${factory.name}#render:You may have returned undefined, an array or some other invalid object`)
    }
    return vnode
}

```

### 4. 更新节点
```js
function updateVnode(vnode, newVnode, node, parentContext) {
    let { vtype } = vnode
    //更新component
    if (vtype === VCOMPONENT) {
        return updateVcomponent(vnode, newVnode, node, parentContext)
    }
    //更新function
    if (vtype === VSTATELESS) {
        return updateVstateless(vnode, newVnode, node, parentContext)
    }
    // 忽略其他类型
    if (vtype !== VELEMENT) {
        return node
    }

    //处理html标签
    let oldHtml = vnode.props[HTML_KEY] && vnode.props[HTML_KEY].__html
    if (oldHtml != null) {
        updateVelem(vnode, newVnode, node, parentContext)
        initVchildren(newVnode, node, parentContext)
    } else {
        updateVChildren(vnode, newVnode, node, parentContext)
        updateVelem(vnode, newVnode, node, parentContext)
    }
    return node
}

//***********************************************************************
//1.更新component
function updateVcomponent(vcomponent, newVcomponent, node, parentContext) {
    // 更新组件
    let uid = vcomponent.uid
    let component = node.cache[uid]
    //很关键，获取updater，调用emitUpdate更新
    let { $updater: updater, $cache: cache } = component
    let { type: Component, props: nextProps } = newVcomponent
    let componentContext = getContextByTypes(parentContext, Component.contextTypes)
    delete node.cache[uid]
    node.cache[newVcomponent.uid] = component
    cache.parentContext = parentContext
    if (component.componentWillReceiveProps) {
        let needToggleIsPending = !updater.isPending
        if (needToggleIsPending) updater.isPending = true
        component.componentWillReceiveProps(nextProps, componentContext)
        if (needToggleIsPending) updater.isPending = false
    }
    //处理ref
    if (vcomponent.ref !== newVcomponent.ref) {
        detachRef(vcomponent.refs, vcomponent.ref, component)
        attachRef(newVcomponent.refs, newVcomponent.ref, component)
    }
    //还是调用updater里的emitUpdate更新
    updater.emitUpdate(nextProps, componentContext)
    
    return cache.node
}

//***********************************************************************
//2.更新function组件
function updateVstateless(vstateless, newVstateless, node, parentContext) {
    let uid = vstateless.uid
    let vnode = node.cache[uid]
    delete node.cache[uid]
    //还是老一套，新建后比较更新
    let newVnode = renderVstateless(newVstateless, parentContext)
    let newNode = compareTwoVnodes(vnode, newVnode, node, parentContext)
    newNode.cache = newNode.cache || {}
    newNode.cache[newVstateless.uid] = newVnode
    if (newNode !== node) {
        syncCache(newNode.cache, node.cache, newNode)
    }
    return newNode
}
```



# Redux底层实现
## 预备知识
1.高阶组件（high-order component)

2.context

3.纯函数（pure function)

## redux核心函数（No magic)
gitlab:http://gitlab.yeelight.com/chentongshuai/react-redux-demo

1.createStore()

```js
 function createStore(reducer) {
    let state = null;
    const listeners = [];
    //设置监听
    const subscribe = (listener)=>listeners.push(listener);
    //获取reducer中的state
    const getState = ()=> state;
    //执行刷新操作
    const dispatch = (action) => {
        state = reducer(state, action);
        //执行监听
        listeners.forEach((listener)=>listener());
    }
    //初始化reducer中的state
    dispatch({}); 
    return {getState, dispatch, subscribe}
}
```

2.connect()

```js
//connect作用是在高阶组件做统一处理，将store 和 context结合
//其目的是把store中的数据取出来，通过props传给WrappedComponent

const connect = (mapStateToProps, mapDispatchToProps) => (WrappedComponent) => {
    class Connect extends Component {
        //声明和验证需要获取的状态的类型
        static contextTypes = {
            store: PropTypes.object
        }
        constructor() {
            super();
            this.state = {allProps:{}}
        }
        componentWillMount() {
            const {store} = this.context;
            //初始化渲染
            this._updateProps();
            //设置监听
            store.subscribe(()=>this._updateProps());
        }
        _updateProps() {
            const {store} = this.context;
            let stateProps = mapStateToProps ? mapStateToProps(store.getState(), this.props) : {};
            let dispatchProps = mapDispatchToProps ? mapDispatchToProps(store.dispatch, this.props) : {};
            this.setState({
                //整合三者，三者都会以props的形式传给组件
                allProps: { 
                    ...stateProps, //想要获取的state
                    ...dispatchProps, //想要触发的dispatch
                    ...this.props //普通的this.props
                }
            })
        }
        render() {            
            return <WrappedComponent {...this.state.allProps}/>
        }
    }
    return Connect;
}
```

3.mapStateToProps()

```js
//目的是告知connect函数，组件需要取出哪些state
const mapStateToProps = (state) => {
  return {
   A: state.A,
   B: state.B,
    ...
  }
}
```

4.mapDispatchToProps()

```js
//目的是告知connect函数，组件需要如何触发dispatch
const mapDispatchToProps = (dispatch) => {
  return {
   A: () => {
      dispatch({ typeA: 'A', A: ''})
    },
   B: (param) => {
      dispatch({ typeA: 'B', B: param})
    }
    ...
  }
}
```

5.Provider

```js
//其实就是一个容器组件，会把嵌套的内容原封不动作为自己的子组件渲染出来。
//还会把外界传给它的props.store放到context，使组件connect的时候都可以获取到。
export class Provider extends Component {
  static propTypes = {
    store: PropTypes.object,
    children: PropTypes.any
  }
    //验证getChildContext的返回对象，跟propsType作用类似
  static childContextTypes = {
    store: PropTypes.object
  }
  getChildContext () {
    return {
      store: this.props.store
    }
  }
  render () {
    return (
      <View>{this.props.children}</View>
    )
  }
}

```


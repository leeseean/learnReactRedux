## 前言

Provider是React的一个组件，利用React的context概念，将数据传递给各个组件。Provider模块的功能很简单，从最外部封装了整个应用，并向connect模块传递store, 接收到状态改变时更新store。

## 源码

```
// 继承自React.Component说明其是一个React组件
class Provider extends Component {
  constructor(props) {
    super(props)

    const { store } = props // 获取<Provider store={store}>的属性
    // 获取的store为初始store状态
    this.state = {
      storeState: store.getState(),
      store
    }
  }

  componentDidMount() {
    this._isMounted = true
    this.subscribe() // 注册监听
  }

  componentWillUnmount() {
    if (this.unsubscribe) this.unsubscribe() // 取消监听

    this._isMounted = false
  }

  componentDidUpdate(prevProps) {// 组件更新的时候重新注册监听
    if (this.props.store !== prevProps.store) {
      if (this.unsubscribe) this.unsubscribe()

      this.subscribe()
    }
  }
  // 比较关键的监听方法
  subscribe() {
    const { store } = this.props
    // redux中的store.subscribe方法，如果状态改变则更新状态
    this.unsubscribe = store.subscribe(() => {
      const newStoreState = store.getState()

      if (!this._isMounted) {
        return
      }

      this.setState(providerState => {
        // If the value is the same, skip the unnecessary state update.
        if (providerState.storeState === newStoreState) {
          return null
        }

        return { storeState: newStoreState }
      })
    })

    // actions可能已经在render和mount之间dispatch了，则执行下面代码
    const postMountStoreState = store.getState()
    if (postMountStoreState !== this.state.storeState) {
      this.setState({ storeState: postMountStoreState })
    }
  }

  render() {
    // 获取Context
    const Context = this.props.context || ReactReduxContext
    // 通过Context将状态值传入
    return (
      <Context.Provider value={this.state}>
        {this.props.children}
      </Context.Provider>
    )
  }
}
```
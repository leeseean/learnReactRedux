## 前言

connect是react-redux的核心，他将react与redux两个互不相干的库连接起来。connect执行后返回一个函数wrapWithConnect，wrapWithConnect接收一个组件作为参数，返回一个带有新状态props的组件。具体流程如下：
<img src="../connect-flow.svg">

## connect.js

#### createConnect

```
// 生成connect方法的函数
export function createConnect({
  connectHOC = connectAdvanced,
  mapStateToPropsFactories = defaultMapStateToPropsFactories,
  mapDispatchToPropsFactories = defaultMapDispatchToPropsFactories,
  mergePropsFactories = defaultMergePropsFactories,
  selectorFactory = defaultSelectorFactory
} = {}) {
  // connect方法   接受的四个参数
  return function connect(
    mapStateToProps,
    mapDispatchToProps,
    mergeProps,
    {
      pure = true, // 是否浅比较
      areStatesEqual = strictEqual,
      areOwnPropsEqual = shallowEqual,
      areStatePropsEqual = shallowEqual,
      areMergedPropsEqual = shallowEqual,
      ...extraOptions
    } = {}
  ) {
    // 一系列的方法执行，对三个参数的类型做了容错处理
    // 分别初始化了各自的参数mapStateToProps,mapDispatchToProps,mergeProps，注入了一些内部的默认参数和方法
    // 他们大致是这样的function： 
    // (dispatch, options) => initProxySelector() => mapToPropsProxy() => props  
    const initMapStateToProps = match(
      mapStateToProps,
      mapStateToPropsFactories,
      'mapStateToProps'
    )
    const initMapDispatchToProps = match(
      mapDispatchToProps,
      mapDispatchToPropsFactories,
      'mapDispatchToProps'
    )
    const initMergeProps = match(mergeProps, mergePropsFactories, 'mergeProps')
    // 返回值由执行connectAdvanced获取,并传入初始化的initMapStateToProps等参数和pure等配置项
    return connectHOC(selectorFactory, {
      // used in error messages
      methodName: 'connect',

      // used to compute Connect's displayName from the wrapped component's displayName.
      getDisplayName: name => `Connect(${name})`,

      // if mapStateToProps is falsy, the Connect component doesn't subscribe to store state changes
      shouldHandleStateChanges: Boolean(mapStateToProps),

      // passed through to selectorFactory
      initMapStateToProps,
      initMapDispatchToProps,
      initMergeProps,
      pure,
      areStatesEqual,
      areOwnPropsEqual,
      areStatePropsEqual,
      areMergedPropsEqual,

      // any extra options args can override defaults of connect or connectAdvanced
      ...extraOptions
    })
  }
}
```

## connectAdvanced.js

#### connectAdvanced

connectAdvanced返回一个react高阶组件, 根据pure,forwardRef配置决定是否采用PureComponent和ref的转移， 通过selectDerivedProps放法生成最终的props，传递给最终返回的react组件

```
export default function connectAdvanced(
  // 属性生成工厂
  selectorFactory,
  // 配置项:
  {
    getDisplayName = name => `ConnectAdvanced(${name})`,

    methodName = 'connectAdvanced',

    renderCountProp = undefined,

    // 是否订阅到状态变化
    shouldHandleStateChanges = true,

    // REMOVED: the key of props/context to get the store
    storeKey = 'store',

    // withRef用来给包装在里面的组件一个ref，可以通过getWrappedInstance方法来获取这个ref，默认为false
    withRef = false,

    // 给包裹的组件暴露一个ref，作用就是能获取到我们传入的Component的DOM，默认为false
    forwardRef = false,

    // the context consumer to use
    context = ReactReduxContext,

    // 其他选项
    ...connectOptions
  } = {}
) {
  invariant(
    renderCountProp === undefined,
    `renderCountProp is removed. render counting is built into the latest React dev tools profiling extension`
  )

  invariant(
    !withRef,
    'withRef is removed. To access the wrapped instance, use a ref on the connected component'
  )

  const customStoreWarningMessage =
    'To use a custom Redux store for specific components,  create a custom React context with ' +
    "React.createContext(), and pass the context object to React-Redux's Provider and specific components" +
    ' like:  <Provider context={MyContext}><ConnectedComponent context={MyContext} /></Provider>. ' +
    'You may also pass a {context : MyContext} option to connect'

  invariant(
    storeKey === 'store',
    'storeKey has been removed and does not do anything. ' +
      customStoreWarningMessage
  )

  const Context = context
  // 传入组件，返回一个组件
  return function wrapWithConnect(WrappedComponent) {
    if (process.env.NODE_ENV !== 'production') {
      invariant(
        isValidElementType(WrappedComponent),
        `You must pass a component to the function returned by ` +
          `${methodName}. Instead received ${JSON.stringify(WrappedComponent)}`
      )
    }

    const wrappedComponentName =
      WrappedComponent.displayName || WrappedComponent.name || 'Component'

    const displayName = getDisplayName(wrappedComponentName)

    const selectorFactoryOptions = {
      ...connectOptions,
      getDisplayName,
      methodName,
      renderCountProp,
      shouldHandleStateChanges,
      storeKey,
      displayName,
      wrappedComponentName,
      WrappedComponent
    }

    const { pure } = connectOptions

    let OuterBaseComponent = Component
    let FinalWrappedComponent = WrappedComponent

    if (pure) {
      OuterBaseComponent = PureComponent
    }
    /*
    makeDerivedPropsSelector的执行，先判断了当前传入的props(组件的props)和state(redux传入的state)
    跟以前的是否全等，如果全等就不需要更新了；
    如果不等，则调用了高阶函数selectFactory，并且获得最终数据，最后再判断最终数据和之前的最终数据是否全等。
    */
    function makeDerivedPropsSelector() {
      // 闭包储存上一次的执行结果
      let lastProps
      let lastState
      let lastDerivedProps
      let lastStore
      let sourceSelector

      return function selectDerivedProps(state, props, store) {
        // props和state都和之前相等 直接返回上一次的结果  
        if (pure && lastProps === props && lastState === state) {
          return lastDerivedProps
        }
        // 当前store和lastStore不等，更新lastStore
        if (store !== lastStore) {
          lastStore = store
          // 更新 sourceSelector
          sourceSelector = selectorFactory(
            store.dispatch,
            selectorFactoryOptions
          )
        }
        // 更新数据
        lastProps = props
        lastState = state
        // 返回的就是最终的包含所有相应的 state 和 props 的结果
        const nextProps = sourceSelector(state, props)
        // 最终的比较
        if (lastDerivedProps === nextProps) {
          return lastDerivedProps
        }
       
        lastDerivedProps = nextProps
        return lastDerivedProps
      }
    }
    // 这里是最终渲染组件的地方，因为需要判断一下刚才最终给出的数据是否需要去更新组件。
    function makeChildElementSelector() {
      // 闭包储存上一次的执行结果  
      let lastChildProps, lastForwardRef, lastChildElement

      return function selectChildElement(childProps, forwardRef) {
        // 判断新旧props， ref是否相同  
        if (childProps !== lastChildProps || forwardRef !== lastForwardRef) {
          lastChildProps = childProps
          lastForwardRef = forwardRef
          lastChildElement = (
            <FinalWrappedComponent {...childProps} ref={forwardRef} />
          )
        }

        return lastChildElement
      }
    }

    class Connect extends OuterBaseComponent {
      constructor(props) {
        super(props)
        invariant(
          forwardRef ? !props.wrapperProps[storeKey] : !props[storeKey],
          'Passing redux store in props has been removed and does not do anything. ' +
            customStoreWarningMessage
        )
        // 添加selectDerivedProps和selectChildElement方法
        // selectDerivedProps为function是makeDerivedPropsSelector的返回值
        this.selectDerivedProps = makeDerivedPropsSelector()
        this.selectChildElement = makeChildElementSelector()
        this.renderWrappedComponent = this.renderWrappedComponent.bind(this)
      }

      renderWrappedComponent(value) {
        invariant(
          value,
          `Could not find "store" in the context of ` +
            `"${displayName}". Either wrap the root component in a <Provider>, ` +
            `or pass a custom React context provider to <Provider> and the corresponding ` +
            `React context consumer to ${displayName} in connect options.`
        )
        const { storeState, store } = value
        // 传入自定义组件的props
        let wrapperProps = this.props
        let forwardedRef
        // forwardRef为真时, Connect组件提供了forwardedRef = {ref}
        if (forwardRef) {
          // wrapperProps为props中的wrapperProps
          wrapperProps = this.props.wrapperProps
          // forwardedRef赋值为props的forwardedRef, 传递的是ref
          // 用于传递给子组件WrappedComponent既let FinalWrappedComponent = WrappedComponent中的FinalWrappedComponent
          forwardedRef = this.props.forwardedRef
        }
        // 导出props
        let derivedProps = this.selectDerivedProps(
          storeState,
          wrapperProps,
          store
        )
        // 返回最终的组件,传入最终的props和ref
        return this.selectChildElement(derivedProps, forwardedRef)
      }

      render() {
        // 默认情况下公用的ReactReduxContext  
        const ContextToUse = this.props.context || Context

        return (
          // <Privoder />的消费者  
          <ContextToUse.Consumer>
            {this.renderWrappedComponent}
          </ContextToUse.Consumer>
        )
      }
    }

    Connect.WrappedComponent = WrappedComponent
    Connect.displayName = displayName
    // 使用react.forwardRef为connect生成的组件的父组件提供孙子(传递给connect的组件)组件
    if (forwardRef) {
      const forwarded = React.forwardRef(function forwardConnectRef(
        props,
        ref
      ) {
        return <Connect wrapperProps={props} forwardedRef={ref} />
      })

      forwarded.displayName = displayName
      forwarded.WrappedComponent = WrappedComponent
      return hoistStatics(forwarded, WrappedComponent)
    }
    // 将组件 WrappedComponent 的所有非 React 
静态方法传递到 Connect 内部。
    return hoistStatics(Connect, WrappedComponent)
  }
}

```
## selectorFactory.js

#### impureFinalPropsSelectorFactory

pure为false时强行合并状态和属性，不管状态和属性有没有变化，不做优化

```
export function impureFinalPropsSelectorFactory(
  mapStateToProps,
  mapDispatchToProps,
  mergeProps,
  dispatch
) {
  return function impureFinalPropsSelector(state, ownProps) {
    return mergeProps(
      mapStateToProps(state, ownProps),
      mapDispatchToProps(dispatch, ownProps),
      ownProps
    )
  }
}
```

#### pureFinalPropsSelectorFactory

pure为true时，定义了几个合并状态和属性的方法，合并状态和属性

```
export function pureFinalPropsSelectorFactory(
  mapStateToProps,
  mapDispatchToProps,
  mergeProps,
  dispatch,
  { areStatesEqual, areOwnPropsEqual, areStatePropsEqual }
) {
  let hasRunAtLeastOnce = false
  let state
  let ownProps
  let stateProps
  let dispatchProps
  let mergedProps

  function handleFirstCall(firstState, firstOwnProps) {
    state = firstState
    ownProps = firstOwnProps
    stateProps = mapStateToProps(state, ownProps)
    dispatchProps = mapDispatchToProps(dispatch, ownProps)
    mergedProps = mergeProps(stateProps, dispatchProps, ownProps)
    hasRunAtLeastOnce = true
    return mergedProps
  }

  function handleNewPropsAndNewState() {
    stateProps = mapStateToProps(state, ownProps)

    if (mapDispatchToProps.dependsOnOwnProps)
      dispatchProps = mapDispatchToProps(dispatch, ownProps)

    mergedProps = mergeProps(stateProps, dispatchProps, ownProps)
    return mergedProps
  }

  function handleNewProps() {
    if (mapStateToProps.dependsOnOwnProps)
      stateProps = mapStateToProps(state, ownProps)

    if (mapDispatchToProps.dependsOnOwnProps)
      dispatchProps = mapDispatchToProps(dispatch, ownProps)

    mergedProps = mergeProps(stateProps, dispatchProps, ownProps)
    return mergedProps
  }

  function handleNewState() {
    const nextStateProps = mapStateToProps(state, ownProps)
    const statePropsChanged = !areStatePropsEqual(nextStateProps, stateProps)
    stateProps = nextStateProps

    if (statePropsChanged)
      mergedProps = mergeProps(stateProps, dispatchProps, ownProps)

    return mergedProps
  }

  function handleSubsequentCalls(nextState, nextOwnProps) {
    const propsChanged = !areOwnPropsEqual(nextOwnProps, ownProps)
    const stateChanged = !areStatesEqual(nextState, state)
    state = nextState
    ownProps = nextOwnProps

    if (propsChanged && stateChanged) return handleNewPropsAndNewState()
    if (propsChanged) return handleNewProps()
    if (stateChanged) return handleNewState()
    return mergedProps
  }

  return function pureFinalPropsSelector(nextState, nextOwnProps) {
    return hasRunAtLeastOnce
      ? handleSubsequentCalls(nextState, nextOwnProps)
      : handleFirstCall(nextState, nextOwnProps)
  }
}
```

#### finalPropsSelectorFactory

finalPropsSelectorFactory是根据option.pure属性来决定是调用impureFinalPropsSelectorFactory还是pureFinalPropsSelectorFactory方法

```
// merge props和merge state
export default function finalPropsSelectorFactory(
  dispatch,
  { initMapStateToProps, initMapDispatchToProps, initMergeProps, ...options }
) {
  const mapStateToProps = initMapStateToProps(dispatch, options)
  const mapDispatchToProps = initMapDispatchToProps(dispatch, options)
  const mergeProps = initMergeProps(dispatch, options)
  const selectorFactory = options.pure
    ? pureFinalPropsSelectorFactory
    : impureFinalPropsSelectorFactory

  return selectorFactory(
    mapStateToProps,
    mapDispatchToProps,
    mergeProps,
    dispatch,
    options
  )
}
```
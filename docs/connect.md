## 前言

connect是react-redux的核心，他将react与redux两个互不相干的库连接起来。connect执行后返回一个函数wrapWithConnect，wrapWithConnect接收一个组件作为参数，返回一个带有新状态props的组件。具体流程如下：
```mermaid
flowchat
st=>start: createConnect({
  connectHOC = connectAdvanced,
  mapStateToPropsFactories = defaultMapStateToPropsFactories,
  mapDispatchToPropsFactories = defaultMapDispatchToPropsFactories,
  mergePropsFactories = defaultMergePropsFactories,
  selectorFactory = defaultSelectorFactory
} = {})
op=>operation: connect(
    mapStateToProps,
    mapDispatchToProps,
    mergeProps,
    {
      pure = true,
      areStatesEqual = strictEqual,
      areOwnPropsEqual = shallowEqual,
      areStatePropsEqual = shallowEqual,
      areMergedPropsEqual = shallowEqual,
      ...extraOptions
    } = {}
  )
op1=>operation: connectHOC(selectorFactory, {
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
op2=>operation:  wrapWithConnect(WrappedComponent)
op3=>operation:  hoistStatics(Connect, WrappedComponent)
op4=>operation:  hoistStatics(Connect, WrappedComponent)
e=>targetComponent

st->op->op1->op2->op3->op4->e
```




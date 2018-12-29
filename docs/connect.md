## 前言

connect是react-redux的核心，他将react与redux两个互不相干的库连接起来。connect执行后返回一个函数wrapWithConnect，wrapWithConnect接收一个组件作为参数，返回一个带有新状态props的组件。具体流程如下：
<img src="../connect-flow.svg">
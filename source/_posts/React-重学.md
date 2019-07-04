title: React 原则与生命周期
author: 江小渔
tags:
  - Rect
categories: []
date: 2019-07-03 20:43:00
---
# React原则
代码相关: https://codesandbox.io/s/6n20nrzlxz

#### 何时创建组件:单一职责原则
1. 每个组件只做一件事情 
2. 如果组件变得复杂那应该拆分成小组件 

#### 数据状态管理 DRY原则
1. 能计算得到的状态就要单独存储 
2. 组件尽量无状态f所需数据通过 props 获取

# React生命周期

![upload successful](/images/pasted-1.png)
图片来源:http://projects.wojtekmaj.pl/react-lifecycle-methods-diagram/

#### constructor
1. 用初始化内部状态(很少使用)
2. 唯一可以直接修改 state 的地方

#### getDerivedStateFromProps(Derived-导出)
1. 当 state 需要从 props 初始化时使用 
2. 尽量不要使用(维护两者状态一致性会增加复杂度)
3. 每次 render 都会调用 
4. 典型场景: 表单控件获取默认值 


#### componentDidMount
1. UI 渲染完成后调用 
2. 只执行一次 
3. 典型场景: 获取外部资源

#### componentWillMount
1. 组件移除时被调用 
2. 典型场景: 资源释放

#### getSnapshotBeforeUpdate
1. 在页面 render 之前调用,state 已更新 
2. 典型场景: 获取 render 之前的 DOM 状态
3. 它的返回值将作为 componentDidUpdate() 的第三个参数 “snapshot” 参数传递

#### componentDidUpdate
1. 在页面 UI 更新时调用
2. 典型场景:页面需要根据 props 变化重新获取数据

#### shouldComponentUpdate
1. 决定 Virtual DOM 是否要重绘 
2. 一般可以由 PureComponent 自动实现 
3. 典型场景: 性能优化

参考: https://react.docschina.org/docs/react-component.html
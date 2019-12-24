*本文仅讨论observer(component), @observer场景，版本为mobx-react v5*

### 问题
- 为什么mobx的observable数据有变化，observer组件就可以渲染，怎么做到的
- observer具体做了什么

#### 处理对象
- 由React.forwardRef创建的组件
- 函数式组件(FC)
- class组件
---
##### React.forwardRef
对于React.forwardRef创建的组件，则重新创建React.forwardRef组件，拦截渲染(render), 直接返回Observer组件
```javascript
return forwardRef(function ObserverForwardRef() {
            return <Observer>{() => baseRender.apply(undefined, arguments)}</Observer>
        })
```
<Observer>{...}</Observer>这一段实际上就是observer(FC)

##### 函数式组件(FC)
将函数式组件包裹一层class组件，复制静态方法/属性，再进入处理流程
##### class组件
直接进入处理流程

---
#### observer处理流程
1. 混入(mixin)生命周期
2. 改造react的state, props为响应式
3. 重写render方法

##### 混入(mixin)生命周期
- componentDidMount。开发工具支持
- componentDidUpdate。开发工具支持
- shouldComponentUpdate。浅比较，类似pureComponent
- componentWillUnmount。清除react组件中$mobx属性挂载的各种observable
componentWillUnmount，是为了防止内存泄漏。$mobx属性挂载在react组件上，统一管理通过装饰器使用的各种observable数据。如
```javascript
@observable count = 1
@computed get total() { return this.count * 3 }
```
调用顺序 原方法 -> mixins

##### 改造react的state, props为响应式
响应式本质：Object.defineProperty. 在get, set方法中，配合mobx的observable.reportObserved, reportChanged,通知mobx做出反应


---
回到问题1:
#### 如何驱动渲染
- 收集依赖(get -> reportObserved), 跟踪依赖变更(set -> reportChanged), 调用（如果变动的是state或props则不调用）forceUpdate触发render生命周期，从而执行渲染.

问题2：见observer处理流程

> 拓展：实现componentDidCatch

#### inject和<Provider/>的运行机制
inject的用法
- @inject('color')
- @inject((stores) => ({user: stores.user})

Provider的实现，本质上是利用react的context机制，及生产消费者机制。具体如下
- 生产者：把<Provider/>组件的props统一挂载到context上
- 消费者：inject。把需要注入的属性按需从Provider的props取出来，注入到目标组件上。同时处理静态方法/属性，ref等细节
#### 如何用
* @computed （工作上用的最多）
* @computed({ options })
* computed(expr, options?)
前两种，通过调用computedDecorator，并返回装饰器
最后一种，返回 ComputedValue

⚠️ComputedValue即是observable，也是derivation

#### 常规使用，以及在autorun中的computed
* 常规使用：即直接获取observable.get()进行运算

#### 在autorun中的computed
* 在autorun中：autorun本质上是一个derivation，通过收集observable依赖，当observable变化时，通知autorun。  Autorun <- -> observable
* 当autorun遇上computed，对于autorun来说，computedValue是作为observable的角色；对于computed其中的observable，computed是作为derivation的角色。
* autorun需要知道computed是否有变化，而computed是由内部observable是否有变化，因此对应关系是 autorun <- -> computedValue <- -> observable (in computedValue)

#### computed的性能优化
* 在computed中，分两种运算。一种是直接获取observable的值，进行运算。第二种是跟踪observable的变化，同时进行运算。后者的性能消耗更大些。
* computed做的性能优化，就是减少依赖跟踪，即第二种，以及shouldComputed的状态判断。
* 因此改变某个observable时，如果不是computed收集的，就不用通知computed。computed通过shouldComputed知道，“我不用重新跟踪依赖并计算噢”，直接返回旧值

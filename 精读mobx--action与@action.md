*mobx版本为4.10.0*

### 问题
- action运行机制是什么？
- @action fn() {}, @action(fn) foo() {} 怎么实现的（如何实现一个装饰器）

action做的事情很简单，在传入的fn函数包裹一层，把实际需要调用的函数放进一个事务(batch)里面。
实际调用栈如 startAction -> fn -> endAction
#### startAction 
- 先把全局derivation重置null(可能是action不需要跟踪依赖，所以减少跟踪依赖的性能损耗)
- 调用startBatch，开启一个事务（mobx通过全局inBatch来减少中间状态转换，到最后inBatch为0时，才执行副作用，达到批量处理的效果）
- 调用allowStateChangesStart(true) 改变全局变量globalState.allowStateChanges为true，用在observable的值有改变的情况。
allowStateChanges为false的情况下改变observable时，开发环境下会报错

#### 执行fn
- 此时执行fn，一般fn里面会改变observable的值，此时一样进入startBatch，endBatch中。observable通知对应的derivation如computedValue重新计算
- 如果执行有错误，则改变全局变量globalState.suppressReactionErrors

#### endAction
这里做的主要是把startAction改变的值还原回去，inbatch到了最外层，开始真正执行副作用
- 调用如：
allowStateChangesEnd
endBatch
untrackedEnd

#### 小结
- mobx推荐用action的原因，除了可读性高一些，同时也有一定的性能提升，并且捕获了异常。
- mobx通过减少依赖跟踪，批量处理副作用来优化性能。


### 如何写一个类方法装饰器

参考链接
[es6中的装饰器](http://es6.ruanyifeng.com/#docs/decorator)
[typescript中的装饰器](https://www.tslang.cn/docs/handbook/decorators.html)
#### 方法装饰器的基本格式
```javascript
function methodDescriptor(target, prop, descriptor) {
    // @method fn() {...}       -> descriptor.value
    // @method fn = () => {...} -> descriptor.initializer
    return descriptor
}
```
descriptor.value即为原方法fn
descriptor.initializer是babel针对箭头方法的处理，实际为
```javascript
function initializer() {
    return fn;
}
```

因此，可有一个通用格式：
```typescript
function createMethodDecorator(enhancer) {
    return function (target, prop, descriptor) {
        // @method fn() {...}
        if (descriptor) {
            if (descriptor.value) {
                return {
                    value: enhancer(descriptor.value),
                    enumerable: false,
                    configurable: true,
                    writable: true,
                }
            }
            // @method fn = () => {...} babel only
            return {
                enumerable: false,
                configurable: true,
                writable: true,
                initializer() {
                    return enhancer(descriptor.initializer.call(this))
                }
            }
        }
    }
}
```

这样我们可以创建自己的~~低配版~~装饰器

AOP
```typescript
const before = (beforeFn: Function) => createMethodDecorator((fn) => {
  return function() {
    beforeFn(...arguments);
    const result = fn.call(this, ...arguments);
    console.log("result: ", result);
  }
});

const after = (afterFn: Function) => createMethodDecorator((fn) => {
  return function() {
    const result = fn.call(this, ...arguments);
    console.log("result: ", result);
    afterFn({args: arguments, result});
  }
});
```

debounce & throttle
```typescript
function debounce$(ms) {
  return createMethodDecorator(function(fn){ return debounce(fn, ms) });
}

function throttle$(ms) {
  return createMethodDecorator(function(fn){ return throttle(fn, ms) });
}

function debounce(fn, ms = 0) {
  let timerId = null;
  let result;
  return function() {
    clearTimeout(timerId);
    timerId = setTimeout(() => {
      result = fn.call(this, ...arguments);
    }, ms);
    return result;
  }
}

function throttle(fn, ms) {
  let timerId = null;
  let result;
  return function() {
    if (timerId) return result;
    timerId = setTimeout(() => {
      timerId = null;
      result = fn.call(this, ...arguments);
    }, ms);
    return result;
  }
}
```

如果有用typescript的朋友，会发现这样写并不凑效
```typescript
    class A {
        count = 0;

        @debounce$(10)
        add = () => this.count++;
    }
```
原因就在于 add = () => {...}实际上是class A上的一个属性，@debounce$(10)被识别成属性装饰器。

#### 属性装饰器 @action fn = () => {}
``` typescript
function createPropertyDecorator(enhancer) {
    function addHiddenProp(object: any, propName: any, value: any) {
        Object.defineProperty(object, propName, {
            enumerable: false,
            writable: true,
            configurable: true,
            value
        })
    }
    return function define(target, prop, descriptor) {
        Object.defineProperty(target, prop, {
            ...descriptor,
            configurable: true,
            enumerable: false,
            get() {
                return undefined
            },
            set(value) {
                addHiddenProp(this, prop, enhancer(value))
            }
        })
    }
}
```

装饰器工具方法完整版为
```typescript
function createMethodDecorator(enhancer) {
    return function (target, prop, descriptor) {
        // @method fn() {...}
        if (descriptor) {
            if (descriptor.value) {
                return {
                    value: enhancer(descriptor.value),
                    enumerable: false,
                    configurable: true,
                    writable: true,
                }
            }
            // @method fn = () => {...} babel only
            return {
                enumerable: false,
                configurable: true,
                writable: true,
                initializer() {
                    return enhancer(descriptor.initializer.call(this))
                }
            }
        }
        // @method fn = () => {...} typescript
        return createPropertyDecorator(enhancer).apply(this, arguments);
    }
}
```

支持 @method fn = () => {...} 或 @method fn() {} 形式
[测试](https://github.com/rosong1/Exercise/blob/master/test/methodDecorator.test.ts)
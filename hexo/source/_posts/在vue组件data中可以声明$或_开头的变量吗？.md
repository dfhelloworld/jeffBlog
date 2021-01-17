# 在vue组件中是否可以声明$或_开头的变量
不吊胃口，首先抛出答案，是可以的，但是行为与通常声明的变量行为很不一致，且听我慢慢道来。

该问题源于前段时间在项目中，某个同事为了在子组件中接受父组件传递的名为item参数，在子组件中添加了一个名为`$item`的变量。
碰巧我也在该组件中添加了新功能，在调试过程中诡异的事情就发生了，`$item`的值可以被修改，但是却触发不了watch和computed，敏锐的我判断难道这跟前面的`$`符号有什么关系，于是乎我把变量改成了itemCopy后果然一切恢复了正常。

一番百度谷歌后，在官网上得到了这样一个结果：
>名字以 `_`或`$`开始的属性不会被 Vue 实例代理，因为它们可能与 Vue 的内置属性与 API 方法冲突。用 vm.$data._property 访问它们。

我对这段话的大致理解就是该属性没有通过vue实现数据的双向绑定，并且在正常访问时都无法访问到，必须通过``vm.$data._property才能够访问。 

我赶忙把`$item`替换成了`_item`，发现果然监听不到数据的变化。

对于这样一个高频错误查到的内容仅仅是上面一小段解答让我觉得一知半解，又仔细搜索了官方文档后依然没有找到相关的解释。本着知其然知其所以然的态度，只能自己去刨根问底看看源码中到底对于这两个特殊符号做了哪些特殊处理。

源码搬运工

对于保留的前缀的告警提示
```javaScript
  const warnReservedPrefix = (target, key) => {
    warn(
      `Property "${key}" must be accessed with "$data.${key}" because ` +
      'properties starting with "$" or "_" are not proxied in the Vue instance to ' +
      'prevent conflicts with Vue internals. ' +
      'See: https://vuejs.org/v2/api/#data',
      target
    )
  }
```

对于_前缀开头的属性处理
```javascript
  const hasHandler = {
    has (target, key) {
      const has = key in target
      const isAllowed = allowedGlobals(key) ||
        (typeof key === 'string' && key.charAt(0) === '_' && !(key in target.$data))
      if (!has && !isAllowed) {
        if (key in target.$data) warnReservedPrefix(target, key)
        else warnNonPresent(target, key)
      }
      return has || !isAllowed
    }
  }
```
主要还是防止覆盖vue的内部定义变量，从而对于`$`和`_`开头的变量不进行数据绑定的操作。虽然在文档很显眼的位置说明了，但在写代码时如果不理解其含义还是很容易犯这个错误的。
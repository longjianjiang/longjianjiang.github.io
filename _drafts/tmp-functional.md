# 函数式编程

## 纯函数

函数不能与外部有任何的耦合关系，既不能依赖外部的状态（相同输入一定相同的输出），也不能对外界产生影响（无副作用）.

## 高阶函数

函数就是值，也就是使用参数，返回值的地方也可以使用函数。

所谓高阶函数就是函数的参数和返回值至少有一个是函数类型。基于此，使复用力度降低到了函数级别，一般是类级别。

## Currying 柯里化

currying就是将 多参数函数 转化成 一系列单参数函数的过程。

{% highlight javascript%}
function add(x, y) {
  return x + y;
}

var addCurrying = function(x) {
  return function(y) {
    return x + y;
  }
}
{% endhighlight %}

`add(1, 2)` 等价于 `addCurrying(1)(2)`。

一个优点是当某个函数需要多次调用，且部分参数相同时，通过柯里化可以减少重复参数样板代码。

{% highlight javascript%}
add(10, 1);
add(10, 2);
add(10, 3);

var addTen = addCurrying(10);
addTen(1);
addTen(2);
addTen(3);
{% endhighlight %}

# References

[http://zxfcumtcs.github.io/2019/11/17/functional/](http://zxfcumtcs.github.io/2019/11/17/functional/)

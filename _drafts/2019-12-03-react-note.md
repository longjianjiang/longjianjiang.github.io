
# Virtual Dom

dom其实就是浏览器上的一个个元素。所谓虚拟dom其实就是react中的component，一个js对象，这个js对象描述了dom的信息。在挂载阶段，react将这些component进行创建渲染。

相应的在更新阶段，首先react会使用diff算法来进行component之间的diff计算，得出哪些真实的dom需要做更新操作，然后去更新dom。

# Component

React 中最简单的组件就是函数的形式进行创建，此时的组件是无状态的，仅仅负责展示内容。当需要处理状态的时候就需要使用class的形式继承`React.Component`。

使用了react-redux后，对于组件的区分有两种，一种是component，一种是container。

component组件算是纯UI，不使用state，不使用redux的api。

container则是容器，负责处理UI展示的逻辑，不负责UI的展示，使用redux的api。

如果一个组件既有UI又要逻辑需要处理，需要进行拆分，将UI展示部分拆成component，将component作为UI放到container中，container内部处理逻辑。

# Hooks

前面说到，函数形式创建的组件是没有状态的，如果需要状态的话，就需要用到Hooks。

Hooks其实就是React为函数组件引入副作用的解决方案。

常见的有`useState`，通用的有`useEffect`。这些useXX都是在函数组件中引入副作用。

## useEffect

useEffect()的作用就是指定一个副效应函数，组件每渲染一次，该函数就自动执行一次。第二个参数可以指定变化的依赖变量，也可以返回一个函数，作为组件重新加载或者组件卸载时被调用，一般是一些清理操作。

## 自定义

可以讲Hooks的代码封装起来，生成一个自定义Hooks，方便共享。

```js
const usePerson = (personId) => {
  const [loading, setLoading] = useState(true);
  const [person, setPerson] = useState({});
  useEffect(() => {
    setLoading(true);
    fetch(`https://swapi.co/api/people/${personId}/`)
      .then(response => response.json())
      .then(data => {
        setPerson(data);
        setLoading(false);
      });
  }, [personId]);
  return [loading, person];
};

const Person = ({ personId }) => {
  const [loading, person] = usePerson(personId);

  if (loading === true) {
    return <p>Loading ...</p>;
  }

  return (
    <div>
      <p>You're viewing: {person.name}</p>
      <p>Height: {person.height}</p>
      <p>Mass: {person.mass}</p>
    </div>
  );
};
```
[Hooks Ref](https://zhuanlan.zhihu.com/p/347136271)

# Ref

有时父组件需要拿到子组件去调用子组件的方法。此时就需要在父组件中拿到子组件的引用。

如果子组件是一个函数组件，需要用到`useImperativeHandle` Hook，改方法作用是对外暴露可调用的方法。

[React Ref](https://reactjs.org/docs/refs-and-the-dom.html)

[useImperativeHandle](https://reactjs.org/docs/hooks-reference.html#useimperativehandle)

# Context

一般我们是在父组件中将props传递给子组件，但是如果层级比较多，那么中间多一些组件传递只是为了给最底层的组件。此时可以使用Context，可以让value在整个组件树中都可以获取到。

[Doc](https://reactjs.org/docs/context.html)

# TypeScript

react 支持ts，记录下过程，以及过程中遇到的一些错误：

1. npm 安装ts； `npm install typescript --save-dev`

2. 文件名调整；react组件改成tsx，普通js改成ts。

上面两步好了以后，重新npm start。中间可能出现错误，但是会给出可能的解决方案，我按步骤操作完成后，继续npm start，此时的报错就是ts相关的语法错误。

改完以后觉得主要的工作还是去添加type，指定参数的类型。还有一个点是connect函数，mapStateToProps和mapDispatchToProps参数如果只有一个，那么缺失的需要传null，否则就会出错。

# npm

npm 包依赖管理。

npm install --save xx; 将模块依赖写入dependencies 节点。

npm install --save-dev xx; 将模块依赖写入devDependencies 节点。

[npm install](https://segmentfault.com/a/1190000021468231)

# 一些函数

## compose

compose接受一系列的函数，将其组合起来，返回一个新的函数，这样外部调用的时候就需要只调用一次函数，而不需要多次调用。

{% highlight javascript%}
function testCompose() {
	const fn1 = num => {
	  console.log("f1 be called");
	  return num + 1;
	}
	const fn2 = num => {
	  console.log("f2 be called");
	  return num + 2;
	};
	const fn3 = num => {
	  console.log("f3 be called");
	  return num + 3;
	};

	let x = 10;

	const fc1 = compose(fn1, fn2, fn3);
	const fc2 = compose(fn3, fn2, fn1);

	let res3 = fc1(x);
	let res4 = fc2(x);

	console.log("res3 = " + res3 + ", res4 = " + res4);
}
{% endhighlight %}

默认compose函数的调用顺序是从参数列表的最后一个函数开始，然后依次进行调用。

compose的实现可以使用递归实现，参考实现如下：

{% highlight javascript%}
function compose(...fns) {
    let len = fns.length
    let res = null
    return function fn(...arg) {
        res = fns[len - 1].apply(null, arg) // 每次函数运行的结果
        if(len > 1) {
            len --
            return fn.call(null, res) // 将结果递归传给下一个函数
        } else {
            return res //返回结果
        }
    }
}
{% endhighlight %}

[ref](https://zhuanlan.zhihu.com/p/345122007)

# References

[更多资料](https://developer.mozilla.org/en-US/docs/Learn/Tools_and_testing/Client-side_JavaScript_frameworks/React_resources)

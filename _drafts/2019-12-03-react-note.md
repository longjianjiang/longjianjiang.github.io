
# Component

React 中最简单的组件就是函数的形式进行创建，此时的组件是无状态的，仅仅负责展示内容。当需要处理状态的时候就需要使用class的形式继承`React.Component`。

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

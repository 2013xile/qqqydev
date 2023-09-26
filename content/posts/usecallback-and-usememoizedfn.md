---
title: React中的 useCallback 和 ahooks 中的 useMemoizedFn
date: "2023-09-20"
categories:
  - Frontend
  - React
---

> 应该是一个比较常见的问题了，但是在我这段工作开始之前，我有两三年没有正儿八经写前端了，我的 `React` 经验还停留在 `React 16`, 基本上算一个新手，所以记录一下。

事情的起因是我在组件的 `useEffect` 中依赖了一个外部函数，当函数没有添加到依赖数组时，eslint 会报错 "mssing dependency".

<img src="https://s2.loli.net/2023/09/21/CBbOykuAm6VFW7N.png"/>

虽然报错级别是 `warning`, 但是作为一个强迫症，我不能忍啊。啪，很快的，我把这个函数直接就丢到依赖数组里了，这下好了，我发现 `useEffect` 开始不停地触发。

这个问题通过搜索，不难找到答案，因为每次函数的地址都会发生修改，导致 `useEffect` 的依赖数组更新。结合搜索结果，我也很快想起来，有 `useCallback` 这个东西。但是我发现 `useCallback` 在我这个场景下好像用不了，因为我这个函数本身是引入的一个外部定义好的函数，放到 `useCallback` 里还是要求将函数传入依赖数组，不传eslint就会有 `warning`. 作为一个强迫症，我不能忍啊。

后面找到解决方法，可以用 `useRef` 来解决。用 `useRef` 把这个函数套一下，在 `useEffect` 里用 `ref.current` 来调用就可以了，也不会要求传入依赖参数了。

这个问题到上面就解决完了。但是刚好我们的代码库里是有用到 [ahooks](https://ahooks.js.org/), 有一天我在看文档的时候，发现有一个 `useMemoizedFn`. 根据文档的描述:

> 持久化 function 的 Hook，理论上，可以使用 useMemoizedFn 完全代替 useCallback。
> 在某些场景中，我们需要使用 useCallback 来记住一个函数，但是在第二个参数 deps 变化时，会重新生成函数，导致函数地址变化。
> 使用 useMemoizedFn，可以省略第二个参数 deps，同时保证函数地址永远不会变化。

这恰恰是用来完美解决我的问题的，于是我就想知道它是怎么实现的，是不是也是通过 `useRef`. 一看[代码](https://github.com/alibaba/hooks/blob/5412bb719de1666ff2e947dfa3e1d231b7f9746f/packages/hooks/src/useMemoizedFn/index.ts#L12)，确实是。

```TypeScript
function useMemoizedFn<T extends noop>(fn: T) {
  if (isDev) {
    if (!isFunction(fn)) {
      console.error(`useMemoizedFn expected parameter is a function, got ${typeof fn}`);
    }
  }

  const fnRef = useRef<T>(fn);

  // why not write `fnRef.current = fn`?
  // https://github.com/alibaba/hooks/issues/728
  fnRef.current = useMemo(() => fn, [fn]);

  const memoizedFn = useRef<PickFunction<T>>();
  if (!memoizedFn.current) {
    memoizedFn.current = function (this, ...args) {
      return fnRef.current.apply(this, args);
    };
  }

  return memoizedFn.current as T;
}
```

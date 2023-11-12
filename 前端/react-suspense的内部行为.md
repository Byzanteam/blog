# react 中的 Suspense 组件和 use Hook

## 介绍

`Suspense` 组件是 react 的内置组件，它允许在子组件完成加载前展示 `fallback` ，可以使用 `lazy` 或者 `use` 告诉 Suspense 子组件是否加载结束

`use` 是 react 快要更新的一个 hook，它接受一个 `Promise`，并且返回这个 Promise 解开后的内容，通过这个钩子可以在同步函数中获得异步的数据（hook 只能在顶层，所以这里的同步函数指的是组件），并且在等待 Promise 时，会触发离它最近的外部 `Suspense` 组件显示它的 `fallback`

就像下面的两个组件，AsyncComp 会在同步的上下文中获得异步的数据，而 App 会在 AsyncComp 获得数据的两秒期间渲染 fallback 中的内容

```react
// father.tsx
function App() {
  return (
    <Suspense fallback={<div>loading...</div>}>
      <AsyncComp />
    </Suspense>
  )
}

// child.tsx
function asyncFunc() {
  return new Promise<string>((resolve) => {
    setTimeout(() => {
      resolve('data')
    }, 2000)
  })
}

export function AsyncComp() {
  const data = use(asyncFunc())

  return <>{data}</>
}
```

## suspense 和 use 的原理

只需要通过 `use` 就能在同步的上下文中获得异步的数据，看起来是很神奇的事，但这只是 `Suspense` 和 `use` 背后做了处理才让我们使用时有这样的感觉，虽然是写的同步代码，但背后是不可能离开异步的

在 `AsyncComp` 中加上 log，就能发现：在 `use` 前的 log 执行了两次，`use` 后的 log 执行了一次 —— `use` 把 `AsyncComp` 的执行中断了

```react
export function AsyncComp() {
  console.log('before get data') // 执行两次

  const data = use(asyncFunc())

  console.log('after get data') // 执行一次

  return <>{data}</>
}
```

其实这里就是 use 函数在做的事了，它会缓存接受的 Promise 在 resolve 后的数据，如果有缓存，那么就返回这个数据，如果没有，就中断组件的执行，等到 resolve 后、组件再次执行时，就会得到数据，这样体感上就完成了在同步上下文中获得异步数据的过程

看下面一段代码，这里用函数的形式简单还原了这一过程（没有考虑错误处理，数据隔离，useTransition 等）：use 函数会抛出 promise 错误中断函数的执行，suspense 则会捕获 use 抛出的错误，并且等到错误 resolve 后再次执行

```typescript
function asyncFunc() {
  return new Promise<string>((resolve) => {
    setTimeout(() => {
      resolve('data')
    }, 2000)
  })
}

function assertPromise<T>(promise: unknown): asserts promise is Promise<T> {
  if (!(promise instanceof Promise)) {
    throw new TypeError('input not Promise')
  }
}

let cache: any = null

function use<T>(promise: Promise<T>): T {
  if (cache) return cache

  throw promise.then((data) => (cache = data))
}

function suspense(children: () => any) {
  try {
    children()
  } catch (promise) {
    assertPromise(promise)

    promise.then(() => {
      children()
    })
  }
}

function children() {
  const data = use(asyncFunc())

  console.log(data)
}

suspense(children)
```

Suspense 的原理就是这样，把这里的 `use` 替换为一开始示例代码的 `use` 会发现行为是没有区别

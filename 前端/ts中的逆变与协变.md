# 介绍

先说结论：

- 在被作为参数的函数中，函数的参数的类型得比定义的范围更小（这个就叫做**逆变**）
- 在被作为参数的函数中，函数的返回值类型得比定义的范围更大（这个就叫做**协变**）

> 注意： 该特性需要在`tsconfig.json`中开启`--strictFunctionTypes`，或`--strict`

## 逆变

接下来举个例子来具体说明下：

```typescript
interface A {
    a: string;
}
interface B extends A {
    b: string;
}
interface C extends B {
    c: string;
}

declare function use(fn: (arg: B) => void): void;

declare function fnA(arg: A): void;
declare function fnB(arg: B): void;
declare function fnC(arg: C): void;

use(fnA);
use(fnB);
use(fnC); // error
```

这里的逻辑是这样的：

`use`函数接收的函数需要对`B`这个类型的参数进行处理，ts 会默认为`fn`函数内会使用到`a b` 两个属性

- 如果参数是`fnA`，`use`函数传递给`fnA`的参数类型为`B`，而`fnA`中只会对`B`类型中含有的`a`属性进行操作，是正常处理的
- 如果参数是`fnC`，`use`函数传递给`fnC`的参数类型为`B`，而`fnC`中会对`B`类型中不含有的`c`属性进行操作，这样就可能出现错误

同理，下面这些调用也是ok的

```typescript
use(() => {});
use((_arg: {}) => {});
use((_arg: object) => {});
```

这样写也是逆变产生的错误 ~~正常代码中不应该有这样的写法~~

```typescript
interface Fn {
    (arg: B): void;
}

const fnA: Fn = (arg: A) => {}; 
const fnB: Fn = (arg: B) => {}; 
const fnC: Fn = (arg: C) => {}; // error
```

## 协变

同样还是那三个类型，作用到函数的返回值上时就相反了

```typescript
declare function use(fn: (arg: unknown) => B): void;

declare function fnA(arg: unknown): A;
declare function fnB(arg: unknown): B;
declare function fnC(arg: unknown): C;

use(fnA); // error
use(fnB);
use(fnC);
```

可以认为`use`函数中会调用到`fn`的返回值，那么很明显，返回值的范围需要和`B`相等或比它更大才可以，所以`fnA`是不安全的

# 替换成vue的情况

这种情况除了在函数做参数时出现，在`vue`的入参中也会遇到：

在组件CompA中定义一个函数式的 props

```typescript
interface Props {
    queryHandler: (query: object) => void;
}
```

调用时

```vue
<template>
    <!-- error: 函数参数类型不兼容 -->
    <CompA :query-handler="queryHandler" />
</template>

<script lang="ts" setup>
interface Query {
    a: string;
}
function queryHandler(query: Query) {}
</script>
```



# 因逆变产生错误的回避方案

> 逆变本身的设计是合理的，出现需要使用回避方案的情况一般是外部函数对传入函数参数没有特别限制，但是传入的函数定义时对参数有限制

只需要在外部的函数上添加泛型即可解决

```typescript
declare function use<Arg extends B>(fn: (arg: Arg) => void): void;

use(fnC); // success

// 即使入参不是继承自B 下面这几个也不会报错，因为它们仍然满足逆变的特性
use(() => {});
use((_arg: {}) => {});
use((_arg: object) => {});
use(fnA);

// 当然，还是得满足继承条件或逆变的特性
use((_arg: { a: string; d: string }) => {}); // error
```

同样的，在 vue 中也可以使用泛型解决（vue3.3 后新增特性限定）

CompA：

```vue
<script setup lang="ts" generic="Query extends object">
interface Props {
    queryHandler: (query: Query) => void;
}

defineProps<Props>();
</script>
```

使用时：

```vue
<template>
    <CompA :query-handler="queryHandler" />
</template>

<script lang="ts" setup>
interface Query {
    a: string;
}
function queryHandler(query: Query) { }
</script>
```

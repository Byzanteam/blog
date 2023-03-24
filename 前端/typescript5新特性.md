# 新操作符 satisfies

> 这个操作符实际上是`4.9`引入的

如下一段赋值，如果在赋值时声明会得到和`Bar`相同的类型，但是使用使用`satisfies`，它会在类型判断时按照实际赋上的值来推断类型

```typescript
interface Bar {
    a: string | number;
}

const bar: Bar = {
    a: "a", // string | number
};
let bar1 = {
    a: "a", // string | number
} as Bar;

let bar2 = {
    a: "a", // string
} satisfies Bar;
```

通过`satisfies`赋值的变量，它的类型会被重新推断，所以通过同样类型声明的两个变量`bar1 bar2`不能互相赋值

```typescript
bar2 = bar1; // error 不能把 { a: string | number } 赋值给 { a: string }
```

```typescript
bar1 = bar2; // success { a: string } 当然可以复制给 { a: string | number }
```

上面是在对象或数组上的表现，在简单类型上使用会获得类似`as const`的结果

```typescript
type NumberOrString = number | string;

const n = 1 satisfies NumberOrString; // n 的类型为 1 和下面的类型相同
const nn = 1 as const;
```

# 函数参数通过 const 声明提供更好的类型支持

声明一个这样的函数，想要返回对象类型符合预期，需要使用`as const`来声明参数

```typescript
declare function func<T extends readonly string[]>(arg: T): Record<T[number], string>;

const r = func(["a", "b"]); // Record<string, string>

const rr = func(["a", "b"] as const); // Record<"a" | "b", string>
```

第一种是不符合预期的，第二种写起来又很繁琐

现在可以通过对泛型添加`const`声明来获得预期的结果

```typescript
declare function func2<const T extends readonly string[]>(arg: T): Record<T[number], string>;

const rrr = func2(["a", "b"])
```

不过要把参数用变量传入时，还是需要给变量加上`const`声明

```typescript
const arg1 = ["a", "b"];
const arg2 = ["a", "b"] as const;

const r = func2(arg1); // Record<string, string>
const rr = func2(arg2); // Record<"a" | "b", string>
```

对于对象的属性，它仍然有用（这是一个官方例子）

```typescript
type HasNames = { names: readonly string[] };
function getNamesExactly<const T extends HasNames>(arg: T): T["names"] {
//                       ^^^^^
    return arg.names;
}

// Inferred type: readonly ["Alice", "Bob", "Eve"]
// Note: Didn't need to write 'as const' here
const names = getNamesExactly({ names: ["Alice", "Bob", "Eve"] });
```

const 声明最好拿来修饰不可变的参数，如上面例子的只读数组，虽然它可以跟上可变参数，但会得到预期外的结果

```typescript
declare function func<const T extends string[]>(arr: T): T;

const r = func(["a","b"]); // string[]
```

更或者会得到一个错误的结果

```typescript
declare function pickByArray<T extends object, const K extends (keyof T)[]>(
    object: T,
    keys: K
): Record<K[number], string>;

const obj = {
    a: "a",
    b: "b",
};

const rr = pickByArray(obj, []); //  Record<"a" | "b", string>
```

> 另外：
>
> 第二个参数加上`readonly`后的行为其实和没有使用`const`限制的结果区别不大，但是更推荐加上
>
> 因为在`K extends (keyof T)[]`的情况下，如果参数为空数组，得到的类型为`Record<any, string>`，如果加上，则为`Recrod<never, string>`

# 新增`export type * from "xxx"`

导出`class`时会只把它当做类型导出，不能实例化

# 枚举的变化

**什么是`union enum`（联合枚举）**

ts 中每个枚举值都会被作为一个类型看待，所以它的多个属性可以组合起来

```typescript
enum Color {
    Red,
    Orange,
    Yellow,
    Green,
    Blue = "blue",
    Violet = "violet",
}

type PrimaryColor = Color.Red | Color.Green | Color.Blue;
```

但是在 ts5 之前如果有一个属性是计算值，会得到错误

```typescript
enum Color {
    Red,
    Orange,
    Yellow,
    Green,
    Blue,
    Violet = Date.now(), // 或者是 null NaN Infinty 等值
}

type PrimaryColor = Color.Red | Color.Green | Color.Blue; // error
```

在 ts5 之后就不会报错了，并且含有字符串的枚举现在也支持使用计算值赋值了

```typescript
enum Enum {
    a = Math.random(), // ts5之前: 含字符串值成员的枚举中不允许使用计算值。 ts(2553)
    b = "b",
}
```

# 类型声明文件支持按照文件类型扩展

> 使用该特性需要开启`allowArbitraryExtensions`选项

可以通过写`${filename}.d.${extends}.ts`文件来声明非 js 文件的类型

```typescript
// src/style/style.css
box { /** */ }

// src/style/style.d.css.ts
// 类型声明文件和被声明文件要在同一目录
declare const style: {
  box: string
}
export default style

// xxx.ts
import style from "@/style/style.css"

style.box // string
```

# `extends`支持数组形式

可以通过数组同时继承多个配置文件，数组后面的配置会覆盖前面的

```json
{
    "extends": ["tsconfig.base.ts", "tsconfig.node.ts"]
}
```

# `moduleResolution`选项新增`bundler`选项

新的模块解释器，主要需要注意的是该选项开启后不会导入 exports 中的`require`选项，具体的可以看[这里](https://github.com/microsoft/TypeScript/pull/51669)

总之未来`vite`项目都推荐启用这个选项

# `customConditions`选项

`package.json`中可以声明`export`选项来声明该包在不同环境下导出的文件：

```json
{
    "exports": {
        ".": {
            "import": "./index.mjs",
            "require": "./index.cjs"
        }
    }
}
```

可以通过这个选项来自定义本项目中导入的条件，如下配置，在项目中始终会导入`index.cjs`

```json
// package.json
{
  "exports": {
    ".": {
      "import": "./index.mjs",
      "require": "./index.cjs"
    }
  }
}
// tsconfig.json
{
    "compilerOptions": {
        "module": "ESNext",
        "moduleResolution": "bundler",
        "customConditions": ["require"]
    }
}
```

# `allowImportingTsExtensions`选项

开启这个选项可以在导入时加上`ts tsx mts`后缀（应该没啥用）

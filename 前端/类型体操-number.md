最近项目上后端返回了这样一个数据

```typescript
interface Data {
    data_1: number;
    data_2: number;
    data_3: number;
    // ...
    data_8: number;
}
```

每个属性都声明实在不够优雅

或许可以`Record<data_${number}, number>`来声明？这样会丢失类型提示，类型约束也变得意义不明

再或者`Record<data_${1 | 2 | 3 | ... | 8}, number>`这样？和最开始的声明没有区别

那么这里想要构建一个接收数字就返回比它小的所有数字的联合的类型，就像这样：

```typescript
type R = GetLessNumbers<9>; // 1 | 2 | 3 | ... | 8
```

## 类型中操作数字的原理

在实现这样一个类型前，首先需要知道怎样能在类型中对数字进行运算，当然类型没有加减法，但是数字可以首先转换为元组，再通过元组的长度来进行计算，下面就是一个求两个数相加的类型

```typescript
// 把数字转换为元组
type NumberToTuple<T, R extends unknown[] = []> = R["length"] extends T
    ? R
    : NumberToTuple<T, [unknown, ...R]>;

// 求两数之和
type SUM<A extends number, B extends number> = [...NumberToTuple<A>, ...NumberToTuple<B>]["length"];

type R = SUM<4, 8>; // 12
```

通过元组的长度这个中间值就可以做到在类型上进行简单运算了

## 实现需求

通过上面的例子可以分析出要实现开始的需求其实也不是很难，只需要在类型中运用递归即可

-   构造一个空元组`Temp`来记录当前的递归次数，构造一个储存数字联合的类型`Result`记录结果
-   判断当前递归次数是否等于传入的数字
    -   等于：返回`Result`
    -   小于：递归调用类型，让`Temp`的长度+1

下面是代码实现：

```typescript
type GetLessNumbers<T extends number, Temp extends unknown[] = [], Result = 0> = Temp["length"] extends T
    ? Result
    : GetLessNumbers<T, [unknown, ...Temp], Result | Temp["length"]>;

type R = GetLessNumbers<8>; // 0 | 1 | 2 | 3 | 4 | 5 | 6 | 7
```

加上左右范围才好用。。。

```typescript
// 通过传入 start end 来获得 [start, end] 范围内的数字联合
type GetRangeNumbers<
    Start extends number,
    End extends number,
    Temp extends unknown[] = NumberToTuple<Start>,
    Result = Start | End
> = Temp["length"] extends End
    ? Result
    : GetRangeNumbers<Start, End, [unknown, ...Temp], Result | Temp["length"]>;

type R = GetRangeNumbers<1, 8>; // 1 | 2 | 3 | 4 | 5 | 6 | 7 | 8
```

所以一开始的那个类型可以这样声明了，完整的类型推断和简短的代码，看着舒服了

```typescript
type Data = {
    [key in `data_${GetRangeNumbers<1, 8>}`]: number;
};
```

## 扩展

虽然实现了需要的类型，但是知道了对数字的操作方法后想要进一步整点活，把加减乘除剩下的运算都实现了吧

### 乘法

乘法本质上就是多次进行加法，`3 x 4`就是三个四相加，那么乘法仍然通过递归实现：

```typescript
type MUL<
    A extends number,
    B extends number,
    T extends unknown[] = [],
    R extends unknown[] = []
> = T["length"] extends A ? R["length"] : MUL<A, B, [...T, unknown], [...NumberToTuple<B>, ...R]>;

type R = MUL<3, 4>; // 12
```

### 减法

首先说明一下，负数在类型中当然也是数字类型，但是通过元组的长度来计算的话当然无法获得一个负数，所以在实现减法之前，必须先实现一个判断两个数大小的类型，这样才能保证输出的正确性

判断数字大小在类型上就是判断两个元组的长度，所以实现它的话就是拿一个元组`A`剪掉另一个元组`B`的长度，根据结果如果大于 0 则`A > B`，反之`A <= B`

```typescript
// 从数组中删去指定长度 如果指定长度大于数组长度 返回[]
type Slice<
    Arr extends unknown[],
    Len extends number,
    Removed extends unknown[] = []
> = Removed["length"] extends Len
    ? Arr
    : Arr extends [infer Head, ...infer Rest]
    ? Slice<Rest, Len, [...Removed, Head]>
    : [];

// 判断是否 A > B
type GT<A extends number, B extends number> = Slice<NumberToTuple<A>, B> extends [] ? false : true;

type R = GT<3, 9>; // false
```

接下来就是实现减法了：

-   首先判断参数`A B`的大小
    -   如果相同返回 0
    -   如果`A > B`则将 A 转换的元组减去 B 的长度
    -   如果`A < B`则反过来计算

```typescript
type SUB<A extends number, B extends number> = A extends B
    ? 0
    : GT<A, B> extends true
    ? [...Slice<NumberToTuple<A>, B>]["length"]
    : [...Slice<NumberToTuple<B>, A>]["length"];

type R = SUB<8, 3>; // 5
type RR = SUB<3, 9>; // 6
type RRR = SUB<4, 4>; // 0
```

需要注意的是`A < B`的情况下不能添加负号，试过通过模板字符串的方式添加负号，但是数字类型一旦转换为字符串类型就无法转换回来了

```typescript
type StringToNumber<T extends string> = T extends `${number}` ? (T extends `${infer R}` ? R : never) : never;

type R = StringToNumber<"4">; // "4"
```

所以实际上减法只能获得两数字差的绝对值

### 除法

实现了大小判断后除法就变得很轻松了

-   首先判断被除数`A`和除数`B`的大小关系
    -   `A == B`：返回当前递归次数+1
    -   `A > B`：在 A 中减去 B 的长度，递归次数+1，再次递归该方法
    -   `A < B`：返回递归次数

```typescript
type DIV<A extends number, B extends number, R extends unknown[] = []> = A extends B
    ? [...R, unknown]["length"]
    : GT<A, B> extends true
    ? DIV<SUB<A, B>, B, [...R, unknown]>
    : R["length"];

type R = DIV<8, 3>; // 2
type RR = DIV<8, 2>; // 4
```

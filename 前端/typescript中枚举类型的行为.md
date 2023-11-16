# Typescript 中的枚举转换

## 枚举直接作为类型的行为

这里使用一个纯字符串枚举来做例子

```typescript
enum FieldType {
  BooleanField = 'booleanField',
  CheckboxField = 'checkboxField',
}

// 当做类型
type MetalField = {
  type: FieldType
  _type: FieldType.CheckboxField
}
// 当做变量
const field: MetalField = {
  type: FieldType.CheckboxField,
}
```

可以看到，枚举在 typescript 中是一个很特殊的东西，它可以当做**类型**使用，它身上的属性却可以当做**变量**使用

同时还能直接对枚举和其属性直接使用`keyof`操作符

```typescript
type A = keyof FieldType
type B = keyof FieldType.CheckboxField
```

可以发现，类型 A B 的内容都是`string`上有的方法，这代表这个枚举在做类型的时候和字符串类型是等价的，因为这个枚举本身就只有字符串

```typescript
type FieldTypeIsString = FieldType extends string ? true : false // true
type CheckboxFieldIsString = FieldType.CheckboxField extends string ? true : false // true
```

更进一步，它其实就是所含数据的联合类型：

```typescript
type IsUnionString = FieldType extends 'checkboxField' | 'booleanField' ? true : false
```

同理，如果这个枚举不是单纯的字符枚举，那么它会被判断为所含数据类型的联合

```typescript
enum TestEnum {
  Num,
  Str = 'string',
}

type IsStringNumberUnion = TestEnum extends string | number ? true : false // true
type IsUnionType = TestEnum extends 0 | 'string' ? true : false // true
```

这里做一个总结：

- 枚举如果直接被当做类型，它会被判断为所含数据组成的联合类型
- 虽然类型行为上和联合类型相似，但是类型被设置为枚举的变量只能通过枚举来设置值，不能直接通过值来设置

通过枚举的类型和联合类型相似的特性，就可以理解下面这串代码的行为了：把枚举中的值全部转换为其对应的字符串\*\*的联合

```typescript
type EnumValueUnion<T extends string | number> = `${T}`

type FieldTypeValues = EnumValueUnion<FieldType> // 'booleanField' | 'checkboxField'
type TestEnumValues = EnumValueUnion<TestEnum> // '0' | 'string'
```

## 枚举加上 typeof 后的行为

如果需要获得枚举的键组成的联合类型，可以通过下面的方法

```typescript
type EnumKeyUnion = keyof typeof FieldType
```

这里加上`typeof`后的类型就变成了以枚举的键为键，以**具体枚举值**为值的对象类型

```typescript
cosnt enum: typeof FieldType = {
  BooleanField: FieldType.BooleanField, // 这里的类型是 FieldType.BooleanField 不是 'booleanField'
  CheckboxField: FieldType.CheckboxField,
}
```

所以上面的行为也可以解释了

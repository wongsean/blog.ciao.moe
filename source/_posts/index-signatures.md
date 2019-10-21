---
title: Index Signatures
date: 2019-09-24 14:04:07
categories:
  - tech
---

TypeScript 中的 Index Signature 的使用场景不可以说不频繁。例如，其可以用来表示可以通过方括号 `[]` 访问字段获取值的数据结构，例如 TypeScript 内建类型 `Array<T>` 中就包含如下实现：

```typescript
interface Array<T> {
  ...

  [n: number]: T
}
```

同时，在 JavaScript 中我们似乎更少用到 Map 类的实例，而更多的是直接构建 object 来实现类似 map 作用的数据结构，TypeScript 中称之为 Record，其类型通常也可以表示为 `type Record<K extends keyof any, T> = { [P in K]: T }`。

然而，如果尝试在追求使用 TypeScript 实现类型安全，或是寄希望于代码通过编译就不会有运行时错误产生，则需要小心谨慎的使用，或是避免 Index Signature。这是因为使用 Index Signature 获取的值类型并不正确，例如

```typescript
const map: { [key: string]: number } = {
  foo: 1,
  bar: 2
};

const value = map["baz"];
// value: number
```

这里使用不存在的 key 获取到的结果是 `undefined`，但由 TypeScript 推导出的值类型并不包含这种可能性， 在 web 应用中我们常常会大量使用到 array 和 map 来批量处理数据，或者映射数据结构，总难以避免会忘记处理这隐藏在安全类型下危险的 `undefined`，从而造成不必要的运行时错误。

其实早在 2017 年初就有人在 [microsoft/TypeScript](https://github.com/microsoft/TypeScript) 提了 [issue](https://github.com/microsoft/TypeScript/issues/13778)。当然也有人随即就提出了谁都能想到的解决方案——修改 Index Signature 的值定义，加上 `| undefined`。但显然这并不能解决问题，在定义上面的 map 时，我们仅希望获取值的时候，类型能够明确表示值可能是不存在的，但在值类型定义后加上 `| undefined`，首先改变了语义，其次也无法阻止往 array 或 map 中添加 `undefined` 值。

当然这个问题至今不被解决，一方面是更多的人使用 TypeScript 只是作为一种 JavaScript 的补充，它已经很好的弥补了后者的不足，且作为渐进式类型系统没有迁移门槛；另一方面是 TypeScript 依然需要保持对 JavaScript 代码的完全兼容，这意味着很多 JavaScript 的遗毒势必会影响 TypeScript 构建一个更安全的类型系统，之前讨论 get/set 的也仅是一隅，全面考虑未必能得出优雅的方案。

我时常想，如果是脱离了 JavaScript 的 TypeScript 应该能成为一门更好的语言，但应该无法成为一门流行的语言，现在的更多是权衡取舍后的结果。当然，最终有些人选择了妥协，有些人选择通过各种 work around 维护内心的秩序，有的人在心里期盼——愿世界没有 JS。

---
layout: post
title: "Ruby Constant Lookup"
slug: ruby-constant-lookup
date: 2017-06-25 17:57:07 +0800
categories: tech
---

写 Ruby on Rails 已有一年有余，从最初接触时的惊艳感，到至今也依然觉得气有，但随着渐渐走近 Rails 的神秘魔法的根源，些许不优雅之处和槽点也可窥见一斑，Rails 作为 Ruby 圈不可避免的重量级框架，似乎在理念上偶尔也与 Ruby 貌合神离。为了几年这一年多里踩过的坑（或是文档读的不够仔细），也为了给自己一个总结所有所见所想所调查的机会，写下这一个系列，从常量查询说起。

当然，个人水平有限，又盲目自信，内容可能错误多多，还望兼听则明，也欢迎随时指正，就酱！

## 定义常量
想要了解 Ruby 查询常量的基本过程，我们需要先知道 Ruby 是如何定义以及存储常量的。

```ruby
# module is a constant
module Namespace
  # class is also a constant
  class Something
    # still a constant
    SOME_VALUE = true
  end
end
```
Ruby 中存在词法作用域的概念，与大多数语言不同的是，`module` 与 `class` 关键字会打开一个全新的作用域。上文中常量 `SOME_VALUE` 定义于 class `Something` 作用域中，那么 `SOME_VALUE` 实际上被储存在哪里呢？

在 Ruby 的 C 实现中，每一个 Ruby class 会对应一个 RCLASS 的 C 结构体，而在这个作用域中定义的常量都会记录在这个 C 结构体中，在 irb 中我们可以通过 `Namespace::Something.constants(false)`[^footnote1] 来查看定义在 `Something` 中的常量。

[^footnote1]: 通常有两个 `constants` 方法，这里特指定义在 `Module` 中的**实例**方法。当参数 `inherit = false` 时，该方法将仅列举存储在类中常量表的常量名。

```ruby
irb(main):008:0> Namespace::Something.constants
=> [:SOME_VALUE]
```

同理，不难想到，class `Something` 定义在 module `Namespace` 所打开的作用域中，因此:

```ruby
irb(main):010:0> Namespace.constants
=> [:Something]
```

那么定义在顶层作用域的 module `Namespace` 又是被存储在哪里的呢？简明扼要的说，他们存储在 `Object` 内的常量表内。通过 `Object.constants(false)` 你可以获取到你定义的所有顶层常量。

## 常量查询 -> nesting
上文提到过，每当 `module` 或 `class` 关键字出现的时候，Ruby 会打开一个全新的作用域，那么 Ruby 又是如何实现，类似传统代码块作用域，内层访问外层作用域常量的特性的呢？

在 Ruby 的 C 实现中，结构体 `rb_cref_t` 用来表示当前作用域:

```c
typedef struct rb_cref_struct {
    VALUE flags;
    const VALUE refinements;
    const VALUE klass;
    struct rb_cref_struct * const next;
    const rb_scope_visibility_t scope_visi;
} rb_cref_t;
``` 
其中，属性 klass 表示当前作用域所在 class/module，next 则指向代表上一层作用域的 cref 结构体。这一条通过 next 指针而产生的链表，很好的表示了从当前作用域到顶层作用域的层级关系。在 irb 中我们可以通过 `Module.nesting` 来查看该链表的 klass 值映射。

```ruby
module A
  class B
    class C
      p Module.nesting # => [A::B::C, A::B, A]
      
      p B == ::A::B # => true
    end
  end
end
```
按照上述代码，当我们尝试在上述代码中访问常量 `B` 时，Ruby 会按照 `Module.nesting` 所展现的顺序，先查询 `A::B::C` 下的常量表，发现找不到，再查找 `A::B` 下的常量表，同样没有。最后，在 `A` 的常量表中找到了 `B` 的定义。


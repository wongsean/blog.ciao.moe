---
layout: post
title: "Ruby Constant Lookup"
slug: ruby-constant-lookup
date: 2017-06-25 17:57:07 +0800
categories: tech
reading_minutes: 10
---

写 Ruby on Rails 已有一年有余，随着渐渐开始走进 Rails 的神秘魔法的根源，些许不优雅之处和槽点也可窥见一斑，Rails 作为 Ruby 圈不可避免的重量级框架，似乎在理念上偶尔也与 Ruby 貌合神离。为了几年这一年多里踩过的坑（或是文档读的不够仔细），也为了给自己一个总结所有所见所想所调查的机会，写下这一个系列，从常量查询说起。

当然，个人水平有限，又盲目自信，内容可能错误多多，还望兼听则明，也欢迎随时指正，就酱！

## Constant Definitions
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

那么定义在顶级作用域的 module `Namespace` 又是被存储在哪里的呢？简明扼要的说，他们存储在 `Object` 内的常量表内。通过 `Object.constants(false)` 你可以获取到你定义的所有顶层常量。

## Module.nesting
上文提到过，每当 `module` 或 `class` 关键字出现的时候，Ruby 会打开一个全新的作用域，那么 Ruby 又是如何实现，类似传统代码块作用域，内层访问外层作用域常量的特性的呢？

在 Ruby 的 C 实现中，结构体 `rb_cref_t`[^footnote2] 用来表示当前作用域:

[^footnote2]: 基于 Ruby 2.4 的实现，在 Ruby 2.3 之前是 `NODE` 结构体

```c
typedef struct rb_cref_struct {
    VALUE flags;
    const VALUE refinements;
    const VALUE klass;
    struct rb_cref_struct * const next;
    const rb_scope_visibility_t scope_visi;
} rb_cref_t;
``` 
其中，属性 klass 表示当前作用域所在 class/module，next 则指向代表上一层作用域的 cref 结构体。这一条通过 next 指针而产生的链表，很好的表示了从当前作用域到顶级作用域的层级关系。在 irb 中我们可以通过 `Module.nesting` 来查看该链表的 klass 值映射。

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
那么当我们尝试在上述代码中访问常量 `B` 时，Ruby 会按照 `Module.nesting` 所呈现的顺序，先查询 `A::B::C` 下的常量表，发现找不到，再查找 `A::B` 下的常量表，同样没有。最后，在 `A` 的常量表中找到了 `B` 的定义。

```ruby
module A; end

module A::B
  class C
    p Module.nesting # => [A::B::C, A::B]
    
    p B == ::A::B # => ???
  end
end
```
可以猜测到以上代码的结果了么？以上代码会抛出异常 `NameError: uninitialized constant A::B::C::B`
    

## Ancestors
仅通过查找作用域链并不能满足所有需求，我们也希望在子类中访问超类中的常量

```ruby
class Base
  CONST = 'constant in base'
end

class Sub < Base
  p CONST # => constant in base
end

p Sub::CONST # => constant in base
```

熟悉面向对象便不难猜到，Ruby 也会通过继承树查询常量。Ruby 会通过当前作用域类[^footnote3]的继承树，我们可以通过 `Sub.ancestors` 来了解继承树的结构。

[^footnote3]: 需要注意的是，继承树的起点并不是 `self.class` 而是当前作用域类

这同时就产生了一个问题，当父作用域与超类中同时存在目标常量的定义，Ruby 会如何选择？尝试运行以下代码。

```ruby
class Base
  CONST = 'constant in base'
  CONST_1 = 1
end

module Namespace
  CONST = 'constant in namespace'
  class Sub < Base
    p Module.constants
    p CONST # => 'constant in namespace'
  end
  p Sub::CONST  # => 'constant in base'
end
```
第一句 `p CONST` 的输出是 'constant in namespace'，这时我们这时可以确定 Ruby 会先尝试在作用域链中查找常量，然后才是继承树。
第二句 `p Namespace::Sub::CONST` 看起来和第一句很类似，但输出结果却并不相同，事实上在运行第二句的时候，我们会先查找常量 `Sub`，然后再在 `Sub` 下查找常量 `CONST`，此时 Ruby 不会在考虑作用域链，而是直接进入第二步，查找 `Sub` 的继承树。除此之外，连个这对 `CONST` 的查找仍然有一些不同，这会在之后的内容里详细说明。

## Toplevel Constants?
等等？我们似乎忘了什么？通过 `Module.nesting` 似乎无法查询到顶级作用域的常量，那么定层作用域的常量是如何查询的？Ruby 是否对顶层常量有特殊的处理？

是，也不是。
说出来你可能不信，Ruby 是通过继承树找到这些顶层常量的。如果你还记得，Ruby 将顶层常量保存在 `Object` 中。

```ruby
class MyClass
  p Math::PI
end
```
因此在上述代码中，当我在 `MyClass` 的类作用域中尝试访问顶层常量 `Math` 时，Ruby 是通过 `MyClass` 的继承树查找到 `Object` 内存储的顶层常量的。
看来问题解决了！，我们并不需要引入一个新的规则来解决顶层常量查找问题了，可喜可贺！

问题真的解决了吗？

如果你十分熟悉 Ruby 的内部的继承树，你可能会注意到 `Object` 并不是继承树的顶端，这还不是最要紧的，当我们定义时常被我们作为命名空间的 module 的时候，其继承树中甚至可能这有这个 module 了！

```ruby
module Namespace; end
p Namespace.ancestors # => [Namespace]
```

这可和说好的不一样啊！我们是时常需要在这些 module 作用域中访问顶层常量的。于是 Ruby 在搜索继承树的逻辑后加了句，如果这个类是个 module，我们就从 `Object` 的继承树中再搜索一遍！这下真的皆大欢喜了，至于那些在继承树中处于 `Object` 之上位置的类，例如 `Kernel, BasicObject` 我们管不了那么多了！
事实上这也是 `BasicObject` 类作用域时常被作为一个特殊的，干净的作用域使用的原因。

```ruby
class BasicObject
  p String # => NameError: uninitialized constant BasicObject::String
end
```

## Constants Lookup, The Ugly
让我们来看一下以下代码：

```ruby
class Hash
  p String == Hash::String
end
```

在一个类作用域（这里是 `Hash`）中可以访问顶层常量（这里是 `String`）显然正是我们之前期望得到的结果。但 `Hash::String` 是什么？显然并不存在一个这样的类。于是，当我们满心期望着这段代码抛出异常时，你可能会惊讶于得到以下结果：

```ruby
(irb):21: warning: toplevel constant String referenced by Hash::String
true
```

什么？这怎么可能？
原因其实很简单，两种对 `String` 常量的查询过程，在搜索继承树这一步骤是极其极其相似的，他们都尝试在 `Hash` 内查找常量 `String` 的定义，查找不到后都会沿继承树向上搜索，显然也都会搜索到 `Object`。

这会出问题吗？
事实是会的，尤其在 Rails autoloading 的情况下，由于 Rails autoloading 依赖于 `const_missing` 的实现，当一个同名的顶层常量已经率先加载的情况下，程序可能将你的常量错误的引用到该顶层常量上，仅仅给出一条 warning，淹没在无尽的 log 中，下面的代码摘自 [Rails 官方文档](http://guides.rubyonrails.org/autoloading_and_reloading_constants.html#when-constants-aren-t-missed)[^footnote4]

[^footnote4]: 值得庆幸的是 Ruby 2.5.0 中，终于将 warning 修改为直接抛出异常

```ruby
# app/models/hotel.rb
class Hotel
end
 
# app/models/image.rb
class Image
end
 
# app/models/hotel/image.rb
class Hotel
  class Image < Image
  end
end
```

```bash
$ bin/rails r 'Image; p Hotel::Image' 2>/dev/null
Image # NOT Hotel::Image!
```




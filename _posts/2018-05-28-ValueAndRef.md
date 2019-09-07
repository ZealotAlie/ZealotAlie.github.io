---
title: "从迭代器到Unity的Coroutines"
tags: C#
---

C#包含三种类型

    - 值类型(Value types)

    - 引用类型(Reference types)

    - 指针类型(Pointer types)

值类型直接保存数据，引用类型保存的是对实际数据的引用，引用类型也被称为是object。指针类型只能在unsafe模式下使用，这里不做讨论。<!--more-->

## 值类型

值类型包括两个主要的类别：

    1. Structs结构体

    2. Enumerations枚举

结构体又进一步分为：

    1. Numeric 数值类型

      - 整形类型

      - 浮点类型

      - decimal

    2. bool 

    3. 用户自定义结构体

与引用类型不同，不能从值类型派生新类型。 但是，与引用类型一样，结构可以实现接口。值类型不能为null（但是可以使用可空类型 [nullable types](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/nullable-types/index)）。

## 引用类型

以下关键字通常用于声明引用类型：

    1. class

    2. interface

    3. delegate

此外C#也有内建的引用类型：

    1.[dynamic](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/dynamic) （只在编译时存在，在运行时则不存在，运行时为object）

    2.object

    3.string

经常有人用C/C++中的指针与C#中的引用做类比，但实际上C#中的引用要比指针简单得多（而且C#中有专门的指针），引用只做两件事，其一是通过解引用对它指向的数据进行操作，其二就是比较两个引用，判断它们是否是指向同一块空间的，所以基于这点，我更倾向于用句柄（handle）来描述引用而不是指针。另外接触C#的人不一定都具有C/C++的知识，所以用指针类比反而不利于理解引用（用一个更难理解的概念去解释一个较容易理解的概念显然是不合常理的）。

## 两者区别

在讨论值类型与引用类型的区别时，人们经常提到堆（heap）和栈（stack），首先应该明确一点，引用类型所引用的数据位于堆上，而值类型**有时**是位于栈上。

<u>值类型之所以称为值类型是因为它们总是 copied "by value"，这才是值类型最根本的定义，值类型和引用类型与堆和栈本身是无关的。</u>

当一个值类型被创建时，一块内存空间被分配给这个实例。

当一个引用类型被创建时，实际上可能会需要两块内存空间。

```c#
Point p1  = new Point();  // Point is a *struct*
Form f1   = new Form();   // Form is a *class*
```

对p1，一块内存空间被分配给p1，而对f1则需要两块内存空间：一块用于存储引用f1自身，另一块数据用于存储引用所指向的数据（Form）。

当我们写成下面的形式看起来会清楚一些

```c#
Form f1;          // Allocate the reference
f1 = new Form();  // Allocate the object
```

这也解释了值类型和引用类型拷贝时的不同。

```c#
Point p2  = p1;
Form f2   = f1;
```

p2作为一个结构体，拷贝了p1成为一个独立的实例，拥有一块自己的内存空间，而对f2，拷贝的实际上只是引用，f1和f2所指向的Form数据还是同一块。

我们知道C#中参数传递默认是值传递，意味着在参数传递的过程中发生了隐式的拷贝。所以传递到函数内部的值参数是一个完全独立的新实例，而对引用类型，拷贝的只是引用本身，而它们所指向的数据仍是同一块，所以在函数内部改变引用类型指向的数据，会导致外部的数据也发生改变，而改变值类型的数据则不会引起外部的改变（除非你使用了ref关键字强制使用引用传递）。

## 值类型引用类型与堆栈

首先我们简单介绍一下堆栈。

![Stack](/assets/images/2018-05-28-ValueAndRef/1.png)
<br>*引用自[C# Heap(ing) Vs Stack(ing) in .NET: Part I](https://www.c-sharpcorner.com/article/C-Sharp-heaping-vs-stacking-in-net-part-i/)*

栈是线性的结构，在分配内存时，新分配的空间总是在栈顶，而释放的时候也是从栈顶开始释放的。分配和释放的过程非常简单，只需要根据所需要的空间移动栈顶的位置即可。由于栈的这种特点，所以必须保证分配和释放的顺序是固定的，最新分配的空间总是最先被释放。

而堆则复杂的多，堆有许多块内存空间，当需要从堆上分配空间时，需要从空余的空间中寻找一块大小合适的空间并将这块空间分配出去。堆上的空间不会被自动释放，需要GC来检测一块空间上的object是否还被引用，如果不被引用则会标记为可删除的，在合适的时候被释放以便于重新被使用。

栈上的分配与释放要比堆上的操作快得多，所以为了效率，我们应该尽可能的多使用栈。但是为什么引用类型的数据不能存放在栈上呢？对引用类型而言，只要还有对象保持着对它的引用，那么我们就不能够去释放这些数据，因此无法保证分配和释放的顺序是固定的。

引用一段话作为结论

["in the Microsoft implementation of C# on the desktop CLR, value types are stored on the stack when the value is a local variable or temporary that is not a closed-over local variable of a lambda or anonymous method, and the method body is not an iterator block, and the jitter chooses to not enregister the value."](https://blogs.msdn.microsoft.com/ericlippert/2010/09/30/the-truth-about-value-types/)

值类型只有在是一个局部变量，且不能是lambda或匿名方法中的外部变量，也不能是迭代块（yield）中的变量的情况下，它才会位于栈上。

## 值类型比引用类型快？

使用值类型确实能够带来性能上的提升。当值类型是本地变量时，它位于栈上，分配和释放都很快，而另一个原因是GC不需要去检测一个值类型是否被引用（这也是为什么我们不能在类中声明ref int这样的成员的原因），所以有一个减轻GC负担的小技巧就是，在构建结构体的时候，尽量确保结构体内部没有引用类型，这样这整个结构体在GC检查的时候都不需要去判断它的引用关系，从而提升性能。

## 参考资料

- http://www.albahari.com/valuevsreftypes.aspx

- https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/types

- https://blogs.msdn.microsoft.com/ericlippert/2009/04/27/the-stack-is-an-implementation-detail-part-one/

- https://blogs.msdn.microsoft.com/ericlippert/2009/05/04/the-stack-is-an-implementation-detail-part-two/

- https://blogs.msdn.microsoft.com/ericlippert/2010/09/30/the-truth-about-value-types/

- https://blogs.msdn.microsoft.com/ericlippert/2009/02/17/references-are-not-addresses/

- https://www.c-sharpcorner.com/article/C-Sharp-heaping-vs-stacking-in-net-part-i/
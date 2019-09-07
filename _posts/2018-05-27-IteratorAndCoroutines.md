---
title: "从迭代器到Unity的Coroutines"
tags: Unity
      C#
---

## Iterator迭代器和foreach循环

Iterator迭代器是设计模式中较为常见的一种，它的意图是提供一种方法顺序访问一个聚合对象中的各个元素而不暴露对象的内部表示。一般而言，Current Item返回当前正在访问的元素，从First开始，通过Next操作逐步执行到最后一个元素从而对容器完成一次访问。<!--more-->

在C#中提供的foreach就是通过迭代器访问容器内元素的一种操作。要使用foreach，容器必须是IEnumerable的实现。

关于 IEnumerable，通过MSDN上关于 [IEnumerable](https://docs.microsoft.com/en-us/dotnet/api/system.collections.ienumerable) 的说明可以了解到实现这个接口只需要实现

```c#
IEnumerator GetEnumerator();
```

这个函数用于返回一个遍历容器的迭代器。这里的IEnumerator是C#中迭代器的抽象接口，这个接口只有两个方法和一个属性

```c#
object Current { get; }
bool MoveNext();
void Reset();
```

和迭代器的一般模式无异，只不过增加了一个Reset操作。

这里需要注意的是，IEnumerable和IEnumerator都有对应的泛型版本IEnumerable<T>和IEnumerator<T>。

IEnumerable<T>的GetEnumerator()返回的是IEnumerator<T>。IEnumerator<T>的Current的返回类型是T而不是object

```c#
T Current { get; }
```

直接返回对应类型而不是object可以避免类型转换带来的性能问题，对引用类型来说这部分的性能问题不大，但对于值类型，这里涉及到了 Boxing 和 Unboxing，在较大的操作中带来的性能问题还是相当可观的，所以尽量实现泛型版本的 IEnumerable 和 IEnumerator。具体的实现细节可以参考[MSDN](https://docs.microsoft.com/en-us/dotnet/api/system.collections.generic.ienumerable-1)。

所以至此我们可以简单的对foreach循环有一个认识，它其实类似于如下的while循环

```c#
var tmp = obj.GetEnumerator();
while( tmp.MoveNext() ) 
{
	doSomething( temp.current);
}
```
## Yield

yield实质是[语法糖](
https://zh.wikipedia.org/wiki/%E8%AF%AD%E6%B3%95%E7%B3%96)，它让程序员能够更方便的去使用迭代器。通过yield，你可以直接使用迭代器操作而不需要去实现 IEnumerable和IEnumerator，也不需要一个临时的Collection来完成迭代。

yield有两种格式的声明

```c#
yield return <expression>;  
yield break;
```

通过一个[例子](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/yield)来了解yield的工作流程

```c#
public class PowersOf2
{
    static void Main()
    {
        // Display powers of 2 up to the exponent of 8:
        foreach (int i in Power(2, 8))
        {
            Console.Write("{0} ", i);
        }
    }

    public static IEnumerable<int> Power(int number, int exponent)
    {
        int result = 1;

        for (int i = 0; i < exponent; i++)
        {
            result = result * number;
            yield return result;
        }
    }

    // Output: 2 4 8 16 32 64 128 256
}
```

每一次foreach的循环中都会调用迭代器方法，当yield return被执行到时，表达式的值会被返回，同时当前的函数的上下文信息被保存下来。下一次循环执行之时会重新从上一次停止的位置继续执行。你也可以使用yield break来终止迭代过程。

上面的例子中，第一次Power执行到yield return result返回2时，Power方法会被暂时停止，int result的值会被保留下来，然后返回到foreach循环体中，执行循环体中的代码，执行完成后再回到Power的迭代中，此时Power从刚才yield return的地方再次开始执行返回4，然后再回到循环体中，依次执行直到Power函数执行完成，循环退出。

yield return除了可以返回IEnumerable之外，还能直接返回IEnumerator，不过直接返回IEnumerator不能使用foreach进行迭代，上面的例子可以改写为

```c#
public class PowersOf2
{
    static void Main()
    {
        // Display powers of 2 up to the exponent of 8:
        var enumerator = Power(2,8);
        while( enumerator.MoveNext() )
        {
            Console.WriteLine("{0} ",enumerator.Current);
        }
    }
    public static IEnumerator<int> Power(int number, int exponent)
    {
       ……
    }
    // Output: 2 4 8 16 32 64 128 256
}
```

## coroutines

实际上问题到此已经很清楚了，我没有看过coroutines的源码，但通过它对yied和IEnumerator的依赖，可以猜测协程实际上就是yield迭代的过程。

当我们使用StartCoroutine时，Unity引擎会通过某种方式进行迭代，从而达到类似于多线程的效果。

## 参考资料

- [yield and coroutines, how do they actually work?](https://answers.unity.com/questions/233010/yield-and-coroutines-how-do-the-actually-work.html)

- [What is the yield keyword used for in C#?](https://blog.csdn.net/Zealot_Alie/article/details/80467872)

- [How does StartCoroutine / yield return pattern really work in Unity?](https://stackoverflow.com/questions/12932306/how-does-startcoroutine-yield-return-pattern-really-work-in-unity)
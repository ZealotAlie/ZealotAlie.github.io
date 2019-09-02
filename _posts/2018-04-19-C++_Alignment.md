---
title: "C/C++中简单数据结构的对齐"
tags: C++
---
这里只讨论简单的数据结构，从[MSDN](https://docs.microsoft.com/en-us/cpp/cpp/alignment-cpp-declarations)上的例子开始:<!--more-->
```c++
struct x_  
{  
   char a;     // 1 byte  
   int b;      // 4 bytes  
   short c;    // 2 bytes  
   char d;     // 1 byte  
};
```
在没有对齐的情况下，x_中的所有数据总计是1+4+2+1=8字节，但在实际程序中，如果使用编译器默认的对齐方式（我使用的是VS2017默认为4byte对齐），则实际上的sizeof(x_) = 12，它们在内存中的分布如下所示：
```c++
0x0000   char a; 
0x0001   [padding 3 bytes]
0x0004   int b;  
0x0008   short c;  
0x000a   char d;  
0x000b   [padding 1 byte]
```
可以看到，12bytes的内存空间中，包括了4bytes的填充。如果我们稍微变更一下四个成员变量的顺序
```c++
struct x_  
{  
   char a;     // 1 byte  
   char d;     // 1 byte  
   short c; // 2 bytes
   int b;      // 4 bytes  
};
```
重新排列之后，sizeof(x_) = 8。

CPU在读取数据时可以一个字一个字的读取，在不同的CPU上，字的大小不同。假设计算机一次读取4个字节的数据，没有对齐的情况下，可能出现一个4字节大小的数据正好位于地址3上，这样为了读取这个数据，需要分两次，第一次读取0-3，第二次读取4-8。而在保证对齐的情况下，只需要一次操作就能完全读取数据。

此外，从操作系统设计的角度上，对齐的系统要比非对齐的更加高效且较容易实现。

再看class和struct，它的对齐按照它内部成员的最大对齐来计算，对之前讨论的x_，它应该按照4bytes对齐。

可以将对齐理解为一个嵌套的过程，每个结构体所在的地址是对齐的，其内部的每个成员相对于结构体也是对齐的，则可以保证地址空间内所有的基本类型都是对齐的。
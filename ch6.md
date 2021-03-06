# 第六章 内存管理

> 作者：[Allen B. Downey](http://greenteapress.com/wp/)

> 原文：[Chapter 6  Memory management](http://greenteapress.com/thinkos/html/thinkos007.html)

> 译者：[飞龙](https://github.com/)

> 协议：[CC BY-NC-SA 4.0](http://creativecommons.org/licenses/by-nc-sa/4.0/)

C提供了4种用于动态内存分配的函数：

+ `malloc`，它接受表示字节单位的大小的整数，返回指向新分配的、（至少）为指定大小的内存块的指针。如果不能满足要求，它会返回特殊的值为`NULL`的指针。
+ `calloc`，它和`malloc`一样，除了它会清空新分配的空间。也就是说，它会设置块中所有字节为0。
+ `free`，它接受指向之前分配的内存块的指针，并会释放它。也就是说，使这块空间可用于未来的分配。
+ `realloc`，它接受指向之前分配的内存块的指针，和一个新的大小。它使用新的大小来分配内存块，将旧内存块中的数据复制到新内存块中，释放旧内存块，并返回指向新内存块的指针。

这套API是出了名的易错和苛刻。内存管理是设计大型系统中，最具有挑战性的一部分，它正是许多现代语言提供高阶内存管理特性，例如垃圾回收的原因。

## 6.1 内存错误

C的内存管理API有点像Jasper Beardly，动画片《辛普森一家》中的一个配角，他是一个严厉的代课老师，喜欢体罚别人，并使用戒尺惩罚任何违规行为。

下面是一些应受到惩罚的程序行为：

+ 如果你访问任何没有分配的内存块，就应受到惩罚。
+ 如果你释放了某个内存块之后再访问它，就应受到惩罚。
+ 如果你尝试释放一个没有分配的内存块，就应受到惩罚。
+ 如果你释放多次相同的内存块，就应受到惩罚。
+ 如果你使用没有分配或者已经释放的内存块调用`realloc`，就应受到惩罚。

这些规则听起来好像不难遵循，但是在一个大型程序中，一块内存可能由程序一部分分配，在另一个部分中使用，之后在其他部分中释放。所以一部分中的变化也需要其它部分跟着变化。

同时，同一个内存块在程序的不同部分中，也可能有许多别名或者引用。这些内存块在所有引用不再使用时，才应该被释放。正确处理这件事情通常需要细心的分析程序的所有部分，这非常困难，并且与良好的软件工程的基本原则相违背。

理论上，每个分配内存的函数都应包含内存如何释放的信息，作为接口文档的一部分。成熟的库通常做得很好，但是实际上，软件工程的实践通常不是这样理想化的。

内存错误非常难以发现，因为这些症状是不可预测的，这使得事情更加糟糕，例如：

+ 如果从未分配的内存块中读取值，系统可能会检测到错误，触发叫做“段错误”的运行时错误，并且中止程序。这个结果非常合理，因为它表示程序所读取的位置会导致错误。但是，遗憾的是，这种结果非常少见。更通常的是，程序读取了未分配的内存块，而没有检测到错误，程序所读取的未分配内存正好储存在一块特定区域中。如果这个值没有解释为正确的类型，结果可能会难以解释。例如，如果你读取字符串中的字节，将它们解释为浮点数，你可能会得到一个无效的数值，非常大或非常小的数值。如果你向函数传递它无法处理的值，结果会非常怪异。
+ 如果你向未分配的内存块中写入值，会更加糟糕。因为在值被写入之后，需要很长时间值才能被读取并且发生错误。此时寻找问题来源就会非常困难。事情还可能更加糟糕！C风格内存管理的一个最普遍的问题是，用于实现`malloc`和`free`的数据结构（我们将会看到）通常和分配的内存块储存在一起。所以如果你无意中越过动态分配块的末尾写入值，你就可能破坏了这些数据结构。系统通常直到最后才会检测到这种问题，当你调用`malloc`或`free`时，这些函数会由于一些谜之原因调用失败。

你应该从中总结出一条规律，就是安全的内存管理需要设计和规范。如果你编写了一个分配内存的库或模块，你应该同时提供释放它的接口，并且内存管理从开始就应该作为API设计的一部分。

如果你使用了分配内存的库，你应该按照规范使用API。例如，如果库提供了分配和释放储存空间的函数，你应该一起使用或都不使用它们。例如，不要在不是`malloc`分配的内存块上调用`free`。你应该避免在程序的不同部分中持有相同内存块的多个引用。

通常在安全的内存管理和性能之间有个权衡。例如，内存错误的的最普遍来源是数组的越界写入。这一问题的最显然的解决方法就是边界检查。也就是说，每次对数组的访问都应该检查下标是否越界。提供数组结构的高阶库通常会进行边界检查。但是C风格数据和大多数底层库不会这样做。

## 6.2 内存泄漏

有一种可能会也可能不会受到惩罚的内存错误。如果你分配了一块内存，并且没有释放它，就会产生“内存泄漏”。

对于一些程序，内存泄露是OK的。如果你的程序分配内存，对其执行计算，之后退出，这可能就不需要释放内存。当程序退出时，所有分配的内存都会由操作系统释放。在退出前立即释放内存似乎很负责任，但是通常很浪费时间。

但是如果一个程序运行了很长时间，并且泄露内存的话，它的内存总量会无限增长。此时会发生一些事情：

+ 某个时候，系统会耗完所有物理内存。在没有虚拟内存的系统上，下一次的`malloc`调用会失败，返回`NULL`。
+ 在带有虚拟内存的系统上，操作系统可以将其它进程的页面从内存移动到磁盘上，之后分配更多空间给泄露的进程。我会在7.8节解释这一机制。
+ 单个进程可能有内存总量的限制，超过它的话，`malloc`会返回`NULL`。
+ 最后，进程可能会用完它的虚拟地址空间（或者可用的部分）。之后，没有更多的地址可分配，`malloc`会返回`NULL`。

如果`malloc`返回了`NULL`，但是你仍旧把它当成分配的内存块进行访问，你会得到段错误。因此，在使用之前检查`malloc`的结果是个很好的习惯。一种选择是在每个`malloc`调用之后添加一个条件判断，就像这样：

```c
void *p = malloc(size);
if (p == NULL) {
    perror("malloc failed");
    exit(-1);
}
```

`perror`在`stdio.h`中声明，它会打印出关于最后发生的错误的错误信息和额外的信息。

`exit`在`stdlib.h`中声明，会使进程终止。它的参数是一个表示进程如何终止的状态码。按照惯例，状态码0表示通常终止，-1表示错误情况。有时其它状态码用于表示不同的错误情况。

错误检查的代码十分讨厌，并且使程序难以阅读。但是你可以通过将库函数的调用和错误检查包装在你自己的函数中，来解决这个问题。例如，下面是检查返回值的`malloc`包装：

```c
void *check_malloc(int size)
{
  void *p = malloc (size);
  if (p == NULL) {
    perror("malloc failed");
    exit(-1);
  }
  return p;
}
```

由于内存管理非常困难，多数大型程序，例如Web浏览器都会泄露内存。你可以使用Unix的`ps`和`top`工具来查看系统上的哪个程序占用了最多的内存。

## 6.3 实现

当进程启动时，系统为`text`段、静态分配的数据、栈和堆分配空间，堆中含有动态分配的数据。

并不是所有程序都动态分配数据，所以堆的大小可能很小，或者为0。最开始堆只含有一个空闲块。

`malloc`调用时，它会检查这个空闲块是否足够大。如果不是，它会向系统请求更多内存。做这件事的函数叫做`sbrk`，它设置“程序中断点”（program break），你可以将其看做一个指向堆底部的指针。

> 译者注：`sbrk`是Linux上的系统API，Windows上使用`HeapAlloc`和`HeapFree`来管理堆区。

`sbrk`调用时，它分配的新的物理内存页，更新进程的页表，并设置程序中断点。

理论上，程序应该直接调用`sbrk`（而不是通过`malloc`），并且自己管理堆区。但是`malloc`易于使用，并且对于大多数内存使用模式，它运行速度快并且高效利用内存。

为了实现内存管理API，多数Linux系统都使用`ptmalloc`，它基于`dlmalloc`，由Doug Lea编写。一篇描述这个实现要素的论文可在[http://gee.cs.oswego.edu/dl/html/malloc.html](http://gee.cs.oswego.edu/dl/html/malloc.html)访问。

对于程序员来说，需要注意的最重要的要素是：

+ `malloc`在运行时通常不依赖块的大小，但是可能取决于空闲块的数量。`free`通常很快，和空闲块的数量无关。因为`calloc`会清空块中的每个字节，执行时间取决于块的大小（以及空闲块的数量）。`realloc`有时很快，如果新的大小比之前更小，或者空间可用于扩展现有的内存块。否则，它需要从旧内存块中复制数据到新内存块，这种情况下，执行时间取决于旧内存块的大小。
+ 边界标签：当`malloc`分配一个快时，它在头部和尾部添加空间来储存块的信息，包括它的大小和状态（分配还是释放）。这些数据位叫做“边界标签”。使用这些标签，`malloc`就可以从任何块移动到内存中上一个或下一个块。此外，空闲块会链接到一个双向链表中，所以每个空闲块也包含指向“空闲链表”中下一个块和上一个块的指针。边界标签和空闲链表指针构成了`malloc`的内部数据结构。这些数据结构穿插在程序的数据中，所以程序错误很容易破坏它们。
+ 空间开销：边界标签和空闲链表指针也占据空间。最小的内存块大小在大多数系统上是16字节。所以对于非常小的内存块，`malloc`在空间上并不高效。如果你的程序需要大量的小型数据结构，将它们分配在数组中可能更高效一些。
+ 碎片：如果你以多种大小分配和释放块，堆区就会变得碎片化。也就是说，空闲空间会打碎成许多小型片段。碎片非常浪费空间，它也会通过使缓存效率低下来降低程序的速度。
+ 装箱和缓存：空闲链表在箱子中以大小排序，所以当`malloc`搜索特定大小的内存块时，它知道应该在哪个箱子中寻找。所以如果你释放了一块内存，之后立即以相同大小分配一块内存，`malloc`通常会很快。

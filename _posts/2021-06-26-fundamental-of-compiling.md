---
title: 编译原理概述
author: 但江
avatar: danjiang
location: 深圳
category: programming
tag: cs
---

本文是对编译原理的概述，便于掌握基本概念，提供进一步深入的索引。

![LLVM Logo](/images/llvm-logo.png)

## 前端

如下深灰色部分为前端：

![Compile AOT](/images/compile-aot.jpg)

### 词法分析

> 词法分析是把程序分割成一个个 Token 的过程。

#### 算法

* 确定的有限自动机（Deterministic Finite Automaton，DFA）。
* 非确定的有限自动机（Nondeterministic Finite Automaton，NFA）。

### 语法分析

> 语法分析是把程序的结构识别出来，并形成一棵便于由计算机处理的抽象语法树。

#### 抽象语法树 AST

如下是 **2 + 3 * 5** 的语法分析，Num、+ 和 * 都是终结符，终结符都是词法分析中产生的 Token，而那些非叶子节点，就是非终结符，文法的推导过程，就是把非终结符不断替换的过程，让最后的结果没有非终结符，只有终结符：

![AST](/images/ast.png)

#### BNF

语法规则常写成如下的巴科斯范式，简称 BNF：

{% highlight text %}
add ::= mul | add + mul
mul ::= pri | mul * pri
pri ::= Id | Num | (add) 
{% endhighlight %}

#### EBNF

语法规则常写成如下的扩展巴科斯范式，简称 EBNF，里面会用到类似正则表达式的一些写法：

{% highlight text %}
add -> mul (+ mul)*
{% endhighlight %}

#### 算法

* 自顶向下：递归下降。
* 自顶向下：LL 算法，针对深度优先算法，给算法加上预测能力避免回溯。
* 自底向上：LR 算法，通过移进 - 规约方法，自底向上地构造 AST。

#### 工具

* **Antlr** 是一个开源的工具，支持根据规则文件生成词法分析器和语法分析器，它自身是用 Java 实现的。
* 还有 Lex（Flex）、Yacc（Bison）。
* Visitor 设计模式针对每一种 AST 节点，都会有一个单独的方法来负责处理，能够让代码更清晰，也更便于维护。 

### 语义分析

> 语义分析的本质是对上下文相关情况的处理，能做词法分析和语法分析所做不到的事情。比如 Java 语言和 JavaScript 在代码块的语法上是一样的，都是用花括号，但在语义上是不同的，一个有块作用域，一个没有。

#### 属性计算

* 属性计算是基于语法规则，增加一些与语义处理有关的规则，会作为属性标注在抽象语法树上。
* 属性计算，可以伴随着语法分析的过程一起进行，也可以在做完语法分析以后再进行，这两个阶段不一定完全切分开。
* I 属性（Inherited Attribute）：也就是继承属性，即 AST 中某个节点的属性是由上级节点、兄弟节点和它自身来决定的。
* S 属性（Synthesized Attribute）：如果一种属性能够从下级节点推导出来，那么这种属性就叫做 S 属性。
* L 属性：如果某个属性的计算，除了可能依赖子节点以外，只依赖左边的兄弟节点，不依赖右边的，这种属性就叫做 L 属性。

#### 工具

语义分析通常需要自己编写，可以实现如下功能：

* 块作用域和函数
* 面向对象的类，实现数据和方法的封装
* 闭包
* 类型系统
* 面向对象运行期的继承和多态

## 后端

如下浅灰色部分为后端：

![Compile AOT](/images/compile-aot.jpg)

### 程序运行机制

* **编译执行**：需要编译成机器码的执行方式。
	* **提前编译（AOT）**：编译成可执行文件后，再执行。
	* **即时编译（JIT）**：在需要运行某段代码的时候，再去编译。
* **解释执行**：不需要编译成机器码的执行方式。

### 链接、装载与库

#### 符号表

**符号**（Symbol）用来表示一个地址，这个地址可能是一个函数的起始地址，也可以是一个变量的起始地址。

每一个目标文件都会有一个相应的**符号表**（Symbol Table），这个表里面记录了目标文件中所用到的所有符号，每个定义的符号有一个对应的值，叫做符号值（Symbol Value），对于变量和函数来说，符号值就是它们的地址。

#### 静态链接

在链接中，目标文件之间相互拼合实际上是**目标文件之间对函数和变量的地址的引用**：

![Static Link](/images/static-link.jpg)

合并目标文件：

![Static Link Process](/images/static-link-process.png)

#### 装载

装载就是操作系统**将可执行文件加载到进程的地址空间**，从而才可以执行程序。

![Load Executable File](/images/load-executable-file.png)

#### 动态链接

如果 **foobar()** 是一个定义在某个动态共享对象中的函数，那么链接器就会**将这个符号的引用标记为一个动态链接的符号**，不对它进行地址重定位，把这个过程留到装载时再进行：

![Dynamic Link](/images/dynamic-link.jpg)

**动态链接器**与普通共享对象一样被映射到了进程的地址空间，在系统开始运行程序之前，首先会把控制权交给动态链接器，由它完成所有的动态链接工作以后再把控制权交给程序，然后开始执行。

### 计算机组成和程序运行环境

程序运行的过程中，主要是跟两个硬件（CPU 和内存）以及一个软件（操作系统）打交道。

![Computer Components](/images/computer-components.png)

#### CPU

* **寄存器**是 CPU 指令在进行计算的时候，临时数据存储的地方。
* 计算机的存储是分成多个级别的：
	* 速度最快的是**寄存器**，通常在寄存器之间复制数据只需要 1 个时钟周期。
	* 其次是**高速缓存**，它根据速度和容量分为多个层级，读取所花费的时间从几个时钟周期到几十个时钟周期不等。
	* **内存**则要用上百到几百个时钟周期。
* **SIMD**（Single Instruction Multiple Data）的指令，单条指令能处理多个数据。

#### 内存

* 程序在使用内存时，会把内存划分为不同的区域，这就是**内存管理模型**，以 C 语言为例，会把内存划分为代码区、静态数据区、栈和堆。
* 程序在运行时顺序执行代码，可以根据跳转指令来跳转；栈被划分成**栈桢**，栈桢的设计有一定的自由度，但通常也要遵守一些约定；栈桢的大小和结构在编译时就能决定；在运行时，栈桢作为活动记录，不停地被动态创建和释放。

#### 操作系统

* 操作系统装载可执行文件，然后运行程序：
	* 操作系统在创建进程后，把控制权交到了程序的入口，这个入口往往是运行库中的某个**入口函数**。
	* 入口函数对运行库和程序运行环境进行初始化，包括堆、I/O、线程、全局变量构造，等等。
	* 入口函数在完成初始化之后，调用 **main 函数**，正式开始执行程序主体部分。
	* main 函数执行完毕以后，返回到入口函数，入口函数进行清理工作，包括全局变量析构、堆销毁、关闭 I/O 等，然后进行系统调用结束进程。
* **运行库**是操作系统之上的抽象层，如 C 语言的运行库从某种程度上来讲是 C 语言的程序和不同操作系统平台之间的抽象层，它将不同的操作系统 API 抽象成相同的库函数。
* **系统调用**是应用程序（运行库这里也看作是应用程序的一部分）与操作系统内核之间的接口，它决定了应用程序是如何与内核打交道的，无论程序是直接进行系统调用，还是通过运行库，最终还是会到达系统调用这个层面上。

### 中间代码

* **IR 的意思是中间表达方式（Intermediate Representation）**，它在高级语言和汇编语言的中间，这意味着，它的特征也是处于二者之间的。
* 与高级语言相比，**IR 丢弃了大部分高级语言的语法特征和语义特征**，比如循环语句、if 语句、作用域、面向对象等等，它更像高层次的汇编语言。
* 而相比真正的汇编语言，**它又不会有那么多琐碎的、与具体硬件相关的细节**。

#### 三地址代码（Three Address Code, TAC）

适合用来讨论算法，每条三地址代码最多有三个地址，其中两个是源地址，一个是目的地址，每条代码最多有一个操作。

C 语言的算术运算：

{% highlight c %}
int a, b, c, d;
a = b + c * d;
{% endhighlight %}

TAC：

{% highlight text %}
t1 := c * d
a  := b + t1
{% endhighlight %}

#### LLVM 汇编码（LLVM Assembly）

**LLVM 汇编码是 LLVM 的 IR**，LLVM 的 IR 有两种格式，第一种是**文本格式**，文件名以 .ll 结尾，还有第二种格式是**字节码格式**，文件名以 .bc 结尾。

C 语言代码：

{% highlight c %}
int fun1(int a, int b){
  int c = 10;
  return a + b + c;
}
{% endhighlight %}

LLVM IR 文本格式：

{% highlight text %}
; ModuleID = 'function-call1.c'
source_filename = "function-call1.c"
target datalayout = "e-m:o-i64:64-f80:128-n8:16:32:64-S128"
target triple = "x86_64-apple-macosx10.14.0"

; Function Attrs: noinline nounwind optnone ssp uwtable
define i32 @fun1(i32, i32) #0 {
  %3 = alloca i32, align 4
  %4 = alloca i32, align 4
  %5 = alloca i32, align 4
  store i32 %0, i32* %3, align 4
  store i32 %1, i32* %4, align 4
  store i32 10, i32* %5, align 4
  %6 = load i32, i32* %3, align 4
  %7 = load i32, i32* %4, align 4
  %8 = add nsw i32 %6, %7
  %9 = load i32, i32* %5, align 4
  %10 = add nsw i32 %8, %9
  ret i32 %10
}

attributes #0 = { noinline nounwind optnone ssp uwtable "correctly-rounded-divide-sqrt-fp-math"="false" "disable-tail-calls"="false" "less-precise-fpmad"="false" "min-legal-vector-width"="0" "no-frame-pointer-elim"="true" "no-frame-pointer-elim-non-leaf" "no-infs-fp-math"="false" "no-jump-tables"="false" "no-nans-fp-math"="false" "no-signed-zeros-fp-math"="false" "no-trapping-math"="false" "stack-protector-buffer-size"="8" "target-cpu"="penryn" "target-features"="+cx16,+fxsr,+mmx,+sahf,+sse,+sse2,+sse3,+sse4.1,+ssse3,+x87" "unsafe-fp-math"="false" "use-soft-float"="false" }

!llvm.module.flags = !{!0, !1}
!llvm.ident = !{!2}

!0 = !{i32 1, !"wchar_size", i32 4}
!1 = !{i32 7, !"PIC Level", i32 2}
!2 = !{!"clang version 8.0.0 (tags/RELEASE_800/final)"}
{% endhighlight %}

**LLVM IR 的对象模型**：LLVM 在内部有用 C++ 实现的对象模型，能够完整表示 LLVM IR，当我们把字节码读入内存时，LLVM 就会在内存中构建出这个模型，也可以通过 API 来生成这些对象，只有基于这个对象模型，我们才可以做进一步的工作，包括代码优化，实现即时编译和运行，以及静态编译生成目标文件。所以说，这个对象模型是 LLVM 运行时的核心。

#### 字节码

字节码也是一种 IR，后面会专门讨论。

#### 工具

* **GCC**（GNU Compiler Collection，GNU 编译器套件）
* **C++**
* **CMake**
* **Clion**

### 代码优化

AST 抽象层次太高，含有的硬件架构信息太少，难以执行很多优化算法。在汇编代码上进行优化会让算法跟机器相关，当换一个目标机器的时候，还要重新编写优化代码。所以，**在 IR 上是最合适的，它能尽量做到机器独立，同时又暴露出很多的优化机会**。

代码优化按范围可分为：

* 本地优化
* 全局优化
* 过程间优化

代码优化还可分为：
	
* 独立于机器的优化
* 依赖于机器的优化：依赖于硬件的特征。
	* **SIMD** 是一种指令级并行技术，它能够矢量化地一次计算多条数据，从而提升计算性能。
	* 充分保持**程序的局部性**，能够更好地利用计算机的**高速缓存**，从而提高程序的性能。

#### 工具

代码优化不是只有针对 IR 阶段的优化，LLVM 在设计上支持**全过程的优化**，在 [LLVM: A Compilation Framework for Lifelong Program Analysis & Transformation](https://llvm.org/pubs/2003-09-30-LifelongOptimizationTR.pdf) 就提出计算机语言可以在各个阶段进行优化，包括编译时、链接时、安装时，甚至是**运行时**。

在 LLVM 内部，优化工作是通过一个个的 **Pass** 来实现的，它支持三种类型的 Pass：

* 一种是分析型的 Pass，只是做分析，产生一些分析结果用于后序操作。
* 一些是做代码转换的 Pass，比如做公共子表达式删除。
* 还有一类 Pass 是工具型的，比如对模块做正确性验证。

#### 算法

* 可用表达式分析
* 活跃性分析
* 数据流分析
* 半格理论
* 树覆盖算法：Maximal Munch 算法、动态规划算法、树文法等
* 图染色算法

### 生成汇编代码

对于静态编译型语言，比如 C 语言和 Go 语言，编译器后端的任务就是**生成汇编代码**，然后再由**汇编器生成机器码**，生成的文件叫目标文件，最后再使用**链接器**就能**生成可执行文件或库文件**了：

![Compile Assembly](/images/compile-assembly.jpg)

#### 汇编代码

* **汇编语言**是由指令、标签、伪指令和注释构成的。其中主要内容都是指令，**指令包含一个该指令的助记符和操作数**。操作数可以使用直接数、寄存器，以及用两种方式访问内存地址。
* 计算机的处理器有很多不同的架构，比如 x86-64、ARM、Power 等，每种**处理器的指令集都不相同**，那也就意味着**汇编语言不同**。

#### 工具

**GNU 汇编器**，macOS 和 Linux 已内置，Windows 系统可安装 MinGW 或 Linux 虚拟机。

#### 优化

汇编代码生成过程中有三个关键：

* **指令选择**：同样一个功能，可以用不同的指令或指令序列来完成。
* **寄存器分配**：每款 CPU 的寄存器都是有限的。
* **指令重排序**：计算执行的次序会影响所生成的代码的效率。

**指令级并行**：因为 CPU 内部存在着多个功能单元，所以在同一时刻，不同的功能单元其实可以服务于不同的指令，这样的话，多条指令实质上是并行执行的，从而减少了总的执行时间。

### 字节码

**字节码生成器**可以将高级语言编译成字节码，**字节码**能够在**虚拟机**上解释执行，或即时编译执行：

![Compile Bytecode](/images/compile-bytecode.jpg)

字节码生成技术可以**向原来的代码中注入新代码**，来实现对性能的监测等功能。

#### 虚拟机

虚拟机的两种技术：

* **基于栈的虚拟机**：不用显式地管理操作数的地址，因此指令会比较短，指令生成也比较容易。
* **基于寄存器的虚拟机**：能更好地利用寄存器资源，也能对代码进行更多的优化。

虚拟机示例：

* **JVM**（Java 虚拟机）：Groovy、Kotlin、Closure、Scala 等很多语言
* Google 公司为 Android 开发的 **Dalvik** 虚拟机和 Lua 语言的虚拟机。

#### 工具

生成字节码的工具：

* **ASM**
* Apache BCEL
* Javassist

## 发展趋势

### Flutter

首先，**Dart** 语言也是基于**虚拟机**的，编译方式支持 **AOT** 和 **JIT**，能够运行在**移动端和桌面端**，能够调用本地操作系统的功能。对于 **Web** 应用则编译成 JavaScript、CSS 和 HTML。**用一门新的语言去整合多个技术栈**。

### Web Assembly（WASM）

**WASM** 是一种二进制的**字节码**，也就是一种新的 IR，能够在**浏览器**里运行。

有以下特点：

* 静态类型；
* 性能更高；
* 支持 **C/C++/Rust** 等各种语言生成 WASM，**LLVM** 也给了 WASM 很好的支持；
* 字节码尺寸比较少，减少了下载时间；
* 因为提前编译成字节码，因此相比 JavaScript 减少了代码解析的时间。

### 元编程

元编程是指用程序操纵程序的能力，也就是用程序修改或者生成程序。也有人用另外的表述方式，认为具有元编程能力的语言，能够把程序当做数据来处理。

两个级别：

* 如果一门语言写的程序，能够报告它自身的信息，这叫做**自省**（introspection）。
* 如果能够再进一步，操纵它自身，那就更高级一些，这叫做**反射**（reflection）。

## 参考

* [程序员的自我修养：链接、装载与库](https://book.douban.com/subject/3652388/)
* [极客时间 - 编译原理之美](https://time.geekbang.org/column/intro/100034101)
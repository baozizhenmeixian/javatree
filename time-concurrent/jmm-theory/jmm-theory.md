Java内存模型（JMM）描述的是**一组规则**，通过这组规则控制程序中各个变量在共享数据区域和私有数据区域的访问方式。从JDK5开始，由[JSR-133](https://download.oracle.com/otn-pub/jcp/memory_model-1.0-pfd-spec-oth-JSpec/memory_model-1_0-pfd-spec.pdf)详细规定。**它并不是描述多线程程序应该怎么执行，而是描述多线程程序允许怎么展示**。

> JSR是JavaSpecification Requests的缩写，意思是“Java 规范提案”。是指向JCP(JavaCommunity Process)提出新增一个标准化技术规范的正式请求。
> JSR -133全称：Java Memory Model and Thread Specification Revision（ Java 内存模型与线程规范修订版）。

JMM的具体规范可以在JSR-133找到详细定义，但是是通过一系列符号和集合给出的理论定义。为了实际理解JMM，我们先来说清楚更简单的顺序一致性模型和 happens-before 模型，再以这两个模型类比理解JMM。

### 2.2.1 顺序一致性模型

顺序一致性内存模型是一个被计算机科学家**理想化**了的理论参考模型，像一个基准点。在设计的时候，处理器级别的内存模型和编程语言级别的内存模型都会以顺序一致性模型作为参照。

> 顺序一致性模型的定义：
> 在顺序一致性里，所有动作以全序（执行顺序）的顺序发生，与程序顺序一致；而且，每个对变量 v 的读操作 r 都将看到写操作 w 写入 v 的值，只要:
>
> - 执行顺序上 w 在 r 之前，且
> - 执行顺序上不存在这样一个 w' ，w 在 w' 之前且 w' 在 r 之前。

我们来更直白地理解。顺序一致性内存模型的特征：

- 所有动作存在一个全序关系，与程序的顺序一致；
- 每个动作都是原子的；
- 每个动作执行后立即对所有线程可见；

其实就是严格遵守了顺序性、原子性、可见性。

可以理解为顺序一致性模型有一个单一的全局内存，这个内存在任一时刻只能够连接到一个线程，同时每一个线程必须按照程序的顺序来执行读写内存的操作（是不是忽然想到了MySQL的串行化，这个后文来做横向概念类比）。

之所以说顺序一致性模型是一个理想模型，是因为它虽然能够保证正确，但是效率太低了。如果将顺序一致性作为实际内存模型，之前讨论的编译器和处理器优化将不再合法。现实中的处理器和语言都是在顺序一致性模型的基础上，为了执行效率优化，做了不同程度的执行顺序牺牲。

具体到JMM，对于正确同步（按照JMM同步规则进行代码编写）的程序，会保证程序的执行结果与该程序在顺序一致性内存模型中的执行结果相同。可以说这对于开发者来讲是一个极强的保证。

对于未正确同步的程序，整体执行上是无需的，其执行结果是无法预知的。包括不保证对64位的 long 型和 double 型变量的写操作具有原子性。JMM仅能够保证最小安全性，即线程读到的值要么是之前某个线程写入的，要么是默认值（0 或 null 或 false），不会无中生有地出现。为了实现这个最小安全性，JMM会在堆上分配对象前对该内存空间初始化。

### 2.2.2 happens-before 模型

#### 2.2.2.1 happens-before 关系的定义

happens-before 规则在JSR-133中是通过 synchronized-with 边缘来辅助定义的。许多资料上将其总结为6条规则，实际上JSR-133中有更多条。

首先来说 synchronized-with 边缘的规则定义（引号里的名字都是为了方便阅读我自己起的，JSR-133并没有这样的表述）：

1. “管程锁定规则”：某个管程m上的解锁动作 synchronizes-with 所有后续在m上的锁定动作；
2. “volatile变量规则”：对volatile变量v的写操作 synchronizes-with 所有后续任意线程对v的读操作；
3. “线程启动规则”：用于启动一个线程的动作 synchronizes-with 该新启动线程中的第一个动作；
4. “线程终止规则”：线程T1的最后一个动作 synchronizes-with 线程T2中任一用于探测T1是否终止的动作（T2 可能通过调用 T1.isAlive() 或者在 T1 上执行一个 join 动作来达到这个目的。）；
5. “线程中断规则”：如果线程T1中断了线程T2，T1的中断操作 synchronizes-with 任意时刻任何其它线程（包括 T2）用于确定 T2 是否被中断的操作；
6. “默认值规则”：为每个变量写默认值（0，false或null）的动作 synchronizes-with 每个线程中的第一个动作；
7. “对象终结规则”：调用对象的终结方法时，会隐式的读取该对象的引用。从一个对象的构造器末尾到该引用的读取之间存在一个 synchronizes-with 边缘；

其次，在 happens-before 规则的定义中，除了synchronized-with 边缘的规则还添加了最基本的程序顺序规则和传递性规则：

8. “程序顺序规则”：如果 x 和 y 是 同一个线程中的动作，且在程序顺序上 x 在 y 之前，那么有 x  happens-before y；
9. “传递性规则”：happens-before 是传递闭包的。换而言之，如果 x happens-before y，且 y happens-before z，那么可以得到 x happens-before z。

#### 2.2.2.2 happens-before 关系的作用

两个动作（action）可以被 happens-before 关系排序。如果一个动作 happens-before 另一个动作，则第一个对第二个可见，且第一个排在第二个之前。必须强调的是，**两个动作之间存在 happens-before 关系并不意味着这些动作在 Java 中必须以这种顺序发生**，即不改变最终结果的重排序是合法的。happens-before 关系主要用于强调两个有冲突的动作之间的顺序，以及定义数据争用的发生时机。

> happens-before 关系本质上和 as-if-serial 语义是一回事。
> as-if-serial 语义保证单线程内程序的执行结果不被改变，happens-before 关系保证多线程程序的执行结果不被改变。它们的目的都是为了在不改变程序执行结果的前提下，尽可能提高程序执行的并行度。

#### 2.2.2.3 happens-before 内存模型和 JMM 的区别——因果关系

这一节比较重要，也比较难以理解。

对于JMM来说， happens-before 关系有一些薄弱，不能满足正确顺序的需求。JMM在其基础上加上了因果关系，即**一个写操作不能发生在一个其依赖的读操作之前**，因为它涉及写操作是否会触发自身发生的问题。

举一个JSR -133给出的官方示例：

![image.png](https://upload-images.jianshu.io/upload_images/9341275-df0ef0161813e0d1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

按照JMM，r1 == r2 == 0 是唯一合法结果。但是在 happens-before 内存模型下，存在执行结果是 r1 == r2 == 1 的可能性，因为程序中没有 synchronizes-with 或 happens-before 边缘，允许每个读操作看到其它线程写的值，单个线程内的执行顺序也不受约束。想得到上述结果的执行顺序是：

```
r1 = x; // 已经看到了x = 1的写操作结果
y = 1;
r2 = y; // 已经看到了y = 1的写操作结果
x = 1;
```

我们会感到奇怪，执行`x = 1`操作之前，为什么会已经看到了x = 1的写操作结果。注意，happens-before 包括JMM都只是一种规范理论，而不去管能不能实现或者怎么样实现（真实情况可能由于多级缓存可能由于重排序也可能先执行再决定是否采纳，这些都是实现层面的事情）。所以在此处，由于遵守了 happens-before 规则的约定（可能会对程序顺序规则疑问，但是上述执行顺序已经遵守了单线程中的程序顺序规则），所以此顺序的任意读操作可以看到任意写操作的结果，也就看到了`x = 1`的写操作结果。总的来说，要从一个内存模型而不是内存模型实现的角度去理解它。

这种执行顺序是 happens-before 允许的，但JMM不允许，也就是我们上述的因果关系。所以，**happens-before 模型可以理解为JMM的真子集，它们的差别就是因果关系（Causality）**。

在此也摘录JSR-133 中 7.1节 和 7.2 节的原文，供大家原汁原味地理解：

> Informally, a read r is allowed to see the result of a write w if there is no happens-before ordering to prevent that read.
> In this execution, the reads see writes that occur later in the execution order. This may seem counterintuitive, but is allowed by happens-before consistency. It turns out that allowing reads to see later writes can sometimes produce unacceptable behaviors.
> Happens-Before Consistency is a necessary, but not sufficient, set of constraints. Merely enforcing Happens-Before Consistency would allow for unacceptable behaviors – those that violate the requirements we have established for Java programs. For example, happens-before consistency allows values to appear “out of thin air”.
> This result is happens-before consistent: there is no happens-before relationship that prevents it from occurring. However, it is clearly not acceptable: there is no sequentially consistent execution that would result in this behavior. The fact that we allow a read to see a write that comes later in the execution order can sometimes thus result in unacceptable behaviors.

对因果关系的理解

实际上因果关系就是要解决什么时候一个顺序靠前的读操作被允许看到顺序靠后的写操作的执行结果。

JSR-133中给出了规定（在 happens-before 基础上）：

- 如果一个写操作无论在何种排序中都会被执行，则可以被较早看到；
- 如果一个写操作依赖于先前的某些读操作才会被执行，则不可以被较早看到。

原文帮助理解：

>Examples such as these reveal that the specification must define carefully whether a read can see a write that occurs later in the execution (bearing in mind that if a read sees a write that occurs later in the execution, it represents the fact that the write is actually performed early).
>The rules that allow a read to see a later write are complicated and can be controversial, in part because they rely on judgments about acceptability that are not covered by traditional program semantics. There are, however, two uncontroversial properties that are easy to specify:
>• If a write is absolutely certain to be performed in all executions, it may be seen early.
>• If a write cannot occur unless it is seen by an earlier read, it cannot be seen by an earlier read.

### 2.2.3 Java内存模型（JMM）

#### 2.2.3.1 JMM的正式理论定义

相信我你不会想看这一节的，很难理解，但是还是要简略地陈述出来。不想看的可以直接跳过看2.2.3.2。

```
1. 首先定义动作与执行过程
1.1 动作 a 是通过元组<t, k, v, u>来描述的，其中:
  - t - 执行该动作的线程
  - k - 动作的类型：volatile read，volatile write，(非 volatile)read，(非 volatile) 写，lock，unlock，特殊的同步动作，外部动作以及线程分散动作
  - v - 动作中涉及到的变量或管程
  - u - 该动作的任意一个唯一标识符
1.2 执行过程 E 通过元组< P, A, --po-->, --so-->, W, V, --sw-->, --hb--> >描述，其中：
- P- 一个程序
- A- 一组动作
- --po--> - 程序顺序，对每个线程 t 来说，程序顺序是 A 中由 t 执行的所有动作上的全序
关系。
- --so--> - 同步顺序，是 A 中所有同步动作上的全序关系。
- W - 一个写可见(write-seen)函数，对于 A 中的每个读操作 r，W(r)表示 E 中对 r 可见的写操作。
- V - 一个值写入(value-written)函数，对于 A 中的每个写操作 w，V(w)表示 E 中 w 写入的值。
-  --sw--> - synchronizes-with，同步动作上的偏序关系。
- --hb--> - happens-before，动作上的偏序关系。

2. 其次定义一个良构的执行过程
当下列条件为 true 时，执行过程 E = < P, A, --po-->, --so-->, W, V, --sw-->, --hb--> > 就是良构的:
- 每个对变量 x 的读都能看到一个对 x 的写。所有对 volatile 变量的读写都是 volatile 动作；
- 同步顺序与程序顺序以及互斥是一致的。意味着 happens-before 顺序是一种合法的偏序关系：自反的、传递的以及反对称的。同时在每个管程上，lock 和 unlock 动作是正确嵌套的；
- 线程的运行遵守线程内(intra-thread)一致性；
- 线程的运行遵守同步顺序一致性；
- 线程的运行遵守 happens-before 一致性；
- 线程的运行对同步操作遵守弱公平性属性；

3. 一个良构的执行过程就是JMM允许的过程。
```

#### 2.2.3.2 JMM定义的理解

综合前几节的内容，我们可以将JMM从两个视角理解。

- 从Java开发者的视角，希望内存模型易于理解、易于编程，所以希望基于一个强内存模型来编写代码。JMM向开发者提供保证，如果程序是正确同步的（满足A happens-before B 和因果关系），那么A的操作结果一定对B可见，且A的执行顺序排在B之前（注意这只是对开发者的保证，是从程序员视角出发的）。

- 从编译器和处理器的视角，希望内存模型对其优化的束缚越小越好，这样就可以尽可能提高性能，所以希望实现一个弱内存模型。所以JMM遵循一个基本原则：只要不改变程序的执行结果（单线程程序和正确同步的多线程程序），编译器和处理器可以随意优化。

即JMM为程序员创造了一个黑盒，**程序员大可以理解为：只要代码正确同步了，就会是按照顺序一致性指定的顺序来执行的**。但是黑盒中会有各种重排序等优化，只要不影响最终可见性和结果就没问题。
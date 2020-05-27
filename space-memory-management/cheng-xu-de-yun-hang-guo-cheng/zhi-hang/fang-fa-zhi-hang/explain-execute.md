# 解释执行

[toc]

Java 源程序经编译后称为字节码，由运行时环境对字节码进行解释执行。提供解释功能的JVM组件称为解释器

**解释执行**可以与**CPU的处理过程**进行类比，CPU就是在重复进行着“**取指-译码-执行**“这一操作，并在这个过程中完成了一条条指令的执行，实现了程序规定的任务，**解释器**就类似于CPU这个角色，也在不断的“取指-译码-执行”，下文将介绍HopSpot VM解释器的分类与组成，并详细介绍解释器的运作流程

## 1、解释器的分类与组成

从译码的角度，可以将HotSpot实现解释器分为**C++解释器**与**模版解释器**两种，顾名思义，C++解释器是将**字节码**“译码“成C++代码，这其实很好理解，毕竟HotSpot本身就是用C++实现的；而模版解释器则是将**字节码**“译码“成了机器码，这也是目前HotSpot目前的默认解释器

### 1.1 C++解释器

对于C++解释器而言，是将字节码译码成C++代码，那么在执行性能上应该是不如译码成机器码的

比如在寄存器分配上时，手写汇编在这种很小范围很细节的地方能有效的、精确的控制寄存器的使用，保证不会浪费寄存器

而C++编译器不一定能完美的选择到最好的寄存器分配方式。在比较大规模的代码里编译器可能比手写做得好，因为人能在脑子管理好的复杂度也是有限的；但小规模代码的精确控制通常还是人做得好些。

那按理来说，模版解释器应该是C++解释器的升级版(**很多博客和书籍中都是这么描述的**)，但是并不是，在@RednaxelaFX的豆瓣笔记中翻到了这样一段关于这两类解释器由来的历史，还挺有意思的

> 其实HotSpot从一开始就有模板解释器，而C++解释器反而是后来加进来的。前者源于HotSpot的前身Strongtalk，而后者源于Sun的另一个JVM(CVM)，又名“CDC HotSpot Implementation”或者“CDC-HI”。CVM更早的前身是Sun的Classic VM。 也就是说，这俩解释器没有任何血缘关系，前者并非将后者翻译为汇编。当时Sun之所以把后者加到HotSpot是在实现Itanium（IA-64）的移植时想偷懒，人肉写Itanium汇编挺烦的所以要实现Itanium版模板解释器不方便，他们就想到了把CVM的解释器移植过来，这样可以少写点汇编就能完成移植。结果一直以来HotSpot能运行的平台上只有Itanium版是真的用了这个C++解释器的，而其它平台上默认都在用模板解释器，但还是可以通过编译参数选择使用C++解释器。```

### 1.2 模版解释器

模板解释器相对于为每一个**字节码指令**都写了一段实现对应功能的汇编代码，在**JVM初始化**时，汇编器会将汇编代码翻译成机器指令加载到内存中，比如执行iload指令时，直接执行对应的汇编代码即可。如何执行汇编代码？直接跳往汇编代码生成的机器指令在内存中的地址即可。

#### 1.2.1 模版解释器的组成

##### 1.2.1.1 模版表

JVM初始化时会为每个**字节码指令**都创建一个模板，每个模板都关联其对应的**汇编代码生成函数**，所有字节码的模版组合在一起，构成一个模版表。表中每个元素都是一个模版，元素按照字节码值的递增顺序排列，第n号元素表示的就是字节码为n对应的模版

> 我们平时说的iload指令等，其实都只是字节码指令的助记符，帮助我们理解，真正的字节码码指令其实就是一个数字，比如iload是21，虚拟机执行21这个指令时，就是执行iload

以下这段代码就是模版的**汇编代码生成函数**

``` c++
void Template::generate(InterpreterMacroAssembler* masm) {
  // parameter passing
  TemplateTable::_desc = this;
  TemplateTable::_masm = masm;
  // code generation
  _gen(_arg);
  masm->flush();
}
```

##### 1.2.1.1 机器码生成器

在JVM启动时，代码生成器将统一为字节码以及JVM内部例程生成机器码，对于字节码来说，就是在模版表里找到字节码对应的模版，然后根据汇编代码生成函数生成代码，具体到源码层面，机器码生成器就是个C++类`TemplateInterpreterGenerator`,通过它的`generate_all()`函数生成代码

在JVM中，所有由代码生成器生成的代码都由一个Codelet来表示，面向解释器的Codelet称为InterpreterCodelet，通过这些Codelet来完成在JVM内部存储、定位和执行代码的任务

每个InterpreterCodelet具有名称和字节码编号，并能够找到机器码在JVM内存中的起、止地址

```c++
class InterpreterCodelet: public Stub {
 private:
  int         _size;                             // the size in bytes
  Bytecodes::Code _bytecode;                     // associated bytecode if any
 public:
  // Code info
  address code_begin() const                     { return (address)this + round_to(sizeof(InterpreterCodelet), CodeEntryAlignment); }
  address code_end() const                       { return (address)this + size(); }

```

为了方便管理，在解释器中，每个InterpreterCodelet并不是孤立的，它们共同构成了一个**Stub队列**

>刚开始学这块时，总是分不清Stub是个什么东西，明明队列里放的是Codelet还要叫Stub队列（博客和书籍里），后面看了上面那段源码，原来Stub就是个父类，此外，Stub有个意思是“一小块代码”，通常是有个caller要调用callee的时候，中间需要一些特殊处理的逻辑，就会用这种“小块代码”去做，它们并不是最终的调用目标，而是做一些简单的处理之后“跳”到真正的目标去。

![](https://upload-images.jianshu.io/upload_images/18927149-c91985c2dec1fd0d.png)

##### 1.2.1.2 字节码表



##### 1.2.1.1 转发表



#### 1.2.2 模版解释器的运作过程



![](../../../../.gitbook/assets/image%20%281%29.png)


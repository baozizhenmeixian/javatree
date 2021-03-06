## 1、虚拟机基础

要理解"虚拟机"，首先要讨论“机器“的含义，“机器“的含义与讨论时所站的角度有关。

从执行用户程序的**进程角度**看，“机器“由分配给进程的逻辑内存地址空间、用户级寄存器和允许执行进程代码的指令组成。机器的I/O部分只能通过操作系统看到，进程与I/O系统交互的唯一方法是通过操作系统调用，通常是将系统库函数作为进程的一部分来执行。所以从进程的角度看，机器由**操作系统**与底层的**用户级硬件**组合而成

从**操作系统**的角度看，整个系统由底层机器支撑。一个系统是一个完整的运行环境，可以同时支持多个不同的用户进程，所有进程共享一个文件系统和其他的I/O资源。系统为进程分配物理内存和I/O资源，允许进程通过操作系统与分配给它们的资源交互。因此，从系统的角度看，机器仅由**底层硬件实现**

![image (42)](../img/vm_classify.png)

对机器可以从进程和系统角度去理解，相应地也就有**进程级**和**系统级**虚拟机

### 1.1 进程虚拟机

顾名思义，一台进程虚拟机能够支持一个独立的进程，虚拟软件同时仿真**用户级指令**和**操作系统调用**，即上图左边机器与应用程序的边界，下图是进程虚拟机的示例

<img src="../img/image (35).png" style="zoom:67%;" />

术语上，我们通常将底层平台称为**主机**，将运行在虚拟机环境上的软件称为**客户机**，虚拟软件经常被称为**运行时**（runtime software），运行时用户支持客户进程，运行在操作系统的上层。虚拟机为客户进程的执行和终止提供相应的支持，**高级语言虚拟机\(如jvm\)**就是进程虚拟机的比较特殊的一种

### 1.2 系统虚拟机

与进程虚拟机大不相同，**系统虚拟机**提供完成的系统环境，这个环境可以支持操作系统及现在的许多用户进程，它使客户操作系统能够访问底层的硬件资源，包括网络、I/O以及图形用户界面

系统虚拟机如下图所示，虚拟软件（也被称为虚拟机监视器，VMM）被放在底层硬件机器和传统软件之间，虚拟软件将一个硬件平台上的指令集翻译成另一个，以构成系统虚拟机，能够执行为不同硬件集开发的系统软件环境，比如 VMware ESX

<img src="../img/image (37).png" style="zoom:67%;" />

以上简单介绍了虚拟机的两种分类：进程虚拟机和系统虚拟机，接下来重点介绍进程虚拟机中比较特殊的一类：高级语言虚拟机

### **1.3 高级语言虚拟机**

高级语言虚拟机本质上还是进程虚拟机，所以它的层级与进程虚拟机是一致的，这里以java虚拟机为例将其细化，如下图所示，与进程虚拟机基本一致，这里的Java核心类库跨越了多个层级：

- 对于部分核心类库，如java.lang包，需要靠Java虚拟机来对语义进行实现 

- 部分核心类库会中调用了native的方法，若是涉及到系统级指令如I/O指令等则需要通过系统调用来执行，若只是涉及到用户态的指令，则无需陷入内核

<img src="../img/image (39).png" style="zoom:50%;" />

一个**高级语言虚拟机**与**传统的进程虚拟机**有一个比较大的区别在于，高级语言虚拟机的指令系统\(ISA\)仅仅是为用户模式的程序定义的，**并不是针对一个真实的硬件处理器**而设计的，它将仅仅在一个虚拟机的处理器上执行，因此也被称为**虚拟指令系统**（V-ISA）

<img src="../img/image (34).png" style="zoom:50%;" />

我们可以从语言/编译器的角度来看待高级语言虚拟机：

在传统系统中，编译器包含一个执行词法、语法和语义分析的前端，它会生成**简单的中间代码\(**类似与机器代码但更抽象\)，这种中间代码通常不包含具体的寄存器分配，然后一个**代码生成器（编译器后端）**接收中间代码，生成面向特定指令系统和操作系统的**二进制机器代码**，这些二进制文件被发布，并且只能在支持编译时给定的指令集系统和操作系统的平台上执行，否则需要重新编译

高级语言虚拟机改变了这种模式，它的编译步骤类似于传统模式，但是在更高的层次发布程序的代码，如上图所示，这个虚拟ISA实际上就是虚拟机的机器代码。可移植的虚拟ISA代码可以被发布，在不同的平台上执行，每个平台都要实现能执行虚拟ISA的虚拟机

以java语言为例，如下图所示，Class文件就对应着虚拟ISA\(可移植代码\)

<img src="../img/image (41).png" style="zoom:50%;" />

参考书籍：《虚拟机系统与进程的通用平台》




# Java体系要解决的问题


## 语言：符号组织

### 符号定义

用于数据类型的关键字有 boolean、byte、char、 double、 false、float、int、long、new、short、true、void、instanceof

- 类型

	- 原生类型

	- 引用类型

		- 类类型

			- Object对象

			- String对象

		- 接口类型

		- 泛型

			- 类型擦除

			- 边界

			- 通配符

			- 自限定

			- 协变

		- 数组类型

	- 类型转换

		- 原生类型转换

		- 引用类型转换

		- 自动装箱

- 值

- 变量

- 关键字

### 过程结构

用于语句的关键字有break、case、 catch、 continue、 default 、do、 else、 for、 if、return、switch、try、 while、 finally、 throw、this、 super

- 顺序结构

- 选择结构

	- if...else 语句

	- switch 语句

	- 条件三目运算符

	- Optional、各种类库等

	- assert

	- try...catch 等

- 循环结构

	- while 语句

	- do...while 语句

	- for 语句

	- 语法糖等

	- 迭代器Iterator

### 符号编译

用于修饰的关键字有 abstract、final、native、private、 protected、public、static、transient  
  
用于方法、类、接口、包的关键字有 class、 extends、 implements、interface、 package、import

- 重载(overloading)

- 遮蔽(shadow)

- 遮掩(obscure)

- 隐藏(hide)

### 执行准备

- 加载

- 链接

	- 准备

	- 验证

	- 解析

		- 覆盖(overriding)

- 初始化

## 时间：并发同步

用于并发同步的关键字：synchronized、volatile

### java语言标准

- Java代码层面

	- volatile

	- synchronize

	- final

	- Concurrent工具包

- Java内存模型

	- 要解决的问题

		- 可见性问题

		- 指令重排问题

	- 要满足的原则

		- happens-before原则
		  解决多线程并发结果可预测的规则

### jvm标准

- volatile

	- 给字段加上ACC_VOLATILE标志

- synchronize

	- 同步方法

		- 通过读取运行时常量池中方法的 ACC_SYNCHRONIZED 标志来隐式获取monitor

	- 同步语句块

		- 通过monitorenter, monitorexit这两个指令来显式获取和释放moniter

### Hotspot实现

- synchronize

	- 重量锁

	- 轻量锁

	- 偏向锁

- volatile

	- lock指令

## 空间：内存管理

### JVM运行时数据区

- java堆

- 虚拟机栈

- 本地方法栈

- 程序计数器

- 方法区

	- 运行时常量池

### 程序的运行过程

- 类加载、链接、初始化

	- 方法区

- 执行

	- 方法执行

		- 虚拟机栈

		- 本地方法栈

	- 类的实例化

		- 对象

			- 创建

			- 销毁

				- GC机制

					- GC时机

					- GC对象

					- GC算法

- 类卸载


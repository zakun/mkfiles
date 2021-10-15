## PHP7新特性及优化

> 参考链接: https://www.cnblogs.com/songgj/p/10398076.html


### 1. 概述
**php7.x增加的新特性介绍可以参考这里：**

> https://www.runoob.com/php/php7-new-features.html

> https://php.net/manual/zh/migration70.new-features.php

php7.x版本系列相比之前的php的版本提交性能提高了不少，下图是wordpress在不同php版本下的压力测试表现。


![1](https://img2018.cnblogs.com/blog/824470/201906/824470-20190617112252274-2108652541.png)

![2](https://img2018.cnblogs.com/blog/824470/201906/824470-20190617112355918-1080464503.png)
 

**这里面其中的一些主要改变是性能提高的关键，主要有以下内容。**


### 2. zval使用栈内存

 在zend引擎和扩展中，经常要创建php变量，其底层就是一个zval指针，之前的php版本都是通过MAKE_STD_ZVAL动态的从堆内存上分配一个zval内存。而php7直接使用栈内存，好处是少了一次内存分配。php程序中回大量创建变量，所以php7会在栈上预分配一块内存来存放这些zval，来节省大量的内存分配和管理操作。

__php5__

	zval *val ; MAKE_STD_ZVAL(val)

__php7__

	zval val;
 

### 3. zend_string存储hash值

zend_string存储hash值，array查询不再需要重复计算hash

数组是php比较重要的数据结构，php程序中会有大量的array关联查询，虽然hashtable查找的复杂度是O（1），但是key的值每次要转化成一个hash值，需要用一个复杂的hash函数去计算的，这样就会占用cpu时间，不过不光数组，在php底层很多地方都会用到hashtable，比如类的属性方法函数等。其实php程序运行起来大部分key的值是不变的，所以php7就保存了这些hash值下次直接使用，那么php7就为字符串单独创建了新类型叫做zend_string,除了char*指针和长度之外，增加了一个hash字段，用于保存字符串的hash值，数据键值查找不再反复需要计算hash值。为了优化数组的键值查找。

![3](https://img2018.cnblogs.com/blog/824470/201902/824470-20190218210857333-236172514.png)

上图代码中 `zend_ulong h`；就是存储`hash`值。

### 4. hashtable桶内直接存放数据

hashtable桶内直接存放数据，减少了内存申请次数，顺便也提升了cache命中率和访问速度。因为指针不是连续的是分布在不同的内存页上，如果读取第一个或者第三个桶，它们的数据可能会在两个页上。
 
php7之前

![3](https://img2018.cnblogs.com/blog/824470/201902/824470-20190218212706496-2112972893.png)

数据存放是在上图arBuckets这个结构体上，存放了一些bucket * 指针，指针上就是对应了一些数据。php7对这些做了一些改进，如下图。

php7

![4](https://img2018.cnblogs.com/blog/824470/201902/824470-20190218212706496-2112972893.png)

php7将之前arBuckets改成了上图中的arData，而这个arData直接就是一个大块内存，这个内存上面就是一个个桶bucket，这样的好处就是每次数据就不需要动态去申请内存。


### 5. zend_parse_parameters改为宏实现，性能提升15%。

### 6. 新增加4种opcode

`call_user_function()`,`is_int()`,`is_string()`,`is_array()`,`strlen()`,`defined()` 4个函数变为`php opcode指令`，速度更快。

### 7. AST（Abstract syntax tree）抽象语法树

PHP7 的内核中有一个重要的变化是加入了 AST（Abstract syntax tree）抽象语法树，指代码在计算机内存的一种树状数据结构，树上的每个节点都表示源代码中的一种结构，便于计算机理解和解析。

![5](https://img2018.cnblogs.com/blog/824470/201906/824470-20190617130827048-1639004078.png)

在 PHP5系列版本中，从 php 脚本到 opcodes 的执行的过程如下：

- 词法扫描分析（Lexing）：将源文件转换成 token 流；

- 语法分析（Parsing）：生成 op arrays。

PHP7 中在语法分析阶段先生成 AST：

- 词法扫描分析（Lexing）：将源文件转换成 token 流。

- 语法分析（Parsing）：从 token 流生成抽象语法树。

- Compilation：从抽象语法树生成 op arrays。　　  

这个表达式`（$a)['b'] = 1` 就会被解析成下图这样的一棵树结构

![6](https://img2020.cnblogs.com/i-beta/824470/202003/824470-20200311134621087-1784422073.png)


### 8.其他更多性能优化

如基础类型 float ， int ， bool等改成直接进行值拷贝。排序算法改进了，PCER with JIT , execute_data和opline使用全局寄存器，使用gdb4.8的PGO功能。

### 9. php7与JIT

最初HHVM退出一个很重要的特性就是JIT，JIT就是just in time的缩写，表示运行时候将指令转换成二进制机器码，我们知道C和C++是将源代码编译然后生成二进制机器码去执行的，而php，python等脚本语言是将源代码转换成中间指令然后在vm（虚拟机）上执行，另外java系语言他们使用的JVM引擎底层也是JIT，是将java的字节码编译成二进制的机器码去执行的。对于计算密集型的的程序，JIT可以将PHP的opcode直接转换成机器码，可以大幅度提升PHP性能。

**不过PHP7.0-final版本中不会带有JIT特性的。**


__为什么php7版本没有使用JIT呢？__

是因为php官方之前有个php中间版本是带有JIT的，后来php官方开发组使用JIT测试时候发现JIT对于实际项目的性能没有太大的性能提升，所以最终放弃使用JIT方案。但后来发现密集计算性的php程序使用JIT后性能还会大幅提升。

### 10. 总结

1. 存储变量的结构体变小，尽量使结构体里成员共用内存空间，减少引用，这样内存占用降低，变量的操作速度得到提升。

2. 字符串结构体的改变，字符串信息和数据本身原来是分成两个独立内存块存放，php7尽量将它们存入同一块内存，提升了cpu缓存命中率。

3. 数组结构的改变，数组元素和hash映射表在php5中会存入多个内存块，php7尽量将它们分配在同一块内存里，降低了内存占用、提升了cpu缓存命中率。

4. 改进了函数的调用机制，通过对参数传递环节的优化，减少一些指令操作，提高了执行效率。
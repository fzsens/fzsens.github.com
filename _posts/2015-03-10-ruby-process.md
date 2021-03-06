---
layout: post
title: Ruby的运行过程
date: 2015-3-10
categories: programming
tags: [Ruby]
description: Ruby以及同类型脚本的执行流程
---

1. Ruby是一门脚本语言，默认是按照语句出现的顺序进行执行，通过控制结构可以变化行为，比如循环等。

2. Ruby与C等编译语言相比，没有一个`main`方法可以作为执行的入口，而是接受一个待执行语句的脚本，然后从第一行代码执行到最后一行。Ruby解释器现在文件中扫描`BEGIN`语句，然后执行该语句所包含的代码，执行`BEGIN`代码块之后，会回到第一行代码开始顺序执行。

3. 另一个差异是关于模块、类和方法的定义，在编译型语言中，上述的语法结构是由编译器来完成的。在Ruby中，它们也是语句，Ruby解释器遇到一个类定义或者方法定义的时候，就执行该语句，产生一个新的类或者新的方法。在后续部分中，解释器可能会碰到执行一个对方法的调用，这个调用就会执行该方法体内的语句。

4. 执行的终止  

>- 它执行一个导致Ruby程序终结的语句
>- 它到达文件的结尾
>- 它读入一行代码，此代码用标记`__END__`标记了文件的逻辑结尾

通常情况下，(除非是调用了`exit!`方法)Ruby解释器在退出之前都会执行任何`END`语句，以及通过`at_exit`函数注册的关闭钩子`shudown hook`代码。
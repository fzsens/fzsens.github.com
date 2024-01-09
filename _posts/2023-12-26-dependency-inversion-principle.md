---
layout: post
title: 依赖倒置原则
date: 2023-12-26
categories: architecture
tags: [architecture,solid]
---

SOLID 原则是 Uncle Bob 提出的在面向对象编程中的五个设计原则，其中最后一个依赖倒置原则（DIP）是比较不容易理解，特别是和 Dependency Injection 放在一起讨论的时候。


什么是依赖关系，看下面的代码

```java
public class A {
   private B componentB;
   
   public A(B b) {
     this.componentB = b;
   }
   
   public void funcOfA() {
     // pass
     this.componentB.funcOfB();
     // pass
   }
}
​
```

![a2b](/assets/img/solid/a2b.png)

在这个最简单的不包含任何业务逻辑示例代码中，A 需要借助 B 来做一些事情，A 无法独立完成工作，这就产生了依赖，A 依赖于 B。

依赖倒置的定义是
A. High-level modules should not depend on low-level modules. Both should depend on abstractions.
B. Abstractions should not depend on details. Details should depend on abstractions.
翻译一下是
A. 高层级模块不应该依赖于低层级模块，两者都应该依赖抽象
B. 抽象不应该依赖细节，细节应该依赖抽象

上面的例子中，A 属于 high-level module， B 属于 low-level module，很显然这违背了 DIP 原则，导致的结果是：
1. A 无法离开 B 而单独存在，当你在考虑和设计 B 的时候需要时刻知道 B 的存在，换而言之，A 无法自洽和自治
2. 当要修改 A 的 funcOfA 的时候，有两种可能：修改 B 的代码，或者修改 A 的代码，这两种方法都会对整体应用的稳定性和可维护性带来破坏，也违背了另外一个 OCP（开闭原则）。

在遇到这样情况的时候，我们一般选择增加一个抽象层，在 Java 中可以简单实现为一个接口，一般写下来的代码如下

```java
package B;
public interface IB {
   public void funcOfB();
}

package B;
public class B implements IB {
    public void funcOfB() {
      // pass
    }
}

package A;
public class A {
   private IB componentB;
   public A(IB b) {
     this.componentB = b;
   }
   public void funcOfA() {
     // pass
     this.componentB.funcOfB();
     // pass
   }
}
```

这其实也是在工作中，我们使用最多的一种编码方式。但是仔细思考一下这种设计只解决了上面的第二个问题。**并没有解决 A 自洽和自治的问题**，因为 IB 本质上只是把 B 的方法做了一个抽取，也就是在对 A 进行建模的时候，依然需要感知 B 的存在，**A 依然是依赖 B** 的。

![a2ib](/assets/img/solid/a2ib.png)

如果我们从 A 自洽的角度思考：

1. A 要实现一个功能，那么这个功能的定义所需要的所有信息，A 应该都是具备的。
2. A 可以针对自己要实现这个功能的定义进行抽象 AA，AA 的所有权属于 A 。
3. 接下来 A 只需要依赖 AA 就可以了实现建模了。

```java
package A;
public interface AA {
   public void contractOfA();
}
package A;
public class A {
   private AA aa;
   public A(AA b) {
     this.aa = b;
   }
   public void funcOfA() {
     // pass
     this.aa.contractOfA();
     // pass
   }
}

package B;
public class B implements AA {
    public void contractOfA() {
      // pass
    }
}
```

在整个思考过程中，A 不需要感知自身以外的其他信息，对于 B 而言，需要做的是遵循这个抽象 AA ，按照其规约来实现功能。

![a2aa](/assets/img/solid/a2aa.png)

对比后面这两个实现，此时 A 变得自洽，B 反过来依赖 A 了，这就是依赖倒置，A 不再感知 B 最终确可以使用 B，**而 B 作为被使用方，反过来依赖使用方**。

最后写出来的代码除了名字的差异外，格式是一样的，但是思考的模式却发生了很大的变化，作为使用方的 **A 在建模的时候具有完全的控制权**，这同时解决了上面提出来的两个问题。这也可以作为为什么编程中最难的事之一是取一个合适的名字，因为一个合适的名字代表一个经过深度思考的模型。

在理解 DIP 之后，可以有以下更加进一步的延伸讨论
1. DIP 是 OCP 的基础之一
2. DIP 编写良好的测试用例的基础——只有测试对象外部依赖非常少的时候，流畅的测试先行才是可能的
3. DIP 除了在狭义的代码层面发挥作用，在更加广义的服务关系和应用关系中也适用

最后，DIP 和 Spring 中依赖注入 Dependency Injection 是什么关系？
简而言之两者讨论的维度是不同的，DI 讨论的是在**运行时依赖方和被依赖方**是如何进行实例化和组装管理的，而 DIP 讨论的这是关于代码的组织方式的问题；二者的联系是，当代码符合 DIP 的时候，通过 DI 框架可以更加容易地管理这种依赖在运行时的关系。

参考资料

[Dependency Inversion Principle - Spring Framework Guru](https://springframework.guru/principles-of-object-oriented-design/dependency-inversion-principle/)

[Understanding SOLID Principles: Dependency Inversion - DEV Community](https://dev.to/tamerlang/understanding-solid-principles-dependency-inversion-1b0f)

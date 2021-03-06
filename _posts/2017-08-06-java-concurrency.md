---
layout: post
title:  Java Concurrency
date:   2017-08-06 15:52:00 +0800
categories: Java
tag: Concurrency
---

* content
{:toc}


本文参考《Java Concurrency in Practice》。
<hr>

1.简介
====================================

编写正确的程序很难，而编写正确的并发程序则难上加难。

2.线程安全性
====================================

要编写线程安全的代码，核心在于要对状态访问操作进行管理，特别是对共享（Shared）和可变（Mutable）状态的访问。从非正式意义上来说，对象的状态是指存储在状态变量（例如实例或静态域）中的数据。对象的状态可能包括其他依赖对象的域。

在对象的状态中`包含了任何可能影响其外部可见行为的数据`。`共享`意味着变量可以由多个线程同时访问，而`可变`则意味着变量的值在其生命周期内可以发生变化。一个对象是否需要是线程安全的，取决于它是否被多个线程访问。要使得对象是线程安全的，需要采用同步机制来协同对对象可变状态的访问。如果无法实现协同，那么可能会导致数据破坏以及其他不该出现的结果。

当多个线程访问某个状态变量并且其中有一个线程执行写入操作时，必须采用同步机制来协同这些线程对变量的访问。Java中的主要同步机制是关键字`synchronized`，它提供了一种独占的加锁方式，但“同步”这个术语还包括volatile类型的变量，显示锁（Explicit Lock）以及原子变量。

> 如果当多个线程访问同一个可变的状态变量时没有使用合适的同步，那么程序就会出现错误。以下三种方式可以修复这个问题：
>
> + 不在线程之间共享该状态变量。
> + 将状态变量修改为不可变的变量。
> + 在访问状态变量时使用同步。

如果在设计类的时候没有考虑并发访问的情况，那么在采用上述方法时可能需要对设计进行重大修改，因此要修复这个问题可谓是知易行难。如果从一开始就设计一个线程安全的类，那么比在以后再将这些修改为线程安全要容易的多。

在一些大型程序中，要找出多个线程在哪些位置上将访问一个变量是非常复杂的。幸运的是，面向对象这种技术不仅有助于编写出结构优雅、可维护性高的类，还有助于编写出线程安全的类。访问某个变量的代码越少，就越容易确保对变量的所有访问都实现正确同步，同时也更容易找出变量在哪些条件下被访问。Java并没有强制要求将状态都封装在类中，可以将状态保存在某个公开的域（甚至公开的静态域）中，或者提供一个对内部对象的公开引用。然而，程序状态的封装性越好，就越容易实现程序的线程安全性，并且代码的维护人员也越容易保持这种方式。

> 当设计线程安全的类时，良好的面向对象技术、不可修改性，以及明晰的不变性规范都能起到一定的帮助作用。

在某些情况下，良好的面向对象设计技术与实际情况的需求并不一致。这时候可能要牺牲一些良好的设计原则，以换取性能或者对遗留代码的向后兼容。有时候，面向对象中的抽象和封装会降低程序的性能（你可能会怀疑这句话），但在编写并发程序时一种正确的编程方法就是：首先使代码正确运行，然后再提高代码速度。即便如此，最好也只是当性能测试结果和应用需求告诉你必须提高性能，以及测量结果表明这种优化在实际环境中确实可以带来性能提升时，才进行优化。

> 在编写并发代码时，应该始终遵循这个原则。由于并发错误是非常难以重现以及调试的，因此如果只是在某段很少执行的代码路径上获得了性能的提升，那么很有可能被程序运行时存在的失败风险而抵消。

2.1什么是线程安全性
------------------------------------

线程安全性的定义中，最核心的就是正确性。

正确性的含义是，某个类的行为与其规范完全一致。在良好的规范中通常会定义各种不变性条件来约束对象的状态，以及定义各种后验条件来描述对象操作的结果。在对正确性给出了一个较为清晰的定义后，就可以定义线程安全性：当多个线程访问某个类时，这个类始终都能表现出正确的行为，那么就称这个类是线程安全的。


3.对象的共享
====================================

4.对象的组合
====================================

我们并不希望对每次一的内存访问都进行分析以确保程序是线程安全的，而是希望将一些现有的线程安全组件组合为更大规模的组件或程序。这一章将介绍一些组合模式，这些模式能够使一个类更容易称为线程安全的，并且在维护这些类时不会无意中破坏类的安全保证。

4.1设计线程安全的类
-----------------------------------
在线程安全的程序中，虽然可以将程序的所有状态都保存在公有的静态域中，但与那些将状态封装起来的程序相比，这些程序的线程安全性更难以得到验证，并且在修改时也更难以确保其线程安全性。通过使用封装技术，可以使得在不对整个程序进行分析的情况下就可以判断一个类是否是线程安全的。
在设计线程安全类的过程中，需要包含以下三个基本要素：
+ 找出构成对象状态的所有变量。

+ 找出约束状态变量的不变性条件。

+ 建立对象状态的并发访问管理策略。

要分析对象的状态，首先从对象的域开始。如果对象中的所有域都是基本类型的变量，那么这些域将构成对象的全部状态。程序4-1中的Counter只有一个域value，因此这个域就是Counter的全部状态。对于含有n个基本类型域的对象，其状态就是这些域构成的n元组。例如，二维点的状态就是它的坐标值（x，y）。**如果在对象的域中引用了其他对象，那么该对象的状态将包含被引用对象的域**。例如LinkedList的状态就包括该链表中所有节点对象的状态。

  ``` java
//程序4-1 使用Java监视器模式的线程安全计数器  
@ThreadSafe
  public final class Counter {
  	@GuardedBy("this") private long value = 0;
    	public synchronized long getValue() {
        return value;
    	}
    	public synchronized long increment() {
        if(value == Long.MAX_VALUE)
          throw new illegalStateException("counter overflow");
        return ++value;
    	} 
  }
  ```

同步策略（synchronization Policy）定义了如何在不违背对象不变性条件或后验条件的情况下对其状态的访问操作进行协同。同步策略规定了如何将不可变性、线程封闭与加锁机制等结合起来以维护线程的安全性，并且还规定了哪些变量由哪些锁来保护。要确保开发人员可以对这个类进行分析与维护，就必须将同步策略写为正式文档。

4.1.1 收集同步需求

要确保类的线程安全性，就需要确保它的`不可变性条件`不会再并发访问的情况下被破坏，这就需要对其状态进行推断。对象和变量都有一个状态空间，即所有可能的取值。状态空间越小，就越容易判断线程的状态。final类型的域使用的越多，就越能简化对象可能状态的分析过程。（极端情况下，不可变对象只有唯一的状态）

在许多类中都定义了一些不可变条件，用于判断状态是否有效。Counter中的value域是long类型的变量，其状态空间从Long.MIN_VALUE到Long.MAX_VALUE，但value在取值范围上存在着一个限制，即不能为负值。

同样，在操作中还会包含一些`后验条件`来判断状态迁移是否是有效的。如果Counter的当前状态是17，那么下一个有效状态只能是18。**当下一个状态需要依赖当前状态时，这个操作就必须是一个复合操作**。并非所有的操作都会在状态转换上施加限制，例如，当更新一个保存当前温度的变量时，该变量之前的状态并不会影响计算结果。

由于不变性以及后验条件在状态及状态转换上施加了各种约束，因此就需要额外的同步与封装。如果某些状态是无效的，那么必须对底层的状态变量进行封装，否则客户代码可能会使对象处于无效状态。**如果在某个操作中存在无效的状态转换。那么该操作必须是原子的**。另外，如果在类中没有施加这种约束，那么就可以放宽封装性或序列化等需求，以便获得更高的灵活性或性能。

在类中也可以包含同时约束多个状态变量的不变性条件，在一个表示数值范围的类中可以包含两个状态的变量，分别表示范围的上下界。这些变量必须遵守的约束是：下界值应小于或等于上届值。类似于这种包含多个变量的不变性条件将带来原子性的需求：这些相关变量必须在单个原子操作中进行读取或更新。不能先更新一个变量，然后释放锁并再次获得锁，然后在更新其他变量。**因为释放锁后，可能会使对象处于无效状态**。如果在一个不变性条件中包含多个变量，那么在执行任何访问相关变量的操作时，都必须保护这些变量的锁。

> 如果不了解对象的不变性条件与后验条件，那么就不能确保线程安全性。要满足在状态变量的有效值或状态转换上的各种约束条件，就需要借助于原子性与封装性。

4.1.2依赖状态的操作

类的不变性条件与后验条件约束了在对象上有哪些状态和状态转换是有效的。在某些对象的方法中还包含一些基于状态的先验条件。例如，不能从空队列中移除一个元素，再删除元素前，队列必须处于非空状态。如果在某个操作中包含基于状态的先验条件，那么这个操作就称为`依赖状态的操作`。

在单线程程序中，如果某个操作无法满足先验条件，那么就只能失败。但在并发程序中，先验条件可能会由于其他线程执行的操作而变成真。在并发程序中要一直等到先验条件为真，然后在执行该操作。

在Java中，等待某个条件为真的各种内置机制（包括的等待和通知机制）都与内置加锁机制紧密关联，要想正确地使用它们并不容易。要想实现某个等待验证先验条件为真时才执行的条件，一种更简单的方法是通过现有库中的类（例如阻塞队列[Blocking Queue]或信号量[Semaphore]）来实现依赖状态的行为。这些会在后续进行介绍。

4.1.3状态的所有权

4.1节中曾指出，如果以某个对象为根节点构造一张对象图，那么该对象的状态将是对象图中所有对象包含的域的一个子集。为什么是“子集”？在从对象可以达到的所有域中，需要满足哪些条件才不属于对象状态的一部分？

在定义哪些变量将构成对象的状态时，只考虑对象拥有的数据。所有权（Ownership）在Java中并没有得到充分的体现，而是属于类设计中的一个要素。如果分配并填充了一个HashMap对象，那么就相当于创建了多个对象：HashMap对象，在HashMap对象中包含的多个对象，以及在Map.Entry中可能包含的内部对象。HashMap对象的逻辑状态包括所有的Map.Entry对象以及内部对象，即使这些对象都是一些独立的对象。

无论如何，垃圾回收机制使我们避免了如何处理所有权的问题。在C++中，当把一个对象传递给某个方法时，必须认真考虑这种操作是否传递对象的所有权，是短期还是长期的所有权。在Java中同样存在这些所有权模型，只不过垃圾回收器为我们减少了许多在引用共享方面常见的错误，因此降低了在所有权处理上的开销。

许多情况下，所有权和封装性总是相互关联的：对象封装它拥有的状态，反之也成立，即对它封装的状态拥有所有权。状态变量的所有者将决定采用何种加锁协议来维持变量状态的完整性。所有权意味着控制权。然而，如果发布了某个可变状态的引用，那么就不再拥有独占的控制权，最多是“共享控制权”。对于从构造函数或者从方法中传递进来的对象，类通常并不拥有这些对象，除非这些方法是被专门设计为转移传递进来的对象的所有权（例如，同步容器封装器的工厂方法）。

容器类通常表现出一种“所有权分离”的形式，其中容器类拥有器自身的状态，而客户代码则拥有容器中各个对象的状态。Servlet框架中的ServletContext就是其中一个示例。ServletContext为Servlet提供了类似于Map形式的对象容器服务，在ServletContext中可以通过名称来注册（setAttribute）或获取（getAttribute）应用程序对象。由Servlet容器实现的ServletContext必须是线程安全的，因为它肯定会被多个线程同时访问。当调用setAttribute和getAttribute时，Servlet不需要使用同步，但当使用保存在ServletContext中的对象时，则可能需要同步。这些对象由应用程序拥有，Servlet容器只是替应用程序报关它们。与所有共享对象一样，它们鼻血被安全地共享。为了防止多个线程在并发访问同一个对象时产生的相互干扰，这些对象应该要么是线程安全的对象，要么是事实不可变的对象，或者由锁来保护的对象。

4.2实例封闭
---------------------------------

如果某对象不是线程安全的，那么可以通过多种技术使其在多线程程序中安全地使用。你可以确保该对象只能由单个线程访问（线程封闭），或者通过一个锁来保护对该对象的所有访问。

封装简化了线程安全类的实现过程，它提供了一种实例封闭机制（Instance Confinement，简称封闭），当一个对象被封装到另一个对象中时，能够访问被封装对象的所有代码路径都是已知的。与对象可以由整个程序访问的情况相比，更易于对代码进行分析。通过将封闭机制与合适的加锁策略结合起来，可以确保以线程安全的方式来使用非线程安全的对象。

> 将数据封装在对象内部，可以将数据的访问限制在对象的方法上，从而更容易确保线程在访问数据时总能持有正确的锁。



<hr>
​最后的最后，老婆我爱你。









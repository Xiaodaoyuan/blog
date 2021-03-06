---
title: Java内存模型
date: 2018-10-07 15:58:29
categories: Java
tags: 并发
---

### 1、Java内存模型的抽象结构  
　　Java内存模型就是Java Memory Model（以下简称JMM），是Java虚拟机规范的一部分。JMM规定：变量存储在主存中，每个线程有自己的工作内存，线程工作内存中保存了变量在主存的副本拷贝。主存是线程共享的，工作内存变量是各个线程独享的。线程工作时，从主存中复制变量到线程的工作内存，然后在工作内存中操作变量副本，最后再把变量副本写回到主内存。
<!--more-->
![图1-Java内存结构](/images/thread-jmm.png)
Java内存模型是一种共享内存模型：
- 线程只能操作工作内存，不能直接操作主存。
- 每个线程只能访问自己的工作内存，不能访问其他线程的工作内存。
- 线程之间的数据通信必须通过主存中完成。

---

### 2、并发三问题
- **重排序**  
为了提高性能，编译器和处理器会对指令进行重排序，重排序主要有三种类型：
    1. 编译器重排序：编译器在不改变单线程程序语义的前提下，可以重新安排字节码的执行顺序；也就是编译生成的机器码顺序和源代码顺序不一样。
    2. 处理器重排序：现代处理器采用指令级并行技术将多条指令重叠执行，如果不存在数据依赖性，处理器可以改变语句对应机器指令的执行顺序；也就是CPU在执行字节码时，执行顺序可能和机器码顺序不一样。
    3. 内存重排序：由于处理器使用缓存和读/写缓冲区，这使得加载和存储操作看上去可能是在乱序执行。
    
```
class ReorderExample {
  int a = 0;
  boolean flag = false;

  public void writer() {
    a = 1;                   // A1
    flag = true;             // A2
  }

  public void reader() {
    if (flag) {              // B1
      int i =  a * a;        // B2
    }
  }
}
```

　　线程1执行writer()方法，线程2执行reader()方法，结果怎样？  
　　由于writer()方法中的A1和A2操作没有依赖关系，所以可能被重排序，先执行A2，再执行A1，这种情况下由于a=0，所以i=0；reader方法中的B1、B2虽然有依赖关系，但是仍然可以重排序，B2先执行a*a=0放在缓存，当flag为true时将i赋值为0。
- **内存可见性**  
内存可见性问题是由于多核缓存引起的。现在的多核cpu都有自己的一级缓存和二级缓存。Java作为高级语言，屏蔽了底层这些细节，构建了JMM规范。JMM抽象了主存和工作内存的概念。由于线程可能不能及时将工作内存中变量刷到主存，造成其他线程读取到的变量可能是错误的，这就是内存可见性问题。
- **原子性**  
原子性即一个操作不可拆分。JMM只保证像read/load/store/write这样很少的操作是原子性的，甚至在32位平台下，对64位数据的读取和赋值都不能保证其原子性（long变量赋值是需要通过两个操作来完成的）。简单说，int i=10; 是原子的；i = i + 1 不是原子的；甚至long v = 100L 也可能不是原子的。

---

### 3、volatile关键字
volatile关键字有两个作用：一是保证内存可见性；二是防止指令重排序。
- 可见性：被volatile修饰的变量，如果一个线程修改了其的值，另一个线程能够立即读取到最新的值。
- 禁止指令重排序：被volatile修饰的变量，保证在它前面的代码先执行，在它后面的代码后执行。

**volatile小结**
1. volatile 修饰符适用于以下场景：某个属性被多个线程共享，其中有一个线程修改了此属性，其他线程可以立即得到修改后的值。在并发包的源码中，它使用得非常多。
2. volatile 属性的读写操作都是无锁的，它不能替代 synchronized，因为它没有提供原子性和互斥性。因为无锁，不需要花费时间在获取锁和释放锁上，所以说它是低成本的。
3. volatile 只能作用于属性，我们用 volatile 修饰属性，这样 compilers 就不会对这个属性做指令重排序。
4. volatile 提供了可见性，任何一个线程对其的修改将立马对其他线程可见。volatile 属性不会被线程缓存，始终从主存中读取。

---

### 4、happens-before规则
　　happens-before用来指定两个操作之间的顺序，这两个操作可以在一个线程内，也可以在不同线程之间。因此，JMM可以通过happens-before关系向程序员提供跨线程的内存可见性。  
　　JSR-133定义了如下happens-before规则：
- 程序顺序规则：一个线程中的每个操作，happens-before于该线程的任意后续操作。
- 监视器锁规则：对一个锁的解锁，happens-before于随后对这个锁的加锁。
- volatile变量规则：对一个volatile变量的写，happens-before于随后对这个volatile变量的读。
- 传递性：如果A happens-before于B，B happens-before于C，可以得出A happens-before于C。
- start()规则：如果线程A执行操作ThreadB.start()（启动线程B），那么A线程的ThreadB.start()操作happens-before于线程B中的任意操作。
- join()规则：如果线程A执行操作ThreadB.join()并成功返回，那么线程B中的任意操作happens-before于线程A从ThreadB.join()操作成功返回。

---

### 5、synchronized关键字
synchronized关键字可以实现volatile的保证内存可见性以及禁止重排序，同时它还可以保证原子性操作。但是synchronized需要进行加锁操作，效率更低。

---

### 6、final关键字
final关键字可以用来修饰类、方法、变量。用来修饰变量时可以在对象初始化完成前，不要将此对象的引用写入到其他线程可以访问到的地方（不要让引用在构造函数中逸出）。如果这个条件满足，当其他线程看到这个对象的时候，那个线程始终可以看到正确初始化后的对象的 final 属性。

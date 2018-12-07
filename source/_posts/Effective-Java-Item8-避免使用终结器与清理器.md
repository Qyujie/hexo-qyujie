---
title: Effective Java Item8 避免使用终结器与清理器
date: 2018-12-07 17:57:07
tags:
    - Java
    - Effective Java
categories:
    - Java
    - Effective Java
    - chapter 2
---

**终结器是不可预测的、常常会很危险，而且通常没必要。使用终结器会导致奇怪的行为、孱弱的性能以及可移植性问题**。终结器有一些有效的用途，我们将在后面的条款中介绍，但是作为一个规则，你应该避免他们。在Java 9中，终结器已经被弃用，但是Java库仍然在使用它们。Java 9中，替代终结器的是清除器（**cleaner**）。**清除器比终结器危险小，但仍然不可预测、效率慢，而且通常没有必要**。 
<!-- more -->

C++程序员们不要将Java中的终结器或是清理器当作是C++中的析构函数。在C++中，析构函数是回收与对象所关联的资源的常规方式，它是与构造函数必要的一个对应之物。 在Java中，当与对象所关联的存储变得不可达时，垃圾收集器就会将其回收，不需要程序员做任何额外的事情。 c++析构函数也可以用于回收其他非内存资源。在Java中，`try-with-resources`或`try-finally`代码块就是用于此目的(Item 9)。

终结器与清理器的一个缺点在于，没有人可以保证他们会立刻执行[JLS, 12.6]。在对象变得不可及与终结器或是清理器开始运行之间可能会间隔任意长的时间。这意味着你永远不要在终结器或是清理器中做任何时间关键的事情。例如，依赖于终结器或清除器来关闭文件就是一个严重的错误，因为打开的文件描述符是有限的资源。如果由于系统运行终结器或是清理器产生了延迟而导致很多文件处于打开的状态，那么程序就有可能失败，因为它无法再打开文件了。

到底哪个终结器和清理器会执行主要是由垃圾回收算法来决定的，而算法在不同的实现间存在着较大的差别 。依赖于终结器或是清理器的立刻执行的程序行为也存在着较大的差别。因此，同样一个程序在你测试的JVM中完美运行，然后却在你最重要的客户上的机器上不幸地失败了，这种情况是完全有可能发生的。

终结器不会立刻执行并不仅仅是个理论上的问题 。为类提供终结器可能会随意地延迟自己的实例的回收。一位同事调试了一个长期运行的GUI应用程序，该应用程序因`OutOfMemoryError`而奇怪地崩溃过。分析显示，在应用程序崩溃的时候，它的终结器队列中有数千个图形对象等待被终结并回收。遗憾的是，终结器运行所在的线程要比另一个应用线程的优先级低，这样对象被终止的速度远远跟不上其进入到终止状态的速度。Java语言规范没有明确说明哪个线程将执行终结器，因此除了避免使用终结器之外，没有其他更方便的方法来防止这类问题。这个问题上，清洁器在这方面要比终结器好一些，因为类的创建者可以控制自己的清洁器线程，不过，清洁器依然运行在后台，在垃圾收集器的控制之下，因此对于立刻清洁这个问题也没有提供任何保证。

规范不仅没有提供终结器或是清理器会立刻运行的保证，也没有对其一定会运行提供任何保证。完全有可能出现这样的情况，当程序终止时，它并没有对早就处于不可达的对象运行其终结器和清理器。因此，你**永远都不应该依赖于终结器或是清理器来更新持久化状态**。比如说，依赖于终结器或是清理器来释放如数据库等共享资源上的持久化锁可能会导致整个分布式系统陷入瘫痪状态。

不要被`System.gc`和`System.runFinalization`方法所诱惑。它们可能会增加终结器或清除器被执行的几率，但他们并不能保证一定如此。曾经有两个方法做过这个保证：`System.runFinalizersOnExit`及其搭档`Runtime.runFinalizersOnExit`。这两个方法存在严重的问题，早就已经不建议使用了[ThreadStop]。

终结器的另一个问题是在执行终结时，未捕获的异常会被忽略掉，这时对象的终结会被终止[JLS, 12.6]。未捕获的异常会导致其他对象的状态被破坏掉。如果另一个线程试图使用这样一个已损坏的对象，则可能导致任意的不确定性行为。正常情况下，未捕获的异常会终止线程并打印堆栈信息，但如果在终结器中就不会这样——它甚至不会打印出任何警告信息。清理器不存在这个问题，因为使用了清理器的库会自己控制其线程。。

**使用终结方法和清除方法会有严重的性能损失**。在我的机器上，创建一个简单的`AutoCloseable`对象，使用`try-with-resources`关闭它，然后让垃圾收集器对其进行回收，大约需要花费12ns。使用终结器可以将时间增加到550纳秒。换句话说，使用终结器创建和销毁对象的速度要慢50倍。这主要是因为终结器阻碍了高效的垃圾回收。如果使用清理器来清除类的所有实例（在我的机器上每个实例大约需要花费500ns），它在速度上与终结器大致相同；不过，如果只是将清理器作为一个安全网（后续将会介绍），那么其速度将会快很多。如下所述。在这些情况下，我的机器上创建、清理与销毁一个对象所花费的时间大约需要66ns，这意味着你为安全网的使用需要付出5倍因子（不是50倍）的代价。

**终结方法有一个严重的安全问题：他们会使你的类遭遇到终结器攻击**。终结器攻击背后的想法非常简单：如果异常是从构造方法或是序列化方法`readObject`与`readResolve`中抛出的（chapter 12），那么恶意的子类终结器就会运行在部分构建完毕的对象上，而这个对象本应该『中途夭折的』。这个终结器会将对对象的引用记录在一个静态字段中，防止其被垃圾回收掉。一旦将这个不完整的对象记录下来后，我们就可以轻松调用这个对象上的任意方法，而这个对象原本是不应该存在的。**从构造方法中抛出异常足以禁止对象的创建；但在使用终结器的情况下，却并非如此**。 这种攻击还会产生非常严重的后果。终态类不受终结器攻击的影响，因为没人可以创建终态类的恶意子类。若想保护非终态类免受终结器攻击，**请编写一个什么都不做的`final`的`finalize`方法**。

那么，对于封装了需要终止的资源（如文件或是线程）的对象来说，如果不为类编写终结器或是清理器，那该怎么办呢？只需让类实现`AutoCloseable`即可，并让其客户端在不需要其实例时调用其`close`方法，通常我们会使用`try-with-resources`来确保终止，即便在异常的情况下亦如此（Item 9）。值得提及的一个细节是，实例必须要追踪其是否已经关闭了：`close`方法必须要在一个字段中记录下对象已经不再有效了，其他方法则必须要检查该字段，如果当对象已经关闭后还调用这些方法，那就需要抛出`IllegalStateException`异常。

那么，清理器与终结器到底有什么好处呢？他们有两个合理的用途。一是作为安全网，防止资源所有者忘记调用其`close`方法。虽然没人能够保证清理器或是终结器会立刻运行（或是否运行），不过如果客户端忘记释放资源，那么迟做总比不做强。如果考虑编写这样的安全网终结器，那么请仔细考虑，这种保护是否真的值得。一些Java库类（如`FileInputStream`、`FileOutputStream`、`ThreadPoolExecutor`及`java.sql.Connection`）都将终结器作为安全网。

清理器的第二个合理使用场景与拥有本地对端（`native peers`）的对象有关。所谓本地对端指的是本地对象（非Java对象），常规对象通过本地方法将调用委托给它由于本地对端并非常规对象，因此垃圾收集器并不知晓它，当其Java对端被回收时，也并不会对其进行回收。清理器或是终结器是完成这个任务的恰当工具，假设性能是可接受的，并且本地对端并没有持有关键资源。如果性能是不可接受的，或是本地对端持有必须要立刻回收的资源，那么类就应该拥有一个`close`方法，如前所示。

清理器的使用有一些棘手。如下是个简单的`Room`类，演示了其使用方式。假设房间在被回收前必须要清理。`Room`类实现了`AutoCloseable`；其自动清理安全网使用了清理器这个事实只不过是个实现细节而已。与终结器不同，清理器不会污染类的公有API：

``` java
// An autocloseable class using a cleaner as a safety net
public class Room implements AutoCloseable {
    
	private static final Cleaner cleaner = Cleaner.create();
    
    // Resource that requires cleaning. Must not refer to Room!
    private static class State implements Runnable {
        int numJunkPiles; // Number of junk piles in this room
        
        State(int numJunkPiles) {
            this.numJunkPiles = numJunkPiles;
        }
        
        // Invoked by close method or cleaner
        @Override 
        public void run() {
            System.out.println("Cleaning room");
            numJunkPiles = 0;
        }
	}
    
    // The state of this room, shared with our cleanable
    private final State state;
    
    // Our cleanable. Cleans the room when it’s eligible for gc
    private final Cleaner.Cleanable cleanable;
    
    public Room(int numJunkPiles) {
        state = new State(numJunkPiles);
        cleanable = cleaner.register(this, state);
    }
    
    @Override 
    public void close() {
        cleanable.clean();
    }
}
```

静态内部类`State`持有清洁器清洁房间所需的资源。在这个例子中，资源就是字段`numJunkPiles`，表示房间中的混乱程度。更为现实的情况，它可以是个`final long`字段，包含着一个指针，指向了本地对端。`State`实现了`Runnable`，其`run`方法至多会被`Cleanable`调用一次，`Cleanable`则是我们在`Room`构造方法中将`State`实例注册到清理器上所得到的。对`run`方法的调用会被两个动作所触发：通常，它会被`Room`的`close`方法调用，`close`方法又会调用`Cleanable`的`clean`方法。如果在`Room`实例可以被垃圾回收时，客户端没有调用`close`方法，那么清理器就会（希望如此）调用`State`的`run`方法。

`state`实例不持有对它的`Room`实例的引用，这一点很重要。如果它持有引用，那么它会创造一个死循环，阻止`Room`实例被垃圾收集器回收（以及自动清理）。因此，`State`必须是一个静态内部类，因为非静态内部类包含对其外部类实例的引用(item 24)。同样不建议使用`lambda`，因为它们可以很容易地捕获对外部类对象的引用。 

如前所述，`Room`的清理器只用作安全网。如果客户端在`try-with-resource`块中完成了所有的`Room`实例化动作，那么自动化清理就永远不需要了。如下行为良好的客户端演示了该行为：

``` java
public class Adult {
    public static void main(String[] args) {
        try (Room myRoom = new Room(7)) {
        	System.out.println("Goodbye");
        }
    }
}
```

如你所想，运行`Adult`程序会打印出`Goodbye`，然后是`Cleaning room`。不过，下面这个有问题的程序呢，它永远不会清理房间？

```java
public class Teenager {
    public static void main(String[] args) {
    	new Room(99);
    	System.out.println("Peace out");
    }
}
```

你可能觉得它会打印出`Peace out`，然后是`Cleaning room`，不过在我的机器上，它永远不会打印出`Cleaning room`；它只不过退出了而已。这就是我们之前提到的不可预测性。**Cleaner**规范说到：『在`System.exit`时清理器的行为是特定于实现的』。没有人可以保证清理动作是否会被调用。虽然规范没这么说，但对于正常的程序退出来说就是如此。在我的机器上，往`Teenager`类的`main`方法里添加一行` System.gc()`，能让程序在退出前打印“`Cleaning room`”，但是不保证在你的机器上就能看到相同的结果。

总结一下，不要使用清理器，或者说不要在Java 9之前的版本使用终结器，除非将其作为安全网或是用来终止不重要的本地资源。即便如此，也请小心其不确定性和性能影响。

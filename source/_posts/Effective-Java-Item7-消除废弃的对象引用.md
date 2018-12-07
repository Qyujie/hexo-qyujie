---
title: Effective Java Item7 消除废弃的对象引用
date: 2018-12-07 17:11:59
tags:
    - Java
    - Effective Java
categories:
    - Java
    - Effective Java
    - chapter 2
---

如果您从使用手动管理内存的语言(如C或c++)切换到使用垃圾收集语言(如Java)，那么你作为程序员的工作就会变得容易得多，因为你的对象在使用完后会自动被回收。 当你第一次体验这种编程的时候，它看起来就像是魔术一般。它很容易给人留下这样的印象：你不必考虑内存管理，但这并不完全正确。 
<!-- more -->

思考下面这个简单的堆栈实现。

``` java
// Can you spot the "memory leak"?
public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;
    
    public Stack() {
   		elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }
    
    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;
    }
    
    public Object pop() {
        if (size == 0)
        	throw new EmptyStackException();
        return elements[--size];
    }
    
    /**
    * Ensure space for at least one more element, roughly
    * doubling the capacity each time the array needs to grow.
    */
    private void ensureCapacity() {
        if (elements.length == size)
        	elements = Arrays.copyOf(elements, 2 * size + 1);
    }
}
```

上述程序并没有明显的错误（不过请查看Item 29来了解更加通用的版本）。你可以不断测试该程序，程序也会顺利通过每个测试，不过有一个潜伏的问题。大致来说，该程序存在一处『内存泄露』，其性能会逐步降低，这是因为不断增加的垃圾收集器活动与内存占用问题。在极端情况下，这种内存泄露会导致磁盘分页，甚至会因`OutOfMemoryError`造成程序失败，不过这种失败的情况是非常少见的。

所以，内存泄漏在哪呢？如果栈不断增长，然后再收缩，那么出站的数据并不会被垃圾回收。即便使用了栈的程序不再引用他们亦如此。这是因为堆栈维护着对他们的***过时的引用***。废弃的引用指的是永远不会被解引用的引用。在该示例中，位于元素数组『活动部分』之外的任何引用都是废弃的。活动部分包含了索引小于`size`的元素。

垃圾收集语言中的内存泄露（更恰当的叫法是无意的对象保持）是非常不易察觉的。如果对象引用被无意保持了，那么不仅该对象会从垃圾收集中排除出去，该对象所引用的其他对象也会被排除出去，以此类推。即便只有少量的对象引用被无意保持了，造成的后果就是会有很多、很多对象会从垃圾收集中排除出去，这会对性能造成很严重的影响。

这类问题的解决方案很简单：一旦引用变成废弃状态，立刻将其置为`null`。对于我们的`Stack`类来说，如果元素从栈中弹出，那么对其的引用就变成废弃状态了。`pop`方法的正确版本如下代码所示：

``` java
public Object pop() {
    if (size == 0)
    	throw new EmptyStackException();
    Object result = elements[--size];
    elements[size] = null; // Eliminate obsolete reference
    return result;
}
```

将过时的引用指向`null`的另一个好处是，如果他们后面被错误地取消引用了，程序会立刻报`NullPointerException`的错误，而不是静静地做错误的事情。尽可能快地发现程序的错误总是有益的。

当程序员初次遇到这个问题时，他们会采取矫枉过正的措施：当程序使用完对象后，会将每个对象引用都设为`null`。这么做既没必要，也不值得；它会毫无必要地将程序搞乱。取消对象引用应该是例外而不是规范。消除废弃引用的最佳方式是让包含了引用的变量离开作用域。如果在最小的作用域内定义每个变量，那么这就是自然而然的事情了（Item 57）。

那么应该在何时将引用置为`null`呢？`Stack`类的哪个地方使得它容易出现内存泄露问题呢？简而言之，它来管理自己的内存。存储池包含了`elements`数组的元素（对象引用单元，而非对象自身）。位于数组活跃部分中的元素（如之前所定义的那样）会被分配，而数组其他部分的元素则是空闲的。垃圾收集器并不知晓这一点；对于垃圾收集器器来说，`elements`数组中的所有对象引用都是有效的。程序员可以与垃圾收集器就这个事实进行高效的沟通，方式是当数组元素进入到非活跃部分中时就立刻将其手工置为`null`。

一般来说，**当类管理自己的内存时，程序员应该警惕内存泄露问题**。当元素释放时，包含在该元素中的任何对象引用都应该被置为`null`。

**另一个常见的内存泄漏源是缓存**。一旦将对象引用放入缓存中，就很容易忘记它的存在，然后当缓存失效后它就会一直在那儿。这里有几个解决该问题的办法。如果实现了一个缓存，只要缓存外有引用指向缓存的键，缓存就处于有效状态时，那么缓存就可以使用`WeakHashMap`来表示;当变为废弃状态时，缓存中的条目就会自动移除。请记住，只有在缓存条目的生命周期是由对其键（而非值）的外部引用所决定时，`WeakHashMap`才是适合的。

更为常见的情况则是，缓存条目的生命周期不是那么明确的，随着时间的流逝，缓存条目的价值也变得越来越低。在这些情况下，我们应该适时清理那些不再使用的缓存条目。这可以通过后台线程（也许是`ScheduledThreadPoolExecutor`）来实现，或是在将新的条目添加到缓存中时顺便完成。`LinkedHashMap`类通过其`removeEldestEntry`方法可以简化后者的操作。对于更加复杂的缓存来说，你可能需要直接使用`java.lang.ref`。

**内存泄漏的第三个常见来源是监听器和其他回调**。如果实现了一个API，客户端在该API上注册了回调，但却没有显式取消注册，那么他们就会不断累积，除非采取一些行动。确保回调会立刻被垃圾收集的一种方式是只存储对其的弱引用，比如说，在`WeakHashMap`中只将其以键的形式存储。

由于内存泄露通常并不会导致立刻失败，因此它们可能会在系统中保留多年。他们通常是通过精心的代码检查或是借助于调试工具（**heap profiler**）的帮助才能发现。因此，你需要学习如何在内存泄露出现前就能预测到问题，并防止他们发生。


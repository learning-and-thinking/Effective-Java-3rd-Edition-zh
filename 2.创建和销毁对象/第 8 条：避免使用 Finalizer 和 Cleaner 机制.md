# 第 8 条：避免使用 Finalizer 和 Cleaner 机制

**Finalizer 具有不可预测性，通常比较危险，并且一般情况下也没有必要使用。** Finalizer 的使用会导致程序行为不稳定，降低性能，还会带来一些可移植性问题。Finalizer 只在很少情况下有用，在本条目的后面部分会做介绍，但根据经验，你应该避免使用它们。从 Java 9 开始，finalizer 已经被弃用了，然而在一些 JAVA 库中依然可以看到它们的身影。Java 9 中替代 finalizer 的是 cleaner。**Cleaner 比 finalizer 危险性低一些，但仍然是不可预测的，会使程序运行缓慢，通常情况下依然没必要使用**。

不要把 Java 的 finalizer 或 cleaner 与 C++ 中的析构函数（destructor）当成类似的机制。在 C++ 中，析构函数是回收一个对象所占用资源的标准方法，而且也是是构造函数所必需的对应物。在 Java 中，当一个对象变得不可达时，垃圾回收器会回收与该对象相关联的存储空间，不需要程序员做专门的工作。 C++ 的析构函数也可以回收其他非内存资源，而在 Java 中，一般用 `try-with-resources` 或者 `try-finally` 块达到同样的目的（[第 9 条][item9]）。

Finalizer 和 Cleaner 的缺点之一是不能保证它们被及时执行[JLS, 12.6]。从一个对象变得不可达开始到它的 Finalizer 或 Cleaner 被执行，所花费的时间是任意长的。这意味着**不应该在Finalizer 和 Cleaner 中执行任何对时序要求严格（time-critical）的任务**。比如，依靠 Finalizer 和 Cleaner 关闭已打开文件的操作就是严重错误，因为打开文件的描述符是一种很有限的资源。由于 JVM 会延迟执行 Finalizer 和 Cleaner ，所以大量文件会保留在打开状态，当一个程序不能再打开文件时，它可能会运行失败。

及时地执行 finalizer 和 cleaner 正是垃圾回收算法的一个主要功能，然而它在不同的 JVM 实现中差别很大。依赖于 finalizer 或 cleaner 执行及时性的程序的表现也可能发现同样的变化。这样的程序在你测试用的 JVM 平台上运行得非常好，而在你最重要的客户的 JVM 平台上却根本无法运行，这种情况是完全有可能发生的。

延迟终结过程不仅仅只是一个理论上的问题。给一个类添加 finaler 会任意地延迟该类实例的回收过程。我的一位同事在调试一个长期运行的 GUI 应用时，程序莫名其妙报了`OutOfMemoryError`的错误然后死掉。我们分析程序死掉的时期发现，应用程序的 finalizer 队列中有成千上万个图形对象（graphics objects）在等着被终结和回收。不幸的是，这个 finalizer 线程的优先级比该程序的其它线程的优先级低，所以这些对象被终结的速度比不上它们进入终结状态的速度。Java 语言规范并不能保证哪个线程会执行 finalizer ，所以除了避免使用 finalizer 之外，没有别的简便方法来避免这类问题。在这方面 cleaner 比 finalizer 好一点，因为类的作者可以控制自己的 cleaner 线程，但是 cleaner 依然会在垃圾回收器的控制下在后台运行，所以也不能保证及时清理。

Java 语言规范不仅不能保证 finalizer 或 cleaner 的及时执行，它甚至都不能保证它们会被执行。当一个程序终止时，某些已经无法访问的对象的 finalizer 却根本没有执行，这也是完全有可能的情况。所以，**永远不要依靠 finalizer 或 cleaner 来更新重要的持久状态**。比如，使用 finalizer 或 cleaner 释放共享资源（比如数据库）上的持久锁（persistent lock），很容易让整个分布式系统垮掉。

不要被 `System.gc`  和 `System.runFinalization` 这两个方法迷惑。它们可能会提高 finalizer 或 cleaner 被执行的几率，但也不保证它们一定会被执行。有两种方法曾经作出这样的保证：`System.runFinalizersOnExit` 和它臭名昭著的兄弟 `Runtime.runFinalizersOnExit` 方法。这些方法都有致命缺陷，并且已经被废弃了数十年[ [ThreadStop](#ThreadStop)]。

Finalizer 存在另外一个问题，在终结过程中抛出的未被捕获的异常会被忽略，并且对象的终止过程也会终止 [JSL, 12.6]。未捕获的异常会使其他对象处于破坏（a corrupt state）的状态。如果其他线程试图使用~~了~~这些被破坏的对象，将会导致任意不可预测的行为。通常，未被捕获的异常会终止线程并打印出栈轨迹（stack trace），但发生在 finalizer 中却不会这么做——它甚至不会打印出任何一条警告信息。Cleaner 没有这个问题，因为使用 Cleaner 的库可以控制它的线程。

**使用  finalizers 或 cleaners 会导致严重的性能损失**。在我的机器上创建一个简单的 `AutoCloseable` 对象，使用 `try-with-resources` 来关闭它并让垃圾回收器回收它所用的时间大约是 12ns。使用 finalizer 反而使时间达到了 550 ns。换句话说，使用 finalizer 创建并销毁对象慢了大约 50 倍。主要是因为 finalizer 抑制了高效的垃圾回收。如果你使用 cleaner 来清除类的所有实例，其速度和 finalizer 差不多（在我的机器上清除每个实例所花费的时间大约是 500 ns），但如果你只是把它们作为安全网（safety net）来使用，cleaner 的速度将会快的多。在这种情况下，创建、清除并销毁对象在我的机器上花了 66 ns，这意味着如果你不使用安全网，为了保险你要付出 5 倍（不是50）的代价。

**Finalizer 还有一个严重的安全问题：它们会打开你的类致使其受到 finalizer 攻击**（finalizer attacks）。Finalizer 攻击背后的思想很简单：如果从构造器或它的序列化等价物 —— `readObject` 和 `readResolve` 方法（[第 12 章][Chapter12]）中抛出一个异常，那么恶意子类的 Finalizer 就可以在部分构造的对象上运行，而这些对象本应该夭折。这个 finalizer 可以在静态字段中记录一个该对象的引用，从而防止对该对象进行垃圾回收。一旦记录了格式错误的对象（malformed object），在这个早就不该存在的对象上任意调用方法就会变得非常简单。**从构造器中抛出异常应该足以阻止对象的产生；但在 finalizer 存在的情况下，却并非如此。 **这样的攻击会导致可怕的结果。final 类避免了Finalizer 攻击，因为没人能编写 final 类的恶意子类。**为了保护非 final 类不受 finalizer 攻击，可以编写一个不执行任何操作的 finalize 方法 **。

那么对于对象中封装的资源确实需要被终止的类，比如文件或线程，除了编写其 finalizer 或 cleaner，你应该怎么做呢？**可以让你的类实现 `AutoCloseable` 接口**，并且要求该类的客户端在每个实例不再被需要时调用 `close` 方法，通常使用 `try-with-resources` 来确保即使出现异常时亦能正常终止（[第 9 条][item9]）。有个值得注意的细节，实例必须持续跟踪自己是否被关闭：`close` 方法必须在一个域中记录该对象不再是有效的，如果该对象中的其他方法在对象被关闭之后调用，它们就必须检查这个域，并抛出 `IllegalStateException` 异常。

那么，finalizer 和 cleaner有没有什么用处呢？它们可能有两种合法用途。第一种用途是当对象的所有者忘记调用前面段落中建议使用的 `close` 方法的时，它们可以作为安全网（safety net）。虽然不能保证 finalizer 或 cleaner 会及时运行（或者根本不运行），但在客户端无法正常结束操作的情况下，迟一点释放资源总比永远不释放要好。如果你正考虑编写这样一个安全网 finalizer，建议你三思是否值得为这种保护付出代价。一些 Java 库中的类，比如 `FileInputStream`、`FileOutputStream`、`ThreadPoolExecutor` 和 `java.sql.Connection` 就把 finalizer 作为了安全网。

Cleaner 的第二种合法用途与对象的本地对等体有关（native peer）。本地对等体是一个本地对象（native object，非 Java 对象），普通对象通过本地方法（native method）委托给一个本地对象。因为本地对等体不是普通对象，垃圾回收器并不知道它的存在，所以当它的 Java 对等体被回收的时候它不会被回收。假设程序性能是可接受的而且本地对等体不含有关键资源，这样的情况下，finalizer 或 cleaner 或许能派上用场。如果性能不允许或者本地对等体持有必须被及时回收的资源，你应该为类添加前面段落中建议的 `close` 方法。

Cleaner 使用起来有些棘手。下面用一个简单的 `Room` 类演示一下。假设 `Room` 的对象必须在它被回收前清除。`Room` 类实现了 `AutoCloseable` 接口；事实上，它的自动清除安全网使用的仅仅是 Cleaner 的一个具体实现。与 Finalizer 不同，Cleaner 不会污染类的公共 API：

```java
// 一个实现了 AutoCloseable 接口使用 Cleaner 作为安全网的类
public class Room implements AutoCloseable {
    private static final Cleaner cleaner = Cleaner.create();

    // 需要清除的资源，不要牵涉到 Room 对象!
    private static class State implements Runnable {
        int numJunkPiles; // 房间里的垃圾堆数量
        State(int numJunkPiles) {
            this.numJunkPiles = numJunkPiles;
        }

        // 由 close 方法或 Cleaner 调用
        @Override
        public void run() {
            System.out.println("Cleaning room");
            numJunkPiles = 0;
        }
    }
    // 这个房间的状态, 与我们定义的 cleanable 共享
    private final State state;
    // 我们定义的 cleanable. 当可以获得垃圾回收器时清理房间
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

静态内部类 `State` 含有 Cleaner 清理房间所需的资源。本例中，使用 `numJunkPiles` 变量简单的表示房间的脏乱程度。更确切地说，它可能包含了一个指向本地对等体的 `long` 常量指针。`State` 类实现了`Runnable` 接口并且它的 `run` 方法只能被 `Cleanable` 对象调用一次，`Cleanable` 对象是在 `Room` 的构造器中注册 `State` 实例时获得的 。对 `run` 方法的调用将由以下两种情况之一触发：通常是通过调用 `Room` 的 `close` 方法，继而调用 `Cleanable` 的 `clean` 方法 来触发。其次，当一个 `Room` 实例适合进行垃圾回收时，若客户端没有调用它的 `close` 方法，那么 Cleaner 将（有望）调用 `State` 的 `run` 方法。

一个 `State` 实例不应该引用其对应的 `Room` 实例，这一点至关重要。否则，将会出现循环引用使 得`Room` 实例变得不可被回收（也不会被自动清理）。因为非静态内部类包含其外部类实例的引用（[第 24 条][Item24]），因此，`State` 必须是静态内部类。同样，使用 lambda 表达式也是不可取的，因为它们可以轻易捕获外部类对象的引用。

正如前文所述，`Room` 的Cleaner 仅仅是用来作为安全网的。如果客户端把所有 Room 对象的实例化过程包含在 try-with-resource 块中，就不再需要自动清理了。下面这个功能良好的客户端演示了这种方式：

```java
public class Adult {
    public static void main(String[] args) {
        try (Room myRoom = new Room(7)) {
            System.out.println("Goodbye");
        }
    }
}
```

如你所愿，Adult 程序运行后会先打印出“Goodbye”，随后会打印“Cleaning room”。那么对于那些“不打扫自己的房间”，功能不正常的程序是怎么样的呢？

```java
public class Teenager{
    public static void main(String[] args){
        new Room(99);
        System.out.println("Peace out");
    }
}
```

你或许期望它会先打印出“Peace out”，随后打印出“Cleaning room”，然而在我的机器上，它直接退出，从没有打印过“Cleanning room”。这就是我们之前谈到的不可预测性。Cleaner 的规范指出：“Cleaner 在 `System.exit` 方法执行期间的行为是特定于实现的。关于是否调用清除操作并没有保证。”虽然规范中没有提到，但普通程序退出时也是如此。 在我的机器上，给 `Teenager` 的 `main` 方法添加`System.gc()` 这一行就可以让它在退出程序前打印出 `Cleaning room`  ，但不保证能在你的电脑上看到同样的情形。

总之，不要使用 Cleaner 或 Java 9 之前的 Finalizer ，除非作为安全网或用来终止非关键的本地资源。即使这样，也要当心不确定性和性能影响。

<p id="Gamma95">[ThreadStop] Why Are Thread.stop, Thread.suspend, Thread.resume and Runtime.runFinalizersOnExit Deprecated? 1999. Sun Microsystems. <https://docs.oracle.com/javase/8/docs/technotes/guides/concurrency/threadPrimitiveDeprecation.html>

[item9]: ./第%209%20条：try-with-resources%20优于%20try-finally.md "第 9 条：try-with-resources 优于 try-finally"
[item24]: url "在未来填入第 24 条的 url，否贼无法进行跳转"
[chapter12]: url "在未来填入第 12 章的 url，否则无法进行跳转"
[chapter21]: url "在未来填入第 21 章的 url，否则无法进行跳转"

> 翻译：Injer
>
> 校对：Angus

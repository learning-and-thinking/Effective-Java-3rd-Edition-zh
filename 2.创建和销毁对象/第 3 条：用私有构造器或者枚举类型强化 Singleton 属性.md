## 第 3 条：用私有构造器或者枚举类型强化 Singleton 属性

单例（singleton）指仅仅被实例化一次的类 [[Gamma95](#Gamma95)]。单例对象通常表示无状态对象，如函数（function [第 24 条](item24)）或本质上唯一的系统组件。**使类成为单例会使测试它的客户端变得困难**，因为不可能用模拟实现代替单例，除非它实现一个充当其类型的接口。 

实现单例的方法有两种。这两者都基于保持构造器私有并导出公有静态成员以提供对唯一实例访问的方式。在第一种方法中，公有静态成员是个 final 域：

```java
// 公有域方法实现 singleton
public class Elvis {
    public static final Elvis INSTANCE = new Elvis();
    private Elvis() { ... }
    public void leaveTheBuilding() { ... }
}
```

私有构造器只会被调用一次，以初始化公有静态 final 域 `Elvis.INSTANCE`。缺少公有或受保护的构造器可以保证全局唯一性：一旦 `Elvis` 类被初始化，`Elvis` 实例只会存在一个——不多也不少。客户端所做的任何事情都无法改变这一点，但要提醒一下：享有特权的客户端可以借助 `AccessibleObject.setAccessible` 方法，通过反射机制（[第 65 项][item65]）调用私有构造器。如果需要防范这种攻击，可以修改构造器，让它在被要求创建第二个实例时抛出异常。 

在实现单例的第二种方法中，公有成员是个静态工厂方法：

```java
// 静态工厂方法实现 singleton
public class Elvis {
    private static final Elvis INSTANCE = new Elvis();
    private Elvis() { ... }
    public static Elvis getInstance() { return INSTANCE; }

    public void leaveTheBuilding() { ... }

}
```

对 `Elvis.getInstance` 的所有调用都会返回相同的对象引用，不会创建其他的 `Elvis` 实例（上述的提醒仍然适用）。

公有域方法的主要优点是 API 能清楚地表明该类是单例：公有静态域是 final 的，因此它始终包含相同的对象引用。第二个优点是它更简单。

静态工厂方法的优势之一在于，它提供了灵活性，在不更改 API 的情况下，你可以改变类是否应该是单例的想法。工厂方法返回唯一的实例，但可以对它进行修改，例如，改成为每个调用它的线程返回一个单独的实例。第二个优势是，如果应用需要，你可以编写通用的单例工厂（generic singleton factory [第 30 项](item30)）。使用静态工厂的最后一个优点是，方法引用（method reference）可以被当作 supplier 来使用，例如 `Elvis::instance` 即是一个 `Supplier<Elvis>`。除非与其中一个优点相关，否则公有域方法更可取。

为了使利用其中任意一种方法实现的单例类是可序列化的（serializable [第12章](chapter12)），仅仅在其声明添加 `implements Serializable` 是不够的。要维护并保证单例，需要声明所有实例字段为 `transient` 并提供 `readResolve` 方法（[第 89 项](item89)）。否则，每次一个已序列化的实例被反序列化时，都会创建一个新实例。比如，在我们的示例中，将导致出现 “假冒的 `Elvis`”。为了防止这种情况发生，需要在 `Elvis` 类中加入下面这个 `readResolve` 方法：

```java
// readResolve 方法保护 singleton 性质
private Object readResolve() {
    // 返回真正的 Elvis， 
    // 让垃圾回收器处理 Elvis 的模仿者
    return INSTANCE;
}
```

实现单例的第三种方法是声明一个单元素的枚举类（single-element enum）： 

```java
// Enum singleton - 首选的方法
public enum Elvis {
    INSTANCE;
    public void leaveTheBuilding() { ... }
}
```

这种方法类似于公有域方法，但它更简洁，并且无偿提供了序列化机制，即使面对复杂的序列化或反射攻击，也能提供防止多实例化的严格保障。这种方法可能会让人感觉有点不自然，但**单元素的枚举类型通常是实现单例的最佳方法**。请注意，如果你的单例必须继承除 `Enum` 之外的超类，就不能使用该方法（即使可以声明一个枚举来实现接口）。 

<p id="Gamma95">[Gamma95] Gamma,	Erich,	Richard	Helm,	Ralph	Johnson,	and	John	Vlissides.	1995. Design	Patterns:	Elements	of	Reusable	Object-Oriented	Software.	Reading, MA:	Addison-Wesley.	ISBN:	0201633612. </p>



[item24]: url "在未来填入第 24 条的 url，否则无法跳转"
[item30]: url "在未来填入第 30 条的 url，否则无法跳转"
[item65]: url "在未来填入第 65 条的 url，否则无法跳转"
[item89]: url "在未来填入第 89 条的 url，否则无法跳转"
[chapter12]: url "在未来填入第 12 张的 url，否则无法跳转"



> 翻译：Angus
>
> 校对：Inno
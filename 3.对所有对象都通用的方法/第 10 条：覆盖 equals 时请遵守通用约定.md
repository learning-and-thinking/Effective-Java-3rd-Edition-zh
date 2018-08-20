# 第 10 条：覆盖 equals 时请遵守通用约定

覆盖 equals 方法似乎很简单，但是有很多方式会导致错误，并且后果会很严重。防止问题出现的最简单的方式就是不要覆盖 equals 方法，在这种情况下，类的每个实例都只等于自身。如果满足一下任何条件，这样做就是对的：

+ **类的每个实例本质上都是唯一的**。 对于代表活动实体而不是值（value）的类来说确实如此，例如 Thread。Object 提供的 equals 方法实现对于这些类来说正是正确的行为
+ **不需要为类提供“逻辑相等（logical quality）”的测试**。例如，java.util.regex.Pattern 覆盖了 equals 方法，以检查两个 Pattern 实例是否确切地代表了相同的正则表达式，但是设计者并不认为客户端会需要或者想要这样的功能。在这种情况下，从 Object 继承的 equals 实现是理想的。
+ **一个超类已经覆盖了 equals 方法，并且超类的行为适用于这个类**。例如，大多数 Set 实现从 AbstractSet 继承 equals 实现，List 实现从 AbstractList 继承 equals 实现，Map 实现从 AbstractMap  继承 equals 实现。
+ **类是私有或者包级私有的，并且你确定类的 equals 方法绝不会被调用**。如果你是极度规避风险的，你可以覆盖 equals 方法，以确保他不会被意外调用：

```java
@Override
public boolean equals(Object o) {
    throw new AssertionError(); // 该方法绝不会被调用
} 
```

那么，什么时候适合覆盖 equals 方法呢？当一个类具有逻辑相等的概念时（有别于单纯的对象等同），并且超类没有覆盖 equals 方法。这通常是“值类（value class）”的情况。值类是指，诸如 Interger 或 String 这种只是代表一个值的类。使用 equals 方法比较值对象的引用的程序员，期望找出它们是否在逻辑上相等，而不是它们是否引用了同一对象。覆盖 equals 方法不仅仅对满足程序员的预期是必要的，而且还能使实例充当映射表（map）的键（key）或者集合（set） 的元素，使映射表或者集合表现出预期的行为。

有一种值类不需要覆盖 equals 方法，该类使用实例控制（[第 1 条][item1]）来确保每个值最多只有一个对象存在。枚举类型（[第 34 条][item34]）就属于这种类。对这种类而言，逻辑相等与对象等同是一样的，所以 Object 的 equals 方法等同于逻辑上的 equals 方法。

当你覆盖了 equals 方法，你一定依遵循它的通用约定。下面是约定的内容，来自 Object 规范：

equals 方法实现了等价关系（equivalence relation）。它有下面这些属性：

+ **自反性（reflexive）**。对任何非空引用值 x，`x.equals(x)` 一定返回 true。
+ **对称性（symmetric）**。对任何非空引用值 x 和 y，当且仅当 `y.equals(x)` 返回 true 时 `x.equals(y)` 一定返回 true。
+ **传递性（transitive）**。对任何非空引用值 x、y、z，如果 `x.equals(y)` 返回 true 并且 `y.equals(z)` 返回 true，那么 `x.equals(z)` 一定返回 true。
+ **一致性（consistent）**。对任何非空引用值 x 和 y，如果 equals 的比较操作在对象中所用的信息没有被修改，多次调用 `x.equals(y)` 就会一致地返回 true 或者一致地返回 false。
+ 对任何非空引用值 x，`x.equals(null)` 一定返回 false。

除非你在数学上有所倾斜，否则这看起来有点恐怖，但是不要忽视它。如果违犯了它，你可能会发现你的程序行为异常或崩溃，并且很难确定故障的根源。用 John Donne 的话说，没有哪个类是孤立的。一个类的实例被频繁的传递给另一个类。许多类，包括所有集合类（collection class），都依赖于传递给它们的对象是否遵循 equals 约定。

现在以已经意识到了违反 equals 约定的危险了，接下来让我们详细讨论一下这些约定。好消息是，尽管这些约定看起来很吓人，但实际上并不是很复杂。一旦你理解了它，那么遵循这些约定就不是什么难事了。

那么，什么是等价关系呢？不严格地讲，它是把一组元素分割成子集的操作符，这些子集的元素被视为是彼此相等的。这些子集被称为等价类（equivalence class）。要使 equals 方法有用，每个等价类中的所有元素必须可以从用户的角度进行互换。现在让我们依次检查五个要求：

* **自反性（reflexivity）**—— 第一个要求仅仅说明一个对象必须与它自身相等。很难想象会无意间违反这个约定。如果你违反了它，并在之后为集合（collection）添加了该类的实例，该集合的 contains 方法可能会告诉你，该集合不曾含有你刚刚添加的实例。
* **对称性（symmetry）**—— 第二个要求说明的是任何两个对象对于“它们是否相等”的问题必须保持一致。于第一个约定不同，若无意间违反了该约定，倒不是很难想象。例如，考虑下面这个类，这个类实现了一个不区分大小写的字符串。字符串的大小由 toString 方法保存，但在 equals 方法的比较操作中被忽略：

```java
// 违反对称性！
public final class CaseInsensitiveString {
    private final String s;

    public CaseInsensitiveString(String s) {
        this.s = Objects.requireNonNull(s);
    }

    // 违反对称性！
    @Override
    public boolean equals(Object o) {
        if (o instanceof CaseInsensitiveString)
            return s.equalsIgnoreCase(((CaseInsensitiveString) o).s);
        if (o instanceof String)    // 单向互通！
            return s.equalsIgnoreCase((String) o);
        return false;
    }
    // ...
    // 其余省略
}
```

在这个类中，出于好意的 equals 方法天真地尝试与普通的字符串（String）对象进行互操作。假设我们有一个不区分大小写的字符串和一个普通的字符串：

```java
CaseInsensitiveString cis = new CaseInsensitiveString("Polish");
String s = "polish";
```

正如预想的那样，`cis.equals(s)` 返回 `true`。这里的问题在于，虽然 CaseInsensitiveString 类中的 equals 方法知道普通的字符串（String）对象，但是 String 类中的 equals 方法却对不区分大小的字符串浑然不知。因此，`s.equals(cis)` 返回 `false`，显然违反了对称性。假设你把一个不区分大小写的字符串放进了集合里：

```java
List<CaseInsensitiveString> list = new ArrayList<>();
list.add(cis);
```

此时 `list.contains(s)` 方法会返回什么结果呢？谁知道呢？在 OpenJDK 的当前实现中，它碰巧返回 `false`，但这只是这个特定实现得出的结果而已。在其它实现中，它可以很容易地返回 true 或者抛出运行时（runtime）异常。**一旦你违反了 equals 约定，当其它对象面对你的对象时，你就不知道这些对象的行为会怎样**。

要解决这个问题，只需从 equals 方法中去除与 String 对象进行互操作这个不明智的尝试即可。一旦这样做了，就可以把该方法重构为单个返回语句：

```java
@Override
public boolean equals(Object o) {
    return o instanceof CaseInsensitiveString
        && ((CaseInsensitiveString) o).s.equalsIgnoreCase(s);
}
```

**传递性（Transitivity）**—— equals 约定的第三个要求说的是如果一个对象等于第二个对象，并且第二个对象等于第三个对象，那么第一个对象一定与第三个对象相等。同样，不难想象无意中违反这一要求。考虑子类的情形，该子类将新的值组件（value component）添加到其超类中。换句话说，该子类添加了一条影响 equals 方法比较的信息。让我们从一个简单的不可变的二维整数型 Point 类开始：

```java
public class Point {
    private final int x;
    private final int y;

    public Point(int x, int y) {
        this.x = x;
        this.y = y;
    }

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Point)) return false;
        Point p = (Point) o;
        return p.x == x && p.y == y;
    }
    // ...    
    // 其余省略 
} 
```

假设你想扩展这个类，为一个点添加颜色的概念：

```java
public class ColorPoint extends Point {
    private final Color color;

    public ColorPoint(int x, int y, Color color) {
        super(x, y);
        this.color = color;
    }
    // ...
    // 其余省略
}
```

equals 方法应该是什么样子呢？如果完全省略不写，那么继承自 Point 的实现以及颜色信息会在 equals 的比较中被忽略。虽然这样做并没有违反 equals 约定，但是这样明显是不能被接受的。假设你写了一个 equals 方法，当且仅当它的参数是另一个具有相同位置和颜色的颜色点时，才会返回 true。

```java
// 违反对称性！
@Override
public boolean equals(Object o) {
    if (!(o instanceof ColorPoint)) return false;
    return super.equals(o) && ((ColorPoint) o).color == color;
}
```

这个方法的问题在于，当你比较一个普通点与有色点时可能会得到不同的结果，反之亦然。普通点与有色点比较会忽略颜色，而有色点与普通点比较会始终返回 false，这是因为参数类型是不正确的。为了更具体的描述这个问题，让我们创建一个普通点和一个颜色点：

```java
Point p = new Point(1, 2);
ColorPoint cp = new ColorPoint(1, 2, Color.RED);
```

然后，`p.equals(cp)` 会返回 true，而 `cp.equals(p)` 会返回 false。你可能会在 “混合比较” 时，通过让 ColorPoint.equals 忽略颜色来修复这个问题：

```java
// 违反传递性
@Override
public boolean equals(Object o) {
    if (!(o instanceof Point)) return false; 
    // 如果 o 是一个普通点，那么做一个色盲比较
    if (!(o instanceof ColorPoint)) return o.equals(this); 
    // 如果 o 是颜色点，做一个完整的比较
    return super.equals(o) && ((ColorPoint) o).color == color;
}
```

这种方法确实提供了对称性，但却是在破坏传递性的情况下： 

```java
ColorPoint p1 = new ColorPoint(1, 2, Color.RED);
Point p2 = new Point(1, 2);
ColorPoint p3 = new ColorPoint(1, 2, Color.BLUE);
```

现在 `p1.equals(p2)` 和 `p2.equals(p3)` 都返回 true，然而 `p1.equals(p3)` 却返回 false，很明显违反了传递性。前两个比较是 “色盲” 比较，而第三个则是考虑了颜色。

此外，这种方法还会导致无限递归：假设这有两个 Point 的子类，比如 ColorPoint 和 SmellPoint，每个都有这种 equals 方法。然后调用 `myColorPoint.equals(mySmellPoint)` 将会抛出 StackOverflowError 异常。

那么解决方案是什么呢？事实上，这是面向对象语言中等价关系的基本问题。除非您愿意放弃面向对象抽象的好处，否则无法扩展可实例化的类，并在保留 equals 约定的同时添加值组件。

你可能听说，通过在 equals 方法中使用 getClass 测试替代 instanceof 测试，可以在扩展可实例化的类并且添加值组件的同时保留 equals 约定：

```java
// 违反里氏替换原则
@Override
public boolean equals(Object o) {
    if (o == null || o.getClass() != getClass()) return false;
    Point p = (Point) o;
    return p.x == x && p.y == y;
}
```

只有当它们具有相同的实现类的时候，它们才具有相同对象的效果。这似乎并不差，但是结果是不能接受的：一个 Point 子类的实例仍然是 Point，并且它仍然需要作为一个函数，但是如果你采用这种方法，它就不能这样做。假设我们要写一个方法来判断一个点是否在单位圆上。 下面是可以采用的其中一种方法：

```java
// 初始化一个单位圆用来包含单位圆上所有的点
private static final Set<Point> unitCircle = Set.of(
    new Point(1, 0), new Point(0, 1),
    new Point(-1, 0), new Point(0, -1));

public static boolean onUnitCircle(Point p) {
    return unitCircle.contains(p);
}
```

虽然这可能不是实现这个功能最快的方法，不过效果很好。假设你通过某种不添加值组件的方式扩展了 Point，例如让它的构造器记录创建了多少个实例：

```java
public class CounterPoint extends Point {
    private static final AtomicInteger counter = new AtomicInteger();

    public CounterPoint(int x, int y) {
        super(x, y);
        counter.incrementAndGet();
    }

    public static int numberCreated() {
        return counter.get();
    }
}
```

里氏替换原则（Liskov substitution principle）说的是，任何一个类型的重要属性也适用于它的子类型，以便任何为这个类型写的方法可以在它的子类型上工作的一样好[[Liskov87](#Liskov87)]。这是我们之前声明的 Point 的子类（比如 CounterPoint）任然是一个 Point 并且必须作为一个整体的正式声明。但是假设我们为 onUnitCircle 方法传递了一个 CounterPoint 实例。假如 Point 类使用了基于 getClass 的 equals 方法，那么不管 CounterPoint 实例的 x 和 y 坐标是什么， onUnitCircle 方法都将返回 false 。之所以如此，是因为像 onUnitCircle 方法所用的 HashSet 这样的集合，利用 equals 方法检验包含条件，没有任何 CounterPoint 实例与任何 Point 对应。然而，如果在 Point 上使用了恰当的基于 instanceof 的 equals 方法，那么当遇到 CounterPoint 时，相同的 onUnitCircle 方法就会工作的很好。

虽然没有一种令人满意的方法可以扩展一个可实例化的类并添加值组件，但是这有一个不错的权宜之计（workaround）。遵循 [第 18 条](item18) 的建议 ：组合优于继承。不再让 ColorPoint 继承 Point，取而代之的是为 ColorPoint 添加一个私有的 Point 属性，以及一个公有的视图（view）方法（[第 6 条](item6)），该方法返回一个与该有色点处相同位置的普通 Point 对象：

```java
// 在不违反相等合同的情况下添加值组件
public class ColorPoint {
    private final Point point;
    private final Color color;

    public ColorPoint(int x, int y, Color color) {
        point = new Point(x, y);
        this.color = Objects.requireNonNull(color);
    }

    /**
     * 返回此有色点的点视图（point-view）。
     */
    public Point asPoint() {
        return point;
    }

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof ColorPoint)) return false;
        ColorPoint cp = (ColorPoint) o;
        return cp.point.equals(point) && cp.color.equals(color);
    }
    // ...
    // 其余省略
}
```

在 Java 平台库中有一些类的确扩展了可实例化类并添加了一个值组件。例如，java.sql.Timestamp 扩展了 java.util.Date 并且添加了一个 nanoseconds 属性。Timestamp 的 equals 方法的确违反了对称性，如果 Timestamp 和 Date 对象在同一个集合或者其他混合方式中使用的话，会导致不稳定的行为。Timestamp 类有一个免责声明，提醒程序员不要混淆 Date 和 Timestamp 对象。只要你将它们分开就不会惹上麻烦，但除此之外没有可以阻止你混淆它们的方法，并且产生的错误可能难以调试。Timestamp 类的行为是错误的，不应该被效仿。

值得一提的是，你可以为抽象（abstract）类的子类添加值组件而不用违反 equals 约定。通过遵循 [第 23 条](item23) 的建议，“用类层次（class hierarchies）代替标签类（tagged class）” 来获得类层次结构，这是非常重要的。例如，你有一个没有值组件的抽象类 Shape，一个添加了 radius 属性的子类 Circle，以及一个添加了 length 和 width 属性的子类 Rectangle。只要不可能直接创建超类实例，那么之前出现的种种问题就都不会发生。

**一致性（consistency）** —— equals 约定的第四个要求是，如果两个对象是相等的，那么它们一定至始至终保持相等，除非某一个对象或者两个都被修改。换句话说，可变对象可以与不同的对象在不同时刻相等，而不可变对象则不行。当你在写一个类时，应该认真考虑该类是否应该不可变（[第 17 条](item17)）。如果你的结论是应该，那么应确保你的 equals 方法强制满足这一限制：相等的对象永远保持相等，不相等的对象永远保持不相等。

无论一个类是否不可变，**千万不要编写依赖不可靠资源的 equals 方法**，如果你违反了这一禁令，那么满足一致性这一要求将是极其困难的。例如，java.net.URL 的 equals 方法依赖与 URL 相关联的主机的 IP 地址的比较。把主机名翻译成 IP 地址需要访问网络，并且随着时间的推移，不能保证都产生相同的结果。这会使得 URL 的 equals 方法违反 equals 约定，在实践中有可能引发一些问题。URL 的 equals 方法的行为是一个大错误，并且不应被效仿。不幸的是，因为兼容性的需要，这并不能被改变。为了避免这类问题，equals 方法应该只对内存驻留对象执行确定性计算。 

**非空性（Non-nullity）** —— 最后一个要求缺少官方的名称，因此我冒昧地称其为 “非空性”，指的是所有对象一定不能等于 null。虽然很难想象在调用 `o.equals(null)` 时意外地返回 true，但是不难想象它会意外地抛出 NullPointerException 异常。这个通用的约定禁止这样类事情的发生。很多类的 equals 方法都通过一个显示的 null 测试来防止这种情况的发生：

```java
@Override	
public boolean equals(Object o) {
    if (o == null) 
        return false; 
    // ... 
} 
```

这个测试是没必要的。为了测试其参数的等同性，equals 方法必须先把参数转换成适当的类型，以便可以调用它的访问方法（accessor），或者访问它的属性。在进行转换之前，equals 方法必须使用使用 instanceof 操作符检查它的参数是否是正确的类型：

```java
@Override 
public boolean equals(Object o) {
    if (!(o instanceof MyType))
        return	false; 
    MyType mt = (MyType) o; 
    // ...
} 
```

如果缺少了类型检查并且传递给 equals 方法错误的参数类型，那么 equals 方法就会抛出 ClassCastException，这违反了 equals 约定。如果第一个操作数是 null 的话，不论第二个操作数是什么，instanceof 操作符都会指定返回 false [JLS, 15.20.2]。因此，如果传入了一个 null，类型检查就会返回 false，所以你不需要一个显式的空检查。

结合所有要求，得出了以下实现高质量 equals 方法的诀窍：

1. **使用 == 操作符来检查 “参数是否是这个对象的引用”**。如果是的话，返回 true。这只是一个性能优化，但如果比较的代价可能很高的话，那么值得这么做。
2. **使用 instanceof 操作符检查 “参数是否是正确的类型”**。如果不是的话，返回 false。一般说来，所谓 “正确的类型” 是指 equals 方法所在的那个类。有时，它指的是由这个类实现的接口，如果类实现了一个改进 equals 约定的接口，允许在实现了该接口的类之间进行比较，那么就使用接口。集合接口（collection interface，如 Set、List、Map 和 Map.Entry）都具有此属性。
3. **把参数转换成正确的类型**。因为在强制转换前会用 instanceof 测试，所以保证会成功。
4. **对类中的每个“关键（significance）”属性，检查参数的属性是否与这个对象的属性相一致**。如果所有的测试都成功了的话，就返回 true，否则返回 false。如果第 2 步中的类型是一个接口，必须会通过接口方法访问参数的属性；如果该类型是一个类，或许能够直接访问其属性，这要依赖它们的可访问性。

对于基本类型属性，其类型不是 float 或 double 的话，使用 == 操作符来进行比较；对于对象引用属性，递归地调用 equals 方法来进行比较；对于 float 属性，使用静态的 Float.compare(float, float)  方法来进行比较；对于 double 属性，则使用 Double.compare(double, double) 来进行比较。由于 Float.Nan，-0.0f 和类似的 Double 值的存在，对 float 和 double 属性进行特殊处理就变得有必要了。详细信息请查看 JLS 15.21.1 或者 Float.equals 的文档。当你使用静态的 Float.equals 和 Double.equals 方法来比较 float 和 double 属性时，其性能将会很差。对与数组属性，则要把这些准则应用到每个元素上。如果在数组属性中，每个元素都很重要的话，那么使用 Arrays.equals 方法的其中一种来进行比较。

有些对象引用属性可能合法地包含 null。为了防止 NullPointerException 异常的出现，使用静态的 Objects.equals(Object, Object) 方法来检查这些属性是否相等。

对于有些类，例如上面的 CaseInsensitiveString 类，属性比较将比简单的等同性测试更加复杂。如果是这种情况，可能会希望保存该属性的一个 “范式（canonical form）”，这样 equals 方法就可以根据这些范式进行低开销的精确比较，而不是高开销的非精确比较。这个技术对于不可变类是最为适用的（[第 17 条](item17)）。如果对象是可以改变的，那么一定要保持其范式是最新的。

equals 方法的性能可能受属性的比较顺序的影响。为了获得最佳性能，应该先比较最有可能不一样的属性，或开销最低的属性，最理想的情况是兼具这两种的属性。一定不要比较不是对象逻辑状态的属性，例如使用了 synchronize 操作符的 lock 属性。你不需要比较派生属性（derived field），这些属性可以从 “重要属性（significant field）” 中计算出来，但这样做可以提升equals 方法的性能。如果派生属性相当于整个对象的摘要描述的话，那么在比较失败的时，比较此属性将节省你比较实际数据的开销。例如，假设你有一个 Polygon 类，并且你缓存了该区域。如果两个 Polygon 对象有着不相同的区域，那么就没有必要去比较它们的边和顶点。

**当你已经编写完成了你的 equals 方法，你需要问你自己三个问题：它是否是对称的？是否是传递的？是否是一致的？**并且不要只是自问，写一个单元测试来进行检查，除非你是用了 AutoValue 来生成你的 equals 方法，在这种情况下，你可以安全地省略这些测试。如果无法遵循某个特性，找出原因，并相应的修改 equals 方法。当然，你的 equals 方法也必须满足其他两个特性（自反性和非空性），但是这两个特性通常会自动满足。 

在这个简单的 PhoneNumber 类中，展示了根据之前的诀窍构造的 equals 方法： 

```java
// 一个有着典型的 equals 方法的类
public final class PhoneNumber {
    private final short areaCode, prefix, lineNum;

    public PhoneNumber(int areaCode, int prefix, int lineNum) {
        this.areaCode = rangeCheck(areaCode, 999, "area	code");
        this.prefix = rangeCheck(prefix, 999, "prefix");
        this.lineNum = rangeCheck(lineNum, 9999, "line	num");
    }

    private static short rangeCheck(int val, int max, String arg) {
        if (val < 0 || val > max) throw new IllegalArgumentException(arg + ":	" + val);
        return (short) val;
    }

    @Override
    public boolean equals(Object o) {
        if (o == this) return true;
        if (!(o instanceof PhoneNumber)) return false;
        PhoneNumber pn = (PhoneNumber) o;
        return pn.lineNum == lineNum && pn.prefix == prefix && pn.areaCode == areaCode;
    }
    // ...
    // 其余省略
}
```

这里有几个最后的警告：

+ **覆盖 equals 时总要覆盖 hashCode（[第 11 条](item11)）。**
+ **不要企图让 equals 过于智能。**如果你只是简单地测试了属性是否相等，则不难遵循 equals 约定。如果你在寻找等价性方面过于激进，则很容易陷入麻烦。考虑任何形式的别名通常都是一个坏主意。例如，File 类不应该企图把指向同一个文件的符号链接（symbolic link）当作相等的对象来看待。所幸 File 类没有这样做。 
+ **不要将 equals 声明中的 Object 对象替换为其他的类型。** 对程序员来说，写出如下的 equals 方法并不罕见，这会使程序员花费数小时困惑于为什么它不能正确地工作： 

```java
// 参数类型必须是 Object
public boolean equals(MyClass o) {
    //...
}
```

这个的问题在于，这个方法没有覆盖参数是 Object 类型的 Object.equals 方法，相反，它重载了这个方法（[第 52 条](item52)）。提供这样一个 “强类型（strongly typed）” 的 equals 方法是无法接受，即使这是除普通 equals 方法之外的方法，因为它会导致子类中的 Override 注解产生误报以及提供错误的安全感。正如本条目自始自终所描述的，一致地使用 Override 注解将防止你犯这个错误（[第 40 条](item40)）。这个 equals 方法将无法被编译，并且错误信息将告诉你确切错误是什么：

```java
// 无法编译
@Override
public	boolean	equals(MyClass o)	{
    // ...
}
```

编写和测试 equals （和 hashCode）方法是乏味的，并且测试结果也是平常的。手动编写和测试这些方法的一个极佳方案就是使用 Google 的开源的 AutoValue 框架，这个框架会自动地为你生成这些方法，只需要通过在类上标注一个简单的注解即可。在多数情况下，AutoValue 生成的方法在本质上与那些你自己编写的方法是相同的。

IDE 也有生成 equals 和 hashCode 方法的工具， 但是生成的结果代码相比于使用 AutoValue 生成的代码而言更加的冗长且难以阅读，不具备自动跟踪类中的更改，因此需要进行测试。即便如此，让 IDE 生成 equals （和 hashCode）方法通常比手动实现它们更可取，因为 IDE 不会犯粗心的错误，而人类则会犯错。

总之，不要覆盖 equals 方法，除非你不得不这么做。在很多情况下，继承自 Object 的实现完全符合你的需求。如果你确实覆盖了 equals 方法，请确保你比较该类的所有重要属性，并且以保留 equals 约定的所有五项规定的方式对它们进行比较。 



<p id="Liskov87">[Liskov87] Liskov,	B.	1988.	Data	Abstraction	and	Hierarchy.	In	Addendum	to	the Proceedings	of	OOPSLA	’87	and	SIGPLAN	Notices,	Vol.	23,	No.	5:	17–34, May	1988. </p>

[item1]: ../2.创建和销毁对象/第%201%20条：考虑用静态工厂方法代替构造器.md  " 第2章第1条"
[item6]: ../2.创建和销毁对象/第%206%20条：避免创建不必要的对象.md  "第2章第6条"
[item11]: url  "在未来填入第11条的url，否则无法跳转"
[item17]: url  "在未来填入第17条的url，负责无法跳转"
[item18]: url  "在未来填入第18条的url，否则无法跳转"
[item23]: url  "在未来填入第23条的url，否则无法跳转"
[item34]: url  "在未来填入第34条的url，负责无法跳转"
[item40]: url  "在未来填入第40条的url，负责无法跳转"
[item52]: url  "在未来填入第52条的url，负责无法跳转"

---

> 翻译：Inno
>
> 校对：
# 第 11 条：覆盖 equals 方法时必须覆盖 hashCode 方法

**覆盖每个类的 equals() 方法必须覆盖对应的 hashCode() 方法**。如果不这么做，类将会违反 hashCode 的通用约定，从而导致该类无法在基于散列的集合中正常运作，比如 HaspMap 和 HashSet 集合。

下面是约定的内容，摘自 Object 规范：

- 在应用程序执行期间，只要某对象的 equals 方法的比较操作所用到的信息没有修改过，那么多次调用该对象的 hashCode 方法时必须返回同一个值。在同一个应用程序的多次执行过程中，每次执行所返回的值可以不一致。
- 如果两个对象通过 equals( Object ) 方法判定是相等的，那么在两个对象上调用 hashCode 必须产生同样的整数值。
- 如果两个对象通过 equals( Object ) 方法判定是不等的，在每个对象上调用 hashCode 方法可以产生相同的结果。然而，程序员应该知道，让不相等的对象产生不同的结果可以提高散列表的性能。

**当没有覆盖 hashCode 时会违反关键约定的第二条：相等的对象必须具有相等的 hash 码**。根据类的 equals 方法，两个不同的实例有可能逻辑上是相同的，但根据 Object 类的 hashCode 方法，它们仅仅是两个没有任何共同之处的对象。因此，Object 类的 hashCode 方法返回两个看上去是随机的整数，而不是第二个要求约定的那样，返回两个相等的整数。

例如，假设你使用[第十条][Item10]中的 PhoneNumber 类的实例作为 HashMap 的 key 值：

```java
Map<PhoneNumber, String> m = new HashMap<>();
m.put(new PhoneNumber(707, 867, 5309), "Jenny");
```

这种情况下，你可能期望 m.get(new PhoneNumber(707, 867, 5309)) 会返回 “Jenny”，然而事实并非如此，它将会返回 null 。注意，这里涉及了两个 PhoneNumber 实例：一个是插入到 HashMap 中的实例，另一个相等的实例用于（试图用于）检索。PhoneNumber 类没有覆盖 hashCode 方法导致这两个相等的实例拥有不相等的 hash 码，违反了 hashCode 的约定。因此，get 方法有可能从一个不同于 put 方法放入的 hash 桶（hash bucket）中查找这个电话号码。即使这两个实例整好被放到同一个 hash 桶中，get方法也必定会返回 null，因为 HashMap 有一项优化，可以将与每个项相关联的 hash 码缓存起来，如果 hash 码不匹配时，也不必检验对象是否相等。

解决这种问题也很简单，只要给 PhoneNumber 写一个适当的 hashCode 方法即可。那么，hashCode 方法应该是怎样的呢？编写一个合法但不好用的 hashCode 方法没有任何价值。比如下面这个，虽然合法但永远都不会被使用：

```java
// 最糟糕的 hashCode 方法的合法实现 - 永远不要使用!
@Override public int hashCode() { return 42; }
```

这样写合法是因为它确保了相等的对象拥有同样的 hash 码，可它同时更残暴地让每个对象都拥有同一个 hash 码。因此，每个对象都被映射到同一个 hash 桶中，而且 hash 表也退化成了链表。本应该线性时间运行的程序却以二次时间运行。对规模更大的 hash 表来说，这会导致能运行和不能运行的区别。

一个好的 hash 函数应该为不等的实例生成不同的 hash 码，这正是 hashCode 约定的第三条。理想情况下，一个 hasn 函数应该把不相等实例的合理集合均匀分布在所有的 int 值上。然而要达到理想状态是很困难的，幸运的是实现近似公平并不是很难。这里有一些简单的方法：

1. 声明一个名为 result 的 int 变量，把对象中第一个有意义的字段初始化为 hash 码 C，如步骤 2 中 a 所计算的。（回想一下[第十条][Item10]，关键域是指 equals 方法中涉及的每个域。）

2. 对于对象中每个残留关键域 f ，执行下列步骤：

   a. 为该域计算一个 int 型的 hash 码 C：

   	i. 如果该域是私有类型，则计算 `Type.hashCode(f)`，其中 Type 是对应于 f 类型的装箱类。
	
   	ii. 如果该域是对象引用，并且该类的 equals 方法递归地调用 equals 方法比较该域，递归调用域内的 hashCode 方法。如果需要更复杂的对比关系，则为这个域计算一个“范式（canonical representation）”，然后针对这个范式调用 hashCode。如果这个域的值是 null，则返回 0（或者其他常数，但通常是 0）。
	
   	iii. 如果该域是数组，则要把每一个元素当作单独的域来处理。也就是说，递归地应用上述规则，对每个关键元素计算一个 hash 码，然后根据步骤 2.b. 中的做法把这些 hash 值组合起来。如果数组中没有关键元素，则使用常量，但最好不要使用 0。如果所有元素都是关键元素，使用 `Arrays.hashCode`。

   b. 按照下面的公式，把步骤 2.a. 计算的 hash 码 C 合并到 result 中：`result = 31 * result + c`

3. 返回 result。

当写完了 hashCode 方法之后，问问自己是否相等的实例都含有相等的 hash 码，并编写测试单元来确认你的直觉是否正确（除非使用自动赋值（AutoValue）生成 equals 方法和 hashCode 方法，这种情况下可以安全的省略这些测试）。如果相等的实例含有不等的 hash 码，要找出原因并修复问题。

在 hash 码计算中，可以把冗余域（redundant field）排除在外。换句话说，如果一个域的值可以根据参与计算的其他域计算出来，则可以把这样的域排除在外。必须排除 equals 比较计算中没有用到的域，否则可能会违反 hashCode 约定中的第二条。

步骤 2.b 中的乘法会使 result 依赖域的顺序，如果类中有多个相似的域，则产生一个更好的 hash 函数。比如，如果从 String 类中的 hash 函数省略了乘法运算，所有的字都拥有相同的 hash 码。选择数值 31 是因为它是奇素数。如果它是偶数而且乘法溢出，便会丢失信息，因为乘法就是移位运算。使用素数的优点并不是很明了，但习惯上都是这么用的。31 有个很好的特性，它可以用一次移位运算和一次减法运算代替乘法运算可以得到更好的性能：`31 * i == ( i << 5 ) - i`。现代虚拟机自动完成这种优化。

现在我们把上述解决办法用到 PhoneNumber 类中：

```java
// 典型的 hashCode 方法
@Override 
public int hashCode() {
	int result = Short.hashCode(areaCode);
	result = 31 * result + Short.hashCode(prefix);
	result = 31 * result + Short.hashCode(lineNum);
    return result;
}
```

因为这个方法返回的是一个简单、确定的计算结果，它的输入只是 PhoneNumber 实例中的三个关键域，因此相等的 PhoneNumber 显然都会有相等的 hash 码。事实上，这个方法完美地实现了 PhoneNumber 类的 hashCode 方法，相当于 Java 平台类库中的实现。它的作法很简单，相当快捷，而且恰当地把不相等的电话号码分散到不同的 hash 桶中。

虽然本条目中的解决方法产生了相当不错的 hash 函数，但它们并不是最优秀的。这些方法在 hash 函数的质量上可以与 Java 平台类库的数据类型的 hash 函数相媲美，对于大多数用途来说都是足够的。如果确实需要 hash 函数来减少冲突，请查看 Guava 的 com.google.common.hash.Hashing [Guava][Guava]。

Objects 类有一个静态方法，这个静态方法可以存放任意数量的对象并为它们返回一个 hash 码。这个方法名叫 hash，只写一行 hashCode 方法，质量却可以和本条提供的解决方法相媲美。不幸的是，它们运行地更慢，因为它们需要创建数组来传递可变数量的参数，而且如果所有参数均属于原始类型，则可以进行装箱和反装箱。

这种风格的 hash 函数只有在性能要求不严格的情况下才推荐使用。下面是使用这种方法写的 PhoneNumber 类的 hash函数：

```java
// 一行式 hashCode 方法 - 性能一般
@Override 
public int hashCode() {
	return Objects.hash(lineNum, prefix, areaCode);
}
```

如果类是不可变的，并且计算 hash 码的开销也比较大，应该考虑把 hash 码缓存在对象内部，而不是每次请求的时候都重新计算 hash 码。如果你觉得这种类型的大多数对象会被当作 hash 键值（hash keys）使用，就应该在创建实例的时候计算 hash 码。否则，可以选择延迟初始化（lazily initialize）hash 码，在第一次调用 hashCode 时初始化。当使用延迟初始化方式时，要保持线程的安全需要注意一些条件（[第 83 条][Item83]）。PhoneNumber 类不适合这样的处理方式，在这里仅仅展示它是如何实现的。注意 hashCode 域的初始值（在本例中为 0 ）不应该是通过常规方式创建的实例的 hash 码。

```java
// 使用延迟初始化的 hashCode 方法缓存 hash 码
private int hashCode; // 自动初始为 0 
@Override 
public int hashCode() {
	int result = hashCode;
	if (result == 0) {
		result = Short.hashCode(areaCode);
		result = 31 * result + Short.hashCode(prefix);
		result = 31 * result + Short.hashCode(lineNum);
		hashCode = result;
		}
    return result;
}
```

**不要试图从 hash 码的计算中排除一些关键域来提高性能**。虽然 hash 函数的结果可以跑得很快，但它的质量太差，可能会降低 hash 表的性能导致其根本无法使用。特别是在实践中，hash 函数要面对大量的实例集合，在你忽略掉的区域中，这些实例仍然区别很大。如果这样的话，hash 函数将会把所有的实例映射到极少数的 hash 码上，本应该以线性时间运行的程序就会以平方级的时间运行。

这不仅仅是一个理论问题。在 Java 2 发行之前，String 类的 hash 函数最多只能检查 16 个字符，从第一个字符开始，在整个字符串中均匀选取。对于像 URL 这样的大型的分层名字的集合来说，这样的 hash 函数正好表现出了这里所提到的病态行为。

**不要给 hashCode 方法返回的值提供明确的规范，这样客户端就不能合理地使用它；也使 hash 函数的改变更复杂了**。在 Java 类库中，比如 String 和 Integer 类，将其 hashCode 方法返回的确切值指定为实例值的函数。这并不是好办法，而是一个我们不得不面对的错误：它限制了在未来版本中提高 hash 函数的能力。如果保留了未指定的细节，并且在 hash 函数中发现了缺陷，或者发现了更好的 hash 函数，可以在后续版本中改进它。

总之，每次重写 hashCode 方法时都必须重写 equals 方法，否则程序将不能正确运行。HashCode 方法必须遵守 Object 类中的规定的约定，必须为不同的实例合理分配不等的 hash 码。这很容易实现，如果稍微单调乏味，可以使用第 51 页的方法。就像[第 10 条][Item10]中提到的那样，自动赋值框架为手动编写 equals 和 hashCode 方法提供了一个很好的选择，IDE 同样提供了一些这样的功能。



[Item10]: url	"./第%2010%20条：覆盖%20equals%20时请遵守通用约定.md"
[Item83]: url	"在未来填入第 83 条的 url，否则无法进行跳转"
[Guava]: https://github.com/google/guava	"Guava. 2017. Google Inc."


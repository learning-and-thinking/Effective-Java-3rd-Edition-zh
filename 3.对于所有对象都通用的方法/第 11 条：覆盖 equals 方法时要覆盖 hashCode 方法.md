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


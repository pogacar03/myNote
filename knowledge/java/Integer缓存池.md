# Integer 缓存池

## 核心结论

Java 中 `Integer` 默认缓存 **-128 ~ 127** 范围内的对象。

自动装箱：

```java
Integer a = 127;
```

底层相当于：

```java
Integer a = Integer.valueOf(127);
```

`Integer.valueOf()` 会优先从 `IntegerCache` 中取对象，因此：

```java
Integer a = 127;
Integer b = 127;

System.out.println(a == b); // true
```

因为 `a` 和 `b` 指向同一个缓存对象。

---

## 超出缓存范围

```java
Integer a = 128;
Integer b = 128;

System.out.println(a == b); // false
```

`128` 默认不在缓存范围内，因此通常会得到两个不同的 `Integer` 对象。

---

## 显式 new 不走缓存

```java
Integer a = new Integer(127);
Integer b = new Integer(127);

System.out.println(a == b); // false
```

显式 `new Integer()` 会创建新对象，不使用 `IntegerCache`。

> `new Integer(int)` 在较新的 Java 版本中已经被标记为 deprecated，不建议使用。


## 面试回答

> Integer 自动装箱底层调用 `Integer.valueOf()`。Java 默认通过 `IntegerCache` 缓存 -128 到 127 范围内的 Integer 对象，所以这个范围内自动装箱得到的对象可能是同一个引用；超出范围通常会创建新的对象。包装类比较数值时应该使用 `equals()`，不要依赖 `==`。


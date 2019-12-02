---
layout:     post
title:      "Java基础-05丨hashCode()和equals()"
date:       2019-12-02 14:23:16
author:     "jiefang"
header-style: text
tags:
    - Java基础
---
# hashCode()和equals()

## hashCode()
### 简介
Java注释：
>Returns a hash code value for the object. This method is supported for the benefit of hash tables such as those provided by {@link java.util.HashMap}.

翻译：
>返回对象的哈希码值。此方法是为了支持哈希表，例如HashMap。

### 约定
hashCode()方法注释
```
     * <p>
     * The general contract of {@code hashCode} is:
     * <ul>
     * <li>Whenever it is invoked on the same object more than once during
     *     an execution of a Java application, the {@code hashCode} method
     *     must consistently return the same integer, provided no information
     *     used in {@code equals} comparisons on the object is modified.
     *     This integer need not remain consistent from one execution of an
     *     application to another execution of the same application.
     * <li>If two objects are equal according to the {@code equals(Object)}
     *     method, then calling the {@code hashCode} method on each of
     *     the two objects must produce the same integer result.
     * <li>It is <em>not</em> required that if two objects are unequal
     *     according to the {@link java.lang.Object#equals(java.lang.Object)}
     *     method, then calling the {@code hashCode} method on each of the
     *     two objects must produce distinct integer results.  However, the
     *     programmer should be aware that producing distinct integer results
     *     for unequal objects may improve the performance of hash tables.
     * </ul>
     * <p>
```
翻译：

{@code hashCode}的一般约定为：
- 在Java应用程序执行期间，只要在同一个对象上多次调用它，{@ code hashCode}方法必须始终返回相同的整数，前提是在对象上进行{@code equals}比较时使用任何信息没有被修改。从一个应用程序的一次执行到同一应用程序的另一次执行，此整数不必保持一致。
- 如果根据{@code equals（Object）}方法，两个对象相等，则分别在两个对象上调用{@code hashCode}方法，这两个对象必须产生相同的整数结果。
- 以下情况不 是必需的:如果根据{@link java.lang.Object＃equals（java.lang.Object）}方法，两个对象不相等，然后调用{@code hashCode} 两个对象中的每个对象上的方法必须产生不同的整数结果。但是，程序员应该意识到，为不相等的对象生成不同的整数结果可能会提高哈希表的性能。

## equals(Object obj)

### 简介
Java注释：
>Indicates whether some other object is "equal to" this one.

翻译：
>指示其他某个对象是否与此对象“相等”。

### 规范

```
     * <p>
     * The {@code equals} method implements an equivalence relation
     * on non-null object references:
     * <ul>
     * <li>It is <i>reflexive</i>: for any non-null reference value
     *     {@code x}, {@code x.equals(x)} should return
     *     {@code true}.
     * <li>It is <i>symmetric</i>: for any non-null reference values
     *     {@code x} and {@code y}, {@code x.equals(y)}
     *     should return {@code true} if and only if
     *     {@code y.equals(x)} returns {@code true}.
     * <li>It is <i>transitive</i>: for any non-null reference values
     *     {@code x}, {@code y}, and {@code z}, if
     *     {@code x.equals(y)} returns {@code true} and
     *     {@code y.equals(z)} returns {@code true}, then
     *     {@code x.equals(z)} should return {@code true}.
     * <li>It is <i>consistent</i>: for any non-null reference values
     *     {@code x} and {@code y}, multiple invocations of
     *     {@code x.equals(y)} consistently return {@code true}
     *     or consistently return {@code false}, provided no
     *     information used in {@code equals} comparisons on the
     *     objects is modified.
     * <li>For any non-null reference value {@code x},
     *     {@code x.equals(null)} should return {@code false}.
     * </ul>
     * <p>
```
翻译：

{@code equals}方法在非null对象引用上实现等价关系：
- 自反性：对于任何非空对象x,x.equals(x)应该返回true;
- 对称性：对于任何非空对象x和y,当且仅当y.equals(x)返回true，则x.equals(y)返回true;
- 传递性：对于任何非空对象x、y和z,x.equals(y)返回true,且y.equals(z)返回true，则x.equals(z)返回true;
- 一致性：对于任何非空对象x和y,如果没有修改{@code equals}方法使用的信息，则多次调用x.equals(y)方法始终返回ture或false;
- 对于任何非空对象x,x.equals（null）返回false;

### 高质量的equals()

- 使用==运算符检查参数是否为该对象的引用。如果是，返回true。这是一种性能优化，但是如果这种比较可能很昂贵的话，那就值得去做。
- 使用instanceof运算符来检查参数是否具有正确的类型。 如果不是，则返回false。 通常，正确的类型是equals方法所在的那个类。 有时候，改类实现了一些接口。 如果类实现了一个接口，该接口可以改进 equals约定以允许实现接口的类进行比较，那么使用接口。 集合接口（如Set，List，Map和Map.Entry）具有此特性。
- 参数转换为正确的类型。因为转换操作在instanceof中已经处理过，所以它肯定会成功。
- 对于类中的每个“重要”的属性，请检查该参数属性是否与该对象对应的属性相匹配。如果所有这些测试成功，返回true，否则返回false。如果步骤2中的类型是一个接口，那么必须通过接口方法访问参数的属性;如果类型是类，则可以直接访问属性，这取决于属性的访问权限。

## hashCode()和equals()的联系
hashCode()方法和equal()方法的作用其实一样，在Java里都是用来对比两个对象是否相等：
- equal()相等的两个对象他们的hashCode()肯定相等，也就是用equal()对比是绝对可靠的；
- hashCode()相等的两个对象他们的equal()不一定相等，也就是hashCode()不是绝对可靠的。

对于需要大量并且快速的对比的话如果都用equal()去做显然效率太低，所以解决方式是：
- 每当需要对比的时候，首先用hashCode()去对比，如果hashCode()不一样，则表示这两个对象肯定不相等（也就不必再用equal()去对比）；
- 如果hashCode()相同，此时再对比他们的equal()，如果equal()也相同，则表示这两个对象是真的相同了，这样既能大大提高了效率也保证了对比的绝对正确性；

## String中的hashCode()和equals()

String类里复写了hashCode()和equals()两个方法，所以Map中经常使用String作为key。

```
    public int hashCode() {
        int h = hash;
        if (h == 0 && value.length > 0) {
            char val[] = value;

            for (int i = 0; i < value.length; i++) {
                h = 31 * h + val[i];
            }
            hash = h;
        }
        return h;
    }
    //String.equals是循环比较两个字符串的char内容
    public boolean equals(Object anObject) {
        if (this == anObject) {
            return true;
        }
        if (anObject instanceof String) {
            String anotherString = (String)anObject;
            int n = value.length;
            if (n == anotherString.value.length) {
                char v1[] = value;
                char v2[] = anotherString.value;
                int i = 0;
                while (n-- != 0) {
                    if (v1[i] != v2[i])
                        return false;
                    i++;
                }
                return true;
            }
        }
        return false;
    }    
```

## 总结
- hashCode()用于提高散列表数据结构中中查找的效率，线性表中没有用；
- 若重写了equals(Object obj)方法，则必须重写hashCode()方法；
- 若两个对象equals(Object obj)返回true，则hashCode（）也必须返回相同的int数；
- 若两个对象equals(Object obj)返回false，则hashCode（）不一定返回不同的int数；
- 若两个对象hashCode（）返回相同int数，则equals（Object obj）不一定返回true；
- 若两个对象hashCode（）返回不同int数，则equals（Object obj）一定返回false；
- 同一对象在执行期间若已经存储在集合中，则不能修改影响hashCode值的相关信息，否则会导致内存泄露问题；
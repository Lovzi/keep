---
title: Java自动装箱和拆箱
date: 2021-01-25
categories: Java
tags:
	- Java
---

# Java自动装箱和拆箱

Java打着一切皆对象的旗号，给了8大基本类型都定义了包装类

因此在赋值过程，包装类型和基本类型会自动进行装箱和拆箱。 以Integer类型为例
装箱： 调用Integer.valueOf()
拆箱： 调用Integer.intValue()

此外不进赋值过程会进行自动拆装箱，以下过程也会设计自动拆装箱

- Integer == int. 将Integer拆箱成 int. 其他包装类同理。
- Integer +-*/ Integer等算数运算会自动拆箱成int. 其他包装类同理。

装箱和拆箱注意以下几点

1. Java基于资源和性能的考量，对基本类型的包装类都进行了一定程度的缓存（浮点类型除外）

> 拿Integer类型来说，[-128, 127]数值的整型， 在Integer.ValueOf()能看到其缓存了对应的引用对象，减少了new对象的损耗。

2. 基本类型比较很简单，但将其装箱之后的比较可能需要注意一下，明白了以下几点，判断比较就不会有大问题。

> a. 使用“==”， 当 “==”运算符的两个操作数都是 包装器类型的引用，则是比较指向的是否是同一个对象。 如果比较过程中，有一方包含了算数运算（进行拆箱），此时变成了包装类==基本类型，比较数值（再次触发另一方自动拆箱的过程）。
>
> b. 使用“equals”, equals方法并不会进行类型转换。
>
> c.  long类型和int类型比较会隐式转换成long==long, 其他基本类型同理，会将其往更大的精度进行转换，使其相同类型并保证精度。
>
> d. 对字节数组的转型稍微特殊, 使用Object接收字节数组之后， 使用instanceof判断对象时，需使用byte[]而不是Byte[]， 同样，转型操作也只能使用(byte[]) obj
>
> 

3. 在使用float转double之后可能会出现精度变大时，数值出现出现一串小尾巴。 `Doubole d = 3.6f;`，解决这个问题，可以通过字符串来搞定。 将浮点数转换为字符串，在通过`Double.parseDouble()`来解决这个问题。
4. 同时双精度转单精度， long转int等导致精度缺失的问题也得注意。 

测试样例1： 自动拆箱，缓存池。

```java
public class Test {
    public static void main(String[] args) {
        int a = 1;
        Integer b = 1;
        int c = 333;
        Integer d = 333;
        long e = 333L;
        Long f = new Long(333L);
        System.out.println(a == b); // true 自动拆箱
        System.out.println(c == d); // true 自动拆箱
        System.out.println(d == e); // true 自动拆箱
        System.out.println(d.equals(f)); // false 不会进行类型转换，不同类型不相同
        System.out.println(d.equals(e)); // false e自动装箱，但不会进行类型转换，不同类型不相同
    }
}
```

测试样例2： 缓存池， 算术运算拆装箱问题

```java
public class Test {
    public static void main(String[] args) {

        Integer a = 1;
        Integer b = 2;
        Integer c = 3;
        Integer d = 3;
        Integer e = 321;
        Integer f = 321;
        Long g = 3L;
        Long h = 2L;

        System.out.println(c==d); // true 比较引用，缓存池引用相同
        System.out.println(e==f); // false 比较引用，同类型不同对象引用，地址不同
        System.out.println(c==(a+b)); // true 算数运算拆箱右边， 包装类==基本类型，再次拆箱左边
        System.out.println(c.equals(a+b)); // true  1. 算数运算拆箱，2. Object形参再装箱Integer，比较引用内数值
        System.out.println(g==(a+b)); // true 1. 算数运算拆箱，包装类==基本类型，再次拆箱左边， 3. long == int 隐式转换int为long
        System.out.println(g.equals(a+b)); //  false 1. 算数运算拆箱，2. Object形参再装箱Integer, 不同包装类，不相同
        System.out.println(g.equals(a+h)); // true 1. 算数运算拆箱, 3. long == int 隐式转换int为long, 3. Object形参再装箱Long, 相同包装类，内容相同
    }
}
```

测试样例3： 字节数组转字符串，字节数组向上转型

```java
    public void test3(){ // 字节数组转字符串，以及字节数组向上转型
        Object s = "test".getBytes(); 字节数组向上转型
        System.out.println(s instanceof byte[]); 
        System.out.println(new String((byte[]) s)); 字节数组转字符串

    }
```

测试样例4:  单精度转双精度导致精度错误。 好像是计算器导致的。 

```java
 public void doubleTest(){
        float f = 3.6f;
        System.out.println(f); //3.6
        Float F = 3.6f;
        System.out.println(F); //3.6

        double d = f;
        System.out.println(d); // 3.5999999046325684
        System.out.println((double) 3.6f); // 3.5999999046325684
        System.out.println((double) f);  // 3.5999999046325684
        System.out.println((double) (Float) f); // 3.5999999046325684
        System.out.println((Double) (double) (Float) f); // 3.5999999046325684
    } 
    
    @Test
    public void fix(){
        String str = String.valueOf(3.6f);   // 单精度转字符串
        System.out.println(Double.parseDouble(str)); // 字符串转双精度
    }
```

测试样例5: 双精度转单精度精度缺失问题，这点需要注意

```java
  @Test
    public void floatTest(){
        System.out.println("===双精度转单精度");
        double d2 = 3.5999999f;
        double d3 = 3.5234234334f;
        System.out.println((float) d2); // 3.6
        System.out.println((float) d3); // 3.5234234
        int i = (int) 34435345345L;
        System.out.println(i);   // 75606977 , 一个完全不同的值， 直接裁剪后面掉后面四个字节。 
    }
```


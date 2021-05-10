# Scala和Java的交互

## Scala的隐式转换

### Java集合与Scala集合互相转换

#### Java集合转Scala

```scala
  @Test
  def asScala(): Unit ={
    val javaList: java.util.List[String] = new util.ArrayList()

    javaList.add("1")
    javaList.add("1")
    javaList.add("1")

//    javaList.map(row => row.toString)
    val arr: mutable.Buffer[Int] = javaList.asScala.map(row => row.toInt)
    val scalaList: Iterator[String] = javaList.iterator().asScala

    val map: mutable.Map[Int, Int] = util.Collections.singletonMap(1, 1).asScala
    val singleList: mutable.Buffer[Int] = util.Collections.singletonList(1).asScala
    // error
//    javaList.toArray().asScala


  }
```

#### Scala集合转Java

```scala
	@Test
  def asJava(): Unit = {
    val list = List(1,2,3,4,5)

    val newJavaList: util.List[Int] = list.asJava
    println(newJavaList)

    val map = Map(1->1, 2->2)
    println(map.asJava)

    var set = Set(1, 2)
    set = set + 3
    set = set + 4
    println(set.asJava)

    val arr = Array(1,2,3,4,5)
    println(arr.mkString("Array(", ", ", ")"))
    // error
//    val newJavaArray = arr.asJava
  }
```

### Scala的注解

Scala注解显示抛出异常

注意， 这里的@throws注解，在scala内部不生效。  scala内部已经舍弃了在方法头部显式表达可能发生的异常，这个注解专为Java程序员而生。

scala定义

```scala
import org.junit.Test

class ThrowTest {
  @Test
  def test(): Unit = {
    Throws.scala()
  }
}

object Throws{
  @throws(classOf[Exception])
  def java(): Unit ={
    throw new Exception("this is a Exception")
  }
  @throws(classOf[Exception])
  def scala(): Unit ={
    throw new Exception("this is a Scala Exception")
  }
}
```

Java调用

```java
public class JavaThrowTest {
    @Test
    public void test() throws Exception {
        Throws.java();
    }
}
```

#### 注解变长方法

在Scala经常使用的变长参数在Java中使用会遇到问题， 需要通过@varargs进行注解

scala定义

```scala
object Varargs{
  @varargs
  def print(args: Any*): Unit ={
    args.foreach(println)
  }
}
```

Java调用

```java
@Test
public void vary(){
    Varargs.print(1,2,3,4,5);
}
```

#### 其他注解

@BeanProperty 	标记生成JavaBean风格的getter和setter方法
@BooleanBeanProperty	标记生成is风格的getter方法，用于boolean类型的field
@NotNull	标记不为空
@volatile	标记为JVM中volatile的字段，可以被多个线程同时更新

@transient  标记为JVM中transient字段，该字段不会被序列化@throws	标记给方法要抛出checked异常
@tailrec	尾递归。递归调用有时候能被转化成循环，能节约栈空间
@varargs	标记方法接收的是变长参数
@SerialVersionUID	标记指定可序列化类的序列化版本
@native	标注用C,C++实现的方法
@strictfp	标记为精确浮点，确保浮点数运算的准确性
@switch	标记跳转表生成与内联
@inline	标记编译器内联
@noinline	标记编译器不要内联
@elidable	标记方法可以省略，编译时不会生成代码
@deprecated	标记方法弃用，让编译器生成警告提示
@deprecatedName	标记弃用的参数名，让编译器生成警告提示
@unchecked	标记匹配不完整时取消警告提示
@uncheckedVariance	标记取消与型变相关的错误提示
@implicitNotFound	标记如果隐式转换类型不存在，生成错误提示
@cloneable	标记可被克隆对象
@remote	标记可被远程连接的对象
@specialized	标记让编译器自动生成方法



Trait在java的继承问题

Scala定义

```scala
import org.junit.Test

class TraitsTest {
  @Test
  def test(): Unit ={
    new iInt().sum(1,2)
  }
}


class iInt extends SumAble{
  override def sum(x: Int, y: Int): Int = super.sum(x, y)
}


trait SumAble{
  def sum(x: Int, y: Int): Int = {
    x + y
  }
}
```

Java调用

```java

class JInt extends iInt{
    @Override
    public int sum(int x, int y) {
        return super.sum(x, y);
    }
}


class JIntError implements SumAble{
    @Override
    public int sum(int x, int y) {
        return x + y;
    }
}
```


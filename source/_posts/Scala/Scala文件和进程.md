---
title: Scala文件和进程
categories: Scala
tags:
  - Scala
abbrlink: 7e3741a7
date: 2021-07-09 10:55:17
---


#

### 打开一个文件按行读取内容

```scala
	// 错误示范，简单的打开一个文件
  @Test
  def read(): Unit ={
    for(line <- Source.fromFile("/Users/bamboo/.zshrc").getLines()){
      println(line)
    }
  }

  // 正确示范1，简单的打开一个文件
  @Test
  def read2(): Unit ={
    val source = Source.fromFile("/Users/bamboo/.zshrc")
    // source.getLines().mkString()
    for(line <- source.getLines()){
      println(line)
    }
    source.close()
  }

  // 正确示范1，简单的打开一个文件
  @Test
  def read3(): Unit ={
    val source = Source.fromFile("/Users/bamboo/.zshrc")
    // source.getLines().mkString()
    for(line <- source.getLines()){
      println(line)
    }
    source.close()
  }
```

Source.getLines将文件的输入流转成一个迭代器，因此正常迭代器该有的操作，这里都有。

### 读取一个文件写入另一个文件

Scala的作者，其实也是Java的作者，非常不喜欢重复造轮子，懒得实现Scala的读写操作了。 所以，你会发现Scala的读写能力少的可怜。 基本都是用Java的老本行

示范一下， 打开一个文件并按字符写入数据

```java
// 逐个读取字符，知道读到-1
  @Test
  def write(): Unit ={
    var in: Option[FileInputStream] = None
    var out: Option[FileOutputStream] = None
    try{
      in = Some(new FileInputStream(fileName))
      out = Some(new FileOutputStream(copyFileName))
      var c: Int = 0
      while ({c = in.get.read; c != -1}){
        out.get.write(c)
      }
    }catch {
      case e: IOException => e.printStackTrace()
    }finally {
      if(in.isDefined){
        in.get.close()
      }
      if(out.isDefined){
        out.get.close()
      }
    }
  }
```

这里尤其注意到是类似Java：(c = in.read()) 这样的语法在Scala中的返回值是Unit

### 列出目录

```scala
@Test
def listFile(): Unit ={
  list(new File("/tmp"), 1)
}

def list(f: File, n: Int): Long = {
  if(!f.canRead ){
    return n
  }
  if(n < 0){
    return 0
  }
  println((if(f.isDirectory) " d " else " f ")  + f.getPath)
  if(f.canRead && f.isDirectory){
    for(nf <- f.listFiles()){
      list(nf, n-1)
    }
  }
  n
}
```

### 如何处理CSV文件

其实scala对文件语法还是很少，一般都是调用的Java的api

```scala
@Test
def csv(): Unit ={
  val source = Source.fromFile("/Users/bamboo/test.csv")
  for(line <- source.getLines()){
    val Array(a,b,c,d,e) = line.split(',').map(_.trim)
    println(s"$a, $b, $c, $d, $e")
  }
  source.close()
}
```

### 将字符串伪装成文件

```scala
// 将字符串伪装成文件
@Test
def mkFile(): Unit ={
  printLine(Source.fromString("hello world"))
}
def printLine(source: Source): Unit = {
  for(line <- source.getLines()){
    print(line)
  }
}
```

### 序列化对象写入文件

一般来讲，Scala在很多方面和Java有共同之处，Scala的序列化方式和Java的别无二致，尽在语法有一定区别

为了使Scala的类能够序列化，肯定是要继承Serializable接口， 随后添加对应的SerializableUID标注

```scala
class SerializerTest {
  @Test
  def test(): Unit ={
    val p = new Person(1, "name", 22, 172.1)
    val p2 = new Student(1, "1班")

    val oos = new ObjectOutputStream(new FileOutputStream("/tmp/person"))
    oos.writeObject(p)
    oos.writeObject(p2)
    oos.close()
    val ois = new ObjectInputStream(new FileInputStream("/tmp/person"))
    val p3 = ois.readObject().asInstanceOf[Person]
    val p4 = ois.readObject().asInstanceOf[Person]
    oos.close()
    println(p)
    println(p2)
    println(p3)
    println(p4)
  }
}


@SerialVersionUID(100L)
class Person(id: Long, name: String, age: Int, height: Double) extends Serializable {
  override def toString: String = id.toString + name.toString + age.toString + height.toString
}


@SerialVersionUID(100L)
class Student(id: Long, c: String) extends Person(id, "", 3, 4.4) {
  override def toString: String = c + super.toString
}
```

### 执行外部命令

scala通过隐式转换主要提供了三种方法能够方便的执行外部shell命令， 比如以下语法糖

```
 // 字符串方式
  @Test
  def ls(): Unit ={
    import sys.process._

		// 使用!方法执行命令，并得到它的退出状态。 
    val exitCode = "ls -al".!
		// 使用！！方法执行命令，并得到它的退出状态。并获得它的输出
    val exitCode2 = "ls -al".!!
		// 后台执行，以流的方式返回结果
    val testString = "echo test".lineStream
    println(testString)
  }
```






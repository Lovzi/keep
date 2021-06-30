# Document序列化Json中 object类型转换问题



## _id序列化问题

`_id`在展示的时候看着像String， 但是其本质是是一个Object对象，当我们正常使用document.toJson()去将其序列化成json字符串的时候， 我们会发现`_id`的结构如下所示

同理， score也被序列化为object

```scala
    val doc: Document = new Document("_id", new ObjectId("60501fe019de5fac92be0fab"))
    doc.put("score", 1111L)
   	println(doc.toJson())  
   	// result: { "_id": { "$oid" : "60501fe019de5fac92be0fab" }, "score" : { "$numberLong" : "1111" } }
    
```

要解决这个问题， 我们只需要自定义JsonWriterSettings,  完整代码如下。 

```scala
import org.bson.Document
import org.bson.json.{Converter, JsonWriterSettings, StrictJsonWriter}
import org.bson.types.ObjectId
import org.junit.Test 

class Mongo{
  @Test
  def test1(): Unit = {
    val settings = JsonWriterSettings.builder
      .int64Converter(new Converter[lang.Long] {
        override def convert(t: lang.Long, strictJsonWriter: StrictJsonWriter): Unit = {
          strictJsonWriter.writeNumber(t.toString)
        }
      })
      .objectIdConverter(new Converter[ObjectId] {
        override def convert(t: ObjectId, strictJsonWriter: StrictJsonWriter): Unit = {
          strictJsonWriter.writeString(t.toString)
        }
      })
      .build

    val doc: Document = new Document("_id", new ObjectId("60501fe019de5fac92be0fab"))
    doc.put("score", 1111L)

    println(doc.toJson())  // 加上setting之前
    println(doc.toJson(settings)) // 加上settings之后
    
    // result
    // { "_id" : { "$oid" : "60501fe019de5fac92be0fab" }, "score" : { "$numberLong" : "1111" } }
		// { "_id" : "60501fe019de5fac92be0fab", "score" : 1111 }
  }
}
```

参考文档：https://stackoverflow.com/questions/35209839/converting-document-objects-in-mongodb-3-to-pojos


---
title: FastJson是真的坑
categories: Java
tags:
  - Java
  - Json
abbrlink: 6857d1ab
date: 2021-03-25 00:00:00
---



最近在熟悉Fastjson的过程，意外发现一个异常情况，每次反序列化json字符串，修改其中内容，在序列化的时候，发现其中某些值莫名的丢失了。 复现之后发现JsonObject对象的null值在序列化之后会将此key丢弃。 

```java
 @Test
    public void test(){
        String t = "{\"remove_date\":null,\"ts\":1616674211648,\"type\":\"UPDATE\"}";
        JSONObject before = JSON.parseObject(t);
        System.out.println("before keySet: " + before.keySet());
        System.out.println("before remove_date" + before.get("remove_date"));
        System.out.println("before containsKey " +  before.containsKey("remove_date"));
        String afterStr = before.toJSONString();
        JSONObject after = JSON.parseObject(afterStr);
        System.out.println("after: " + after);
        System.out.println("after keySet : " + after.keySet());
        System.out.println("after remove_date: " + after.get("remove_date"));
        System.out.println("after containsKey: " + after.containsKey("remove_date"));
      
      
//        运行结果
//        before keySet: [remove_date, type, ts]
//        before remove_datenull
//        before containsKey true
//        after: {"type":"UPDATE","ts":1616674211648}
//        after keySet : [type, ts]
//        after remove_date: null
//        after containsKey: false
    }
```


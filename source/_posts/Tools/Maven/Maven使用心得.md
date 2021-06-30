



## Exclusions

在Maven导入某个依赖时，此包依赖的间接依赖可能会和当前模块的其他依赖有版本冲突，此时，可以使用`Exclusions`来排除某个包的间接依赖，并使用自己导入的依赖。 

Dependency 中排除某个包的依赖。

```xml
  <dependencies>
    <dependency>
      <groupId>org.apache.maven</groupId>
      <artifactId>maven-embedder</artifactId>
      <version>2.0</version>
      <exclusions>
        <exclusion>
          <groupId>org.apache.maven</groupId>
          <artifactId>maven-core</artifactId>
        </exclusion>
      </exclusions>
    </dependency>
    ...
  </dependencies>
```

如果某个包引入的间接依赖很多， 而你本身只需要使用其中的很少一部分， 其他依赖对你并不重要。 那你可以使用通配符匹配来排除所有的包。 

```xml
  <dependencies>
    <dependency>
      <groupId>org.apache.maven</groupId>
      <artifactId>maven-embedder</artifactId>
      <version>3.1.0</version>
      <exclusions>
        <exclusion>
          <groupId>*</groupId>
          <artifactId>*</artifactId>
        </exclusion>
      </exclusions>
    </dependency>
  </dependencies>
```

甚至可以单用匹配符去匹配groupId、artifactId等，来排除间接依赖，如下

```xml
<dependency>
    <groupId>com.tyc</groupId>
    <artifactId>tethys-ylinkc</artifactId>
    <version>1.3.8-SNAPSHOT</version>
    <exclusions>
        <exclusion>
            <groupId>com.google.protobuf</groupId>
            <artifactId>protobuf-java</artifactId>
        </exclusion>
        <exclusion>
            <groupId>org.apache.flink</groupId>
            <artifactId>*</artifactId>
        </exclusion>
        <exclusion>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
        </exclusion>
        <exclusion>
            <groupId>*</groupId>
            <artifactId>tablestore</artifactId>
        </exclusion>
    </exclusions>
</dependency>
```

## 官方网址https://maven.apache.org/pom.html#exclusions

​        <Logger level="DEBUG" name="com.tyc.warehouse.judicial.process"/>
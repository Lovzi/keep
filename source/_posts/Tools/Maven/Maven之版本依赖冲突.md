

### 冲突背景

在使用Maven管理Java项目时，由于我们需要直接或者间接依赖很多第三方的代码 ， 同时依赖第三方又回继续依赖其他组件的代码，就很容易出现版本依赖冲突的情况。

 当组件之间依赖同一个组件时，比如A依赖B、C, 但B、C依赖D，而这些代码开发者肯定不会默契的使用同一个版本。 比如分别依赖D的1.0、2.0版本。 

如下图。 当D开发者不遵守向下兼容的规范，导致1.0和2.0之间不兼容时（Google经常这样干）， 就会出现依赖冲突。 

![Jar包依赖](https://gitee.com/Goook/pictures/raw/master/uPic/Jar%E5%8C%85%E4%BE%9D%E8%B5%96.png)

问题出现了，我们如何去解决呢？

### 依赖解决

#### 伪造冲突

其实解决起来很简单，首先我们新建一个maven工程， 地址: `git@git.ty.ink:jindi_data/shade-maven.git`

然后，我们从B、C中任选一个，假设为B，创建一个同名模块，artifactId命名为B1，groupId命名为`com.tyc.shade`,

同时，我们在pom中导入B的依赖，此时我们已经伪造了一个B组件，我们完全可以使用B1打包后的Jar来替换B，在使用上没有任何影响，此时B1依旧依赖D(1.0)。 

比如下面的例子

```xml
 <artifactId>canal.client</artifactId>
<version>1.1.4</version>
<properties>
  <maven.compiler.source>8</maven.compiler.source>
  <maven.compiler.target>8</maven.compiler.target>
</properties>
<dependencies>
  <dependency>
    <groupId>com.alibaba.otter</groupId>
    <artifactId>canal.client</artifactId>
    <version>${version}</version>
    <exclusions>
      <exclusion>
        <groupId>ch.qos.logback</groupId>
        <artifactId>*</artifactId>
      </exclusion>
      <exclusion>
        <groupId>org.slf4j</groupId>
        <artifactId>*</artifactId>
      </exclusion>
    </exclusions>
  </dependency>
  <dependency>
    <groupId>com.alibaba.otter</groupId>
    <artifactId>canal.protocol</artifactId>
    <version>${version}</version>
    <exclusions>
      <exclusion>
        <groupId>ch.qos.logback</groupId>
        <artifactId>*</artifactId>
      </exclusion>
      <exclusion>
        <groupId>org.slf4j</groupId>
        <artifactId>*</artifactId>
      </exclusion>
    </exclusions>
  </dependency>
</dependencies>
```

然后伪造已完成，但我们直接使用的话，B1依赖的D(1.0)依旧和C依赖的D(2.0)有冲突，还是会出现下面的依赖问题, 我们的问题依旧没有解决。 



![Jar包依赖1.2](https://gitee.com/Goook/pictures/raw/master/uPic/Jar%E5%8C%85%E4%BE%9D%E8%B5%961.2.png)

#### 冲突依赖解决

不同的是，我们不能修改B的相关依赖等maven配置，但我们可以修改B1的相关依赖，因为现在B1是我们创建的，我们可以修改任何配置。 

我们只需要简单在build中增加一下配置。 

```xml
<build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.0</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <encoding>UTF-8</encoding>
                    <!--    <verbal>true</verbal>-->
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.2.1</version>
                <configuration>
                    <createDependencyReducedPom>false</createDependencyReducedPom>
                </configuration>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <filters>
                                <filter>
                                    <artifact>*:*</artifact>
                                    <excludes>
                                        <exclude>META-INF/*.SF</exclude>
                                        <exclude>META-INF/*.DSA</exclude>
                                        <exclude>META-INF/*.RSA</exclude>
                                    </excludes>
                                </filter>
                            </filters>
                            <transformers>
                                <!--                                <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">-->
                                <!--                                    <mainClass></mainClass>-->
                                <!--                                </transformer>-->
                                <transformer
                                        implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                                    <resource>reference.conf</resource>
                                </transformer>
                            </transformers>
                            <relocations>
                                <relocation>
                                    <pattern>com.google</pattern>
                                    <shadedPattern>shade.canal.client.google</shadedPattern>
                                </relocation>
                                <relocation>
                                    <pattern>org.apache.commons.cli</pattern>
                                    <shadedPattern>shade.canal.client.apache.commons.cli</shadedPattern>
                                </relocation>
                            </relocations>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
```

上面使用了`maven-shade-plugin`， 这个插件好处是会把多个包打包成一个，坏处是打包之后，我们上传到私服后再下载时，一般我们看不到这个包的间接依赖了，不解压这个包，我们是不知道这个包都依赖了什么的。

当然，最重要的是`maven-shade-plugin`插件中的下面配置

```xml
<relocations>
                                <relocation>
                                    <pattern>com.google</pattern>
                                    <shadedPattern>shade.canal.client.google</shadedPattern>
                                </relocation>
                                <relocation>
                                    <pattern>org.apache.commons.cli</pattern>
                                    <shadedPattern>shade.canal.client.apache.commons.cli</shadedPattern>
                                </relocation>
                            </relocations>
```

这个配置的含义是，在打包过程中，将`com.google`包命名为`shade.canal.client.google`，将`org.apache.commons.cli`命名为`shade.canal.client.apache.commons.cli`。 

这个`com.google`，其实就是上图中D组件内部的包名，实际为guava和protobuf两个组件，而`org.apache.commons.cli`,则是属于common-cli这个组件。他们几个组件，在不同版本，都会出现兼容性问题，因此我们都修改掉。 

因此，B1内部引用`com.google`下面的类时，会自动重定向到`shade.canal.client.google`下面查找，而C内部会继续去`com.google`下面查找。 这样依赖冲突就解决了。 

我们通过`mvn deploy::deploy-file`将B1上传到私服。 在shade-maven这个工程中，我编写了一个命令行工具来帮助大家上传， 不过大家需要先配置上传私服的settings.xml配置，配置完成后，使用`sh deployer.sh module=canal.client  version=1.1.4 prod `命令来上传

module代表对应需要安装工程的哪个模块，version代指要上传的版本， 上传正式版本需要加上prod，测试版本（版本号带上SNAPSHOT)可以忽略

最后以canal.client的配置模版结束这个文档。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <parent>
        <artifactId>shade-maven</artifactId>
        <groupId>com.tyc.shade</groupId>
        <version>1.0</version>
    </parent>
    <modelVersion>4.0.0</modelVersion>

    <artifactId>canal.client</artifactId>
    <version>1.1.4</version>
    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
    </properties>
    <dependencies>
        <dependency>
            <groupId>com.alibaba.otter</groupId>
            <artifactId>canal.client</artifactId>
            <version>${version}</version>
            <exclusions>
                <exclusion>
                    <groupId>ch.qos.logback</groupId>
                    <artifactId>*</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.slf4j</groupId>
                    <artifactId>*</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>com.alibaba.otter</groupId>
            <artifactId>canal.protocol</artifactId>
            <version>${version}</version>
            <exclusions>
                <exclusion>
                    <groupId>ch.qos.logback</groupId>
                    <artifactId>*</artifactId>
                </exclusion>
                <exclusion>
                    <groupId>org.slf4j</groupId>
                    <artifactId>*</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.0</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <encoding>UTF-8</encoding>
                    <!--    <verbal>true</verbal>-->
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.2.1</version>
                <configuration>
                    <createDependencyReducedPom>false</createDependencyReducedPom>
                </configuration>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <filters>
                                <filter>
                                    <artifact>*:*</artifact>
                                    <excludes>
                                        <exclude>META-INF/*.SF</exclude>
                                        <exclude>META-INF/*.DSA</exclude>
                                        <exclude>META-INF/*.RSA</exclude>
                                    </excludes>
                                </filter>
                            </filters>
                            <transformers>
                                <!--                                <transformer implementation="org.apache.maven.plugins.shade.resource.ManifestResourceTransformer">-->
                                <!--                                    <mainClass></mainClass>-->
                                <!--                                </transformer>-->
                                <transformer
                                        implementation="org.apache.maven.plugins.shade.resource.AppendingTransformer">
                                    <resource>reference.conf</resource>
                                </transformer>
                            </transformers>
                            <relocations>
                                <relocation>
                                    <pattern>com.google</pattern>
                                    <shadedPattern>shade.canal.client.google</shadedPattern>
                                </relocation>
                                <relocation>
                                    <pattern>org.apache.commons.cli</pattern>
                                    <shadedPattern>shade.canal.client.apache.commons.cli</shadedPattern>
                                </relocation>
                            </relocations>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```




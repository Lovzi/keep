---
title: Flink小坑之kerberos动态认证
categories: Flink
tags:
  - Flink
abbrlink: 3c8fe302
date: 2021-10-12 14:09:39
---

## Kerberos认证方式

### 方式一(仅限于YARN）：

在 YARN 模式下，可以部署一个没有 keytab 的安全 Flink 集群，只使用票据缓存（由`kinit`管理）。这避免了生成密钥表的复杂性，并避免将其委托给集群管理器。在这种情况下，Flink CLI 获取 Hadoop 委托令牌（用于 HDFS 和 HBase等）。主要缺点是集群必然是短暂的，因为生成的委托令牌将过期（通常在一周内）。

使用以下步骤运行安全的 Flink 集群`kinit`：

1. 使用`kinit`命令登录。
2. 正常部署 Flink 集群。

**注：tgt有一个有效期，过期了就无法使用了，这种方式不适合长期任务。**

### 方式二：

在原生 Kubernetes、YARN 和 Mesos 模式下运行安全 Flink 集群的步骤

1.在客户端的 Flink 配置文件中添加安全相关的配置选项（见[这里](https://nightlies.apache.org/flink/flink-docs-release-1.13/zh/docs/deployment/config/#auth-with-external-systems)）。

```yaml
security.kerberos.login.keytab: /kerberos/flink.keytab
security.kerberos.login.principal: flink
security.kerberos.login.contexts: Client
security.kerberos.login.use-ticket-cache: true
```

2.确保密钥表文件存在于`security.kerberos.login.keytab`客户端节点上的指示的路径中

3.正常部署 Flink 集群。 

在 YARN、Mesos 和原生 Kubernetes 模式下，keytab 会自动从客户端复制到 Flink 容器。

**注： 这里就遇到了我们说的小坑，kerberos认证先于命令行解析，命令行定义的配置在认证阶段是无效的**

因此，我们目前无法通过在命令行指定不同的keytab和principal覆盖文件中的配置，从而达到多用户认证，也就是目前我们只能通过flink用户来向YARN提交作业，没有了用户区分，我们就无法进行权限控制。 

那如何解决这个问题，从而达到每个用户都能以自己名义去提交作业呢，这里我们只要解决命令行解析先于kerberos认证就可以了（修改源码）

Flink自带的命令行解析器，如果我们借助自身命令行解析器（减少改动也就减少了bug）。 我们可能同时需要兼容所有的解析器，并且在新添加解析器时也增加了限制，可能在某个解析器没有对应参数，解析就不生效。 

因此，我们采用最简单最暴力的办法，自己定义一个解析器，解析命令行的参数来覆盖配置文件里定义参数。 

具体步骤如下：

1.自定义一个命令行解析器。 

```java
// 位于org.apache.flink.client.cli.CliFrontendParser下
public static class ExtendedGnuParser extends GnuParser {
        private final boolean ignoreUnrecognizedOption;

        public ExtendedGnuParser(boolean ignoreUnrecognizedOption) {
          	// GnuParser、DefaultParser在遇到未定义的参数时都会抛出异常，这里是为了进行兼容
            this.ignoreUnrecognizedOption = ignoreUnrecognizedOption;
        }

        protected void processOption(String arg, ListIterator<String> iter) throws ParseException {
            boolean hasOption = this.getOptions().hasOption(arg);
            if (hasOption || !this.ignoreUnrecognizedOption) {
                super.processOption(arg, iter);
            }

        }
    }
```

定义命令行参数并解析, 同时覆盖配置文件参数。 

```java
   
// 位于org.apache.flink.client.cli.CliFrontend
// 其this.configuration为从配置文件中读取的配置。 
    public Configuration compatCommandLineSecurityConfiguration(String[] args)
            throws CliArgsException, FlinkException, ParseException {
        // 定义一个自定义都参数解析Options。
        Options securityOptions = new Options()
                .addOption(Option.builder("D")
                .argName("property=value")
                .numberOfArgs(2)
                .valueSeparator('=')
                .desc("Allows specifying multiple generic configuration options.")
                .build());

        // 解析命令行传入的参数
        CommandLine commandLine =
                new CliFrontendParser.ExtendedGnuParser(true)
                .parse(securityOptions, args);

        // 从配置文件中读取的参数。
        Configuration securityConfiguration = new Configuration(this.configuration);

        // 用命令行传入参数替换掉配置文件中的security参数
        Properties dynamicProperties = commandLine.getOptionProperties("D");
        for(Map.Entry<Object, Object> entry: dynamicProperties.entrySet()){
            securityConfiguration.setString(
                entry.getKey().toString(),
                entry.getValue().toString()
            );
        }
        return securityConfiguration;
    }
```

修改之后我们进行编译``` mvn clean package -DskipTests -Dfast```

编译完成后，flink根目录下会出现一个build-target的软连接，指向编译后的flink真正的安装包。 

![image-20211012162901732](https://gitee.com/Goook/pictures/raw/master/uPic/image-20211012162901732.png)

我们修改一下flink安装包的名称，由flink-1.13.2 修改为flink-1.13.2.1 来区分，同时上传到线上就可以了。 




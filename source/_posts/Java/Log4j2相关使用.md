## 为什么要使用Log4j2

1.  设计用于审计日志框架， 解决了logback 和 log4j1.x 在某种情况会丢失事件， log4j2 已经解决了这个问题。 同时， 在logback中异常对于日志Appender是不可见的， 但是在log4j2 是可以配置去允许异常渗透到这个程序。 
2.  Log4j2 基于 LMAX Disruptor Library 生成了下一代异步日志记录器。 在多线程情况下，吞吐量相对于1.x 和 logback 超过了10倍， 延迟也低了几个数量级。 
3. Log4j2 相对于独立程序做到 零GC， 对于WEB应用程序能够显著的降低GC， 减少了GC的压力并且提高程序的响应时间。 
4.  Log4j2 使用插件系统，来让添加一个Appender、Filters、Layouts、Pattern等变得非常容易，它对Log4j2不需要做任何修改。 
5.  由于插件系统配置十分容易， 配置中的条目不需要指定类名
6. 支持自定义日志级别， 可以在配置里或者在代码里自定义日志类别。 
7. 支持 lambda 拓展， 运行在Java8的代码可以使用 lambda表达式进行懒加载，从而达到仅在日志级别达到时，创造一条日志消息，不需要显式的级别检查，从而生成更干净的代码。 
8.  支持日志对象。通过日志系统和高效的操作，用户能够自由的去创建他们的消息类型并且通过Layouts、Filters、Lookups去操作他们。   从而能够创造非常有趣并且非常复杂的消息
9.  Log4j1 支持在Appenders上进行过滤，Logback 添加了TurboFilters 在处理一条日志之前能够过滤事件。 Log4j2 支持了Filters能够在处理一个事件之前就过滤这条数据，就好像在一个Logger或者Apender中过滤数据。 
10. 很多Logback的Appender不能接受一个Layout 并且仅能发送固定格式的数据。 大多数Log4j2的Appenders 可以接受一个Layouts，允许数据在进行传输之前设计任何格式。 
11.  在Log4j 1.x 和 Logback的布局会返回一个字符串， 在Logback的编码器有讨论过这个问题的结果， Log4j2 通过一个简单的方法，即布局总是返回一个字节数组。 这样做的好处意味着他几乎可以用于任何Appender, 而不仅仅是写入OutputStream的Appender. 
12. 日志Appender同时支持TCP和UDP以及BSD Syslog和RFC 5424格式
13.  Log4j2通过Java5在并发上的支持，在尽可能低的级别上执行锁定。 Log4j1 有已知的死锁问题，很多在Logback已经进行了修复，但是依然需要非常高的级别上进行同步。 
14.  它是apache 软件基金会的社区项目，想要贡献或者进行一个提交只需要遵守它的贡献规则。 






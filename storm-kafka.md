####Maven配置：
Storm-kafka的Kafka在maven中的依赖被定义为 ***provided*** ，就意味着在运行阶段就不需要这个jar包了。

当构建一个含有storm-kafka的项目时，你必须明确地添加Kafka依赖。假如你使用的是用Scala 2.10编辑的Kafka 0.8.1.1，你应该在你的pom.xml文件中加入如下依赖：

```java
<dependency>
	<groupId>org.apache.kafka</groupId>
	<artifactId>kafka_2.10</artifactId>
	<version>0.8.1.1</version>
	<exclusions>
		<exclusion>
			<groupId>org.apache.zookeeper</groupId>
			<artifactId>zookeeper</artifactId>
		</exclusion>
		<exclusion>
			<groupId>log4j</groupId>
			<artifactId>log4j</artifactId>
		</exclusion>
	</exclusions>
</dependency>
```

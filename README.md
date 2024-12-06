
使用 kafka\-clients 原本是比较简单的事情。但有些同学习惯了 spring\-kafka 后，对原始 java 接口会陌生些。会希望有个集成的示例。



```
<dependency>
    <groupId>org.apache.kafkagroupId>
    <artifactId>kafka-clientsartifactId>
    <version>${kafka.version}version>
dependency>

```

现在我们使用原始 sdk 的依赖包，做一个 solon 项目的集成分享（其它的框架，也可以参考此例）。


### 1、添加集成配置


使用 [Solon 初始器](https://github.com) 生成一个 Solon Web 模板项目，然后添加上面的 kafka\-clients 依赖。之后：


* 添加 yml 配置（具体的配置属性，参考：ProducerConfig，ConsumerConfig）



```
solon.app:
  name: "demo-app"
  group: "demo"

solon.logging:
  logger:
    root:
      level: INFO

# 配置可以自由定义，与 @Bean 代码对应起来即可（以下为参考）
solon.kafka:
  properties:  #公共配置（配置项，参考：ProducerConfig，ConsumerConfig 的公用部分）
    bootstrap:
      servers: "127.0.0.1:9092"
    key:
      serializer: "org.apache.kafka.common.serialization.StringSerializer"
      deserializer: "org.apache.kafka.common.serialization.StringDeserializer"
    value:
      serializer: "org.apache.kafka.common.serialization.StringSerializer"
      deserializer: "org.apache.kafka.common.serialization.StringDeserializer"
  producer: #生产者专属配置（配置项，参考：ProducerConfig）
    acks: "all"
  consumer: #消费者专属配置（配置项，参考：ConsumerConfig）
    enable:
      auto:
        commit: "false"
    isolation:
      level: "read_committed"
    group:
      id: "${solon.app.group}:${solon.app.name}"

```

* 添加 java 配置器



```
@Configuration
public class KafkaConfig {
    @Bean
    public KafkaProducer producer(@Inject("${solon.kafka.properties}") Properties common,
                                             @Inject("${solon.kafka.producer}") Properties producer) {

        Properties props = new Properties();
        props.putAll(common);
        props.putAll(producer);

        return new KafkaProducer<>(props);
    }

    @Bean
    public KafkaConsumer consumer(@Inject("${solon.kafka.properties}") Properties common,
                                                  @Inject("${solon.kafka.consumer}") Properties consumer) {
        Properties props = new Properties();
        props.putAll(common);
        props.putAll(consumer);

        return new KafkaConsumer<>(props);
    }
}

```

完成上面两步，就算是集成了（后面，是应用的事儿）。配置也可以没有，直接写代码设定属性。


### 2、应用


* 发送（或生产），这里代控制器由用户请求再发送消息（仅供参考）：



```
@Controller
public class DemoController {
    @Inject
    private KafkaProducer producer;

    @Mapping("/send")
    public void send(String msg) {
        //发送
        producer.send(new ProducerRecord<>("topic.test", msg));
    }
}

```

* 拉取（或消费），这里采用定时拦取方式：（仅供参考）


这里需要引入一个 solon 的简单调度插件（或，别的调度插件），用于定时拉取消息：



```
<dependency>
    <groupId>org.noeargroupId>
    <artifactId>solon-scheduling-simpleartifactId>
dependency>

```

编写定时拉取任务：



```
@Component
public class DemoJob {
    @Inject
    private KafkaConsumer consumer;

    @Init
    public void init() {
        //订阅
        consumer.subscribe(Arrays.asList("topic.test"));
    }

    @Scheduled(fixedDelay = 10_000L, initialDelay = 10_000L)
    public void job() throws Exception {
        //拉取
        ConsumerRecords records = consumer.poll(Duration.ofSeconds(10));
        for (ConsumerRecord record : records) {
            System.out.println(record.value());
            //确认
            consumer.commitSync();
        }
    }
}

```

 本博客参考[wgetcloud全球加速服务机场](https://wa7.org)。转载请注明出处！

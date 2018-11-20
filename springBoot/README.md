# kafka与spring boot集成

### 依赖
[最新spring kafka maven仓库地址](https://mvnrepository.com/artifact/org.springframework.integration/spring-integration-kafka)

    <dependency>
        <groupId>org.springframework.integration</groupId>
        <artifactId>spring-integration-kafka</artifactId>
        <version>3.1.0.RELEASE</version>
    </dependency>

### 配置文件application.properties
    kafka.servers=172.30.3.70:9092
    kafka.consumer.enable.auto.commit=true
    kafka.consumer.session.timeout=6000
    kafka.consumer.auto.commit.interval=100
    kafka.consumer.auto.offset.reset=latest
    kafka.consumer.topic=test
    kafka.consumer.group.id=test
    kafka.consumer.concurrency=10
    kafka.producer.retries=0
    kafka.producer.batch.size=4096
    kafka.producer.linger=1
    kafka.producer.buffer.memory=40960
### producer configuration
 ```java
    @Configuration
    @EnableKafka
    public class KafkaProducerConfig {
        @Value("${kafka.servers}")
        private String servers;
        @Value("${kafka.producer.retries}")
        private int retries;
        @Value("${kafka.producer.batch.size}")
        private int batchSize;
        @Value("${kafka.producer.linger}")
        private int linger;
        @Value("${kafka.producer.buffer.memory}")
        private int bufferMemory;
        /**配置*/
        public Map<String, Object> producerConfigs() {
            Map<String, Object> props = new HashMap<>();
            props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, servers);
            props.put(ProducerConfig.RETRIES_CONFIG, retries);
            props.put(ProducerConfig.BATCH_SIZE_CONFIG, batchSize);
            props.put(ProducerConfig.LINGER_MS_CONFIG, linger);
            props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, bufferMemory);
            props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
            props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
            return props;
        }
        /**生产者工厂*/
        public ProducerFactory<String, String> producerFactory() {
            return new DefaultKafkaProducerFactory<>(producerConfigs());
        }
        /**kafka模板*/
        @Bean
        public KafkaTemplate<String, String> kafkaTemplate() {
            return new KafkaTemplate<>(producerFactory());
        }
    }
 ```
 ### consumer configuration
```java
    @Configuration
    @EnableKafka
    public class KafkaConsumerConfig {
        @Value("${kafka.servers}")
        private String servers;
        @Value("${kafka.consumer.enable.auto.commit}")
        private boolean enableAutoCommit;
        @Value("${kafka.consumer.session.timeout}")
        private String sessionTimeout;
        @Value("${kafka.consumer.auto.commit.interval}")
        private String autoCommitInterval;
        @Value("${kafka.consumer.group.id}")
        private String groupId;
        @Value("${kafka.consumer.auto.offset.reset}")
        private String autoOffsetReset;
        @Value("${kafka.consumer.concurrency}")
        private int concurrency;
        /**监听器工厂*/
        @Bean
        public KafkaListenerContainerFactory<ConcurrentMessageListenerContainer<String, String>> kafkaListenerContainerFactory() {
            ConcurrentKafkaListenerContainerFactory<String, String> factory = new ConcurrentKafkaListenerContainerFactory<>();
            factory.setConsumerFactory(consumerFactory());
            factory.setConcurrency(concurrency);
            factory.getContainerProperties().setPollTimeout(1500);
            return factory;
        }
        /**消费者工厂*/
        public ConsumerFactory<String, String> consumerFactory() {
            return new DefaultKafkaConsumerFactory<>(consumerConfigs());
        }
        /**配置*/
        public Map<String, Object> consumerConfigs() {
            Map<String, Object> propsMap = new HashMap<>();
            propsMap.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, servers);
            propsMap.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, enableAutoCommit);
            propsMap.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, autoCommitInterval);
            propsMap.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, sessionTimeout);
            propsMap.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
            propsMap.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
            propsMap.put(ConsumerConfig.GROUP_ID_CONFIG, groupId);
            propsMap.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, autoOffsetReset);
            return propsMap;
        }
        /**监听器*/
        @Bean
        public Listener listener() {
            return new Listener();
        }
    }
```
### listener
```java
    public class Listener {
        @KafkaListener(topics = "test")
        public void listen(ConsumerRecord<?, ?> record){
            System.out.println("kafka的key: " + record.key());
            System.out.println("kafka的value: " + record.value().toString());
        }
    }
```
### 请求接口
```java
    @RestController
    @RequestMapping("/kafka")
    public class KafkaController {
        @Autowired
        private KafkaTemplate kafkaTemplate;
        @RequestMapping(value = "/send", method = RequestMethod.GET)
        public void sendMessage(HttpServletRequest request, HttpServletResponse response) {
            String message = request.getParameter("message");
            kafkaTemplate.send("test", "key", message);
            System.out.println("发送成功");
        }
    }
```
### 可能出现的异常
    
    无法连接注意要将port打开
    配置server.properties中的listeners=PLAINTEXT://{ip}:9092  # ip可以内网、外网ip、127.0.0.1 或域名
    将链接地址改为PLAINTEXT配置的地址
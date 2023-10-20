---
title: Integrate RocketMQ 5.0 client with Spring - 记录本次开发的思考与实践
date: 2023-10-20 20:11:50
tags: 开源之夏
---
# 前言

RocketMQ-Spring 目前已经支持了4.x Remoting SDK，我们还需要支持RocketMQ 5.0 gRPC SDK，即是对 rocketmq-clients-SDK 进行封装和整合，最终以 rocketmq-spring-v5-starter 的形式呈现给用户。
<a name="bBac3"></a>

## Spring 中的消息框架

Spring Messaging 是Spring Framework 4中添加的模块，是Spring与消息系统集成的一个扩展性的支持。它实现了从基于JmsTemplate的简单的使用JMS接口到异步接收消息的一整套完整的基础架构，Spring AMQP提供了该协议所要求的类似的功能集。 在与Spring Boot的集成后，它拥有了自动配置能力，能够在测试和运行时与相应的消息传递系统进行集成。<br />单纯对于客户端而言，Spring Messaging 提供了一套抽象的 API 或者说是约定的标准，对消息发送端和消息接收端的模式进行规定，不同的消息中间件提供商可以在这个模式下提供自己的Spring实现：在消息发送端需要实现的是一个 **XXXTemplate** **形式的 Java Bean**，结合 Spring Boot 的自动化配置选项提供多个不同的发送消息方法；在消息的消费端是一个 **XXXMessageListener 接口**（实现方式通常会使用一个注解来声明一个消息驱动的 POJO），提供回调方法来监听和消费消息，这个接口同样可以使用Spring Boot的自动化选项和一些定制化的属性。参考 Spring Messaging 的既有实现，RocketMQ的 spring-boot-starter 中遵循了相关的设计模式并结合 RocketMQ 自身的功能特点提供了相应的API (如，顺序，异步和事务半消息等)。
<a name="HK0Gx"></a>

## 模块与适配

按照 Spring Boot 的规范，原有的 4.X SDK 划分为四个子模块，我们 v5 SDK 也将延用分成四个子模块的方式完成开发，在这一点上，会参考 4.X 的模块设计。<br />![image.png](https://cdn.nlark.com/yuque/0/2023/png/25620162/1697387941724-eb0b85d5-0519-4002-b62b-9bfb6e7c2e34.png#averageHue=%23eef0f4&clientId=ub125c5f9-33da-4&from=paste&height=194&id=u7597b39a&originHeight=363&originWidth=693&originalType=binary&ratio=1.875&rotation=0&showTitle=false&size=60312&status=done&style=none&taskId=ufe606334-5aea-40d0-8112-16d02215730&title=&width=369.6)

- rocketmq-v5-client-spring-boot-parent （父pom文件，定义相关的依赖管理和Plugin，供其它几个模块引用）
- rocketmq-v5-client-spring-boot (定义 auto-configuration 实现，具体RocketMQ相关的自动配置和 Bean 创建代码都集中在这里)
- rocketmq-v5-client-spring-starter (将 rocketmq-v5-client-spring-boot 和其它的依赖打包生成全量的依赖，用户引用它即可完成所有 rocketmq-spring 的客户端操作)
- rocketmq-v5-client-spring-samples (使用示例，展示如何使用 spring-boot 方式发送和消费消息)

为了更方便理解模块之间的依赖关系，我做了一个模块依赖图，图示如下，其中rocketmq-grpc-spring-boot-parent 等含有 grpc 写法皆代表 v5 模块，就不再更改了。<br />![image.png](https://cdn.nlark.com/yuque/0/2023/png/25620162/1697388054241-08253c4e-6430-4d72-8870-a44eccb3d589.png#averageHue=%23dce4e8&clientId=ub125c5f9-33da-4&from=paste&height=476&id=uab7765bb&originHeight=892&originWidth=1331&originalType=binary&ratio=1.875&rotation=0&showTitle=false&size=569159&status=done&style=none&taskId=udd4cda05-ef93-44ce-988b-6c1230e0855&title=&width=709.8666666666667)<br />可以看到，我们在rocketmq-spring-all项目顶级模块之下，又开发了rocketmq-grpc-spring-boot-parent 新模块，他是我们其他 maven 子模块的父模块。所有的子模块都继承于父模块，项目中所有要使用到的 jar 包的版本都集中由父工程管理。 rocketmq-grpc-spring-boot-starter 会发布到maven仓库中，用户需要使用时只需要引入这个 starter 即可使用。
<a name="bBD1O"></a>

# 设计思路

<a name="v0wL4"></a>

## 添加maven依赖

项目需在 pom.xml 文件中引入SDK5.0 依赖 rocketmq-client-java 依赖，并且设置 rocketmq-client-java-version 最新版本

```xml
<dependency>
  <groupId>org.apache.rocketmq</groupId>
  <artifactId>rocketmq-client-java</artifactId>
  <version>${rocketmq-client-java-version}</version>
</dependency> 
```

<a name="CSLKP"></a>

## 适配spring-boot自动化配置

<a name="MsXpd"></a>

### 支持适配新客户端实体

目前 rocketmq-spring 中基于 4.x SDK 实现的自动配置类主要是两个bean：

``` java
    public static final String PRODUCER_BEAN_NAME = "defaultMQProducer";
    public static final String CONSUMER_BEAN_NAME = "defaultLitePullConsumer";
```

引入 v5-sdk 之后，在 template 中需要加入 IOC 容器进行管理的有以下几个bean：

``` java
    private ProducerBuilder producerBuilder;
    private SimpleConsumerBuilder simpleConsumerBuilder;
    private Producer producer;
    private SimpleConsumer simpleConsumer;
```

在确定好我们需要进行控制反转注入ioc容器的bean有哪些后，就要考虑如何将他们交给ioc容器管理。按照目前rocketmq-spring给出的@Configuration+@Bean注解的方式，我们同样可以利用这些注解来进行对producer、pushConsumer和simpleConsumer进行管理。<br />但是 PushConsumer 需要交给 DefaultListenerContainer 容器进行管理，而不是放到 <br />RocketMQGRpcTemplate容器中。这里为什么要让 IOC 容器管理额外的 builder 对象（像ProducerBuilder、SimpleConsumerBuilder 以及DefaultListenerContainer容器中的 PushConsumerBuilder），⽽不是只管理调⽤build()⽅法之后构建的3个真正的⽣产消费者实体bean， 乃是基于以下的考量： <br />我们在RocketMQAutoConfiguration这个⾃动配置类中，需要构造producer和consumer，但是此时我 们的producer尚不是⼀个完整的producer，因为如果⽤户要发送事务消息，是需要设置⼀个 TransactionChecker.

``` java
ProducerBuilder setTransactionChecker(TransactionChecker checker);
```

也就是说，如果我们在 RocketMQAutoConfiguration ⾥⾯构建bean时，就调⽤build⽅法去构建⼀个完整的 producer 对象，那么⽤户在使⽤我们的核⼼类 RocketmqTemplate 使⽤⽣产者发送事务消息时就只 能传⼊⼀个 TransactionChecker 的实现类作为参数去设置Producer的transactionChecker 属性，⽽ Producer ⼀旦被 build 构建，我们是⽆法再去修改的，所以在 RocketMQAutoConfiguration ⾥⾯构建 bean 时只能构建⼀个ProducerBuilder，等待⽤户实现的 TransactionChecker 参数设置进去。 <br />同理的，我们的pushConsumer也是需要⼀个回调函数去处理 MessageView 的逻辑，所以这⾥也是需要先构建好⼀个builder，最后等到所有参数注⼊完成后才可以再去调⽤build⽅法⽣成⼀个真正的实例。
<a name="nz0eP"></a>

### 装配Producer

我们需要实现对 5.0SDK 中的Producer进⾏管理：<br />(1)引⼊Producer属性的配置类<br />我们通过在RocketMQProperties类中定义我们的Producer实例引⼊的配置⽂件的配置

``` java
public static class Producer {
 /**
 * The property of "access-key".
 */
 private String accessKey;
 /**
 * The property of "secret-key".
 */
 private String secretKey;
 /**
 * Proxy address and port list
 */
 private String endpoints;
 /**
 * topic is used to prefetch the route
 */
 private String topic;
//此处省略对应的getter/setter代码
 }
```

<a name="fMooc"></a>

#### 代码实现

首先我们需要对 5.0 SDK 中的 producer 进行管理，通过 RocketMQProperties 引入的参数直接对 producer 进行构造。

``` java
    /**
     * description:gRPC-SDK ProducerBuilder
     */
    @Bean(PRODUCER_BUILDER_BEAN_NAME)
    @ConditionalOnMissingBean(ProducerBuilderImpl.class)
    @ConditionalOnProperty(prefix = "rocketmq", value = {"producer.endpoints"})
    public ProducerBuilder producerBuilder(RocketMQProperties rocketMQProperties) {
        RocketMQProperties.Producer rocketMQProducer = rocketMQProperties.getProducer();
        log.info("Init Producer Args: " + rocketMQProducer);
        String topic = rocketMQProducer.getTopic();
        String endPoints = rocketMQProducer.getEndpoints();
        Assert.hasText(endPoints, "[rocketmq.producer.endpoints] must not be null");
        ClientConfiguration clientConfiguration = RocketMQUtil.createProducerClientConfiguration(rocketMQProducer);
        final ClientServiceProvider provider = ClientServiceProvider.loadService();
        ProducerBuilder producerBuilder;
        producerBuilder = provider.newProducerBuilder()
                .setClientConfiguration(clientConfiguration)
                // Set the topic name(s), which is optional but recommended. It makes producer could prefetch the topic
                // route before message publishing.
                .setTopics(rocketMQProducer.getTopic())
                .setMaxAttempts(rocketMQProducer.getMaxAttempts());
        log.info(String.format("a producer init on proxy %s", endPoints));
        return producerBuilder;
    }
```

可以看到，由于需要⽀持 v5 版本的 producer，与原有 4.x 的内部类 Producer 不同，换新了以下字段：

- Endpoints字段：我们知道endpoints是接⼊点地址，需要设置成Proxy的地址和端⼝列表。RocketMQ Proxy的出现，解决了4.X版本客户端多语⾔客户端实现Remoing协议难度⼤、复杂、功能不⼀致、维护⼯作⼤的问题。使⽤业界熟悉的gRPC协议， 各个语⾔代码统⼀、简单，使得多语⾔使⽤RocketMQ更⽅便容易。
- Topic字段：在4.x SDK的Producer中，我们通常在messge中设置topic，⽽不在producer中直接设置topic。但是在5.0 SDK中，我们既可以在producer，也可以在message中指定topic。这样做的好处是，⽣产者能够在发送消息之前预先获取该主题的路由信息，并缓存到本地。这样，当⽣产者要发送消息时，它就可以直接从本地缓存中获取路由信息，避免了每次发送消息都需要进⾏路由查找的问题，从⽽提升了发送消息的效率和性能。

(2)加⼊配置属性注解在RocketMQProperties上加⼊注解@ConfigurationProperties(prefix = "rocketmq")，能够引⼊配置⽂件中的⽤户⾃定义属性。

``` java
@ConfigurationProperties(prefix = "rocketmq")
public class RocketMQProperties {
    
}
```

(3)注⼊bean<br />我们需要在 RocketMQAutoConfiguration 中配置交给 IOC 容器管理的 bean，这⾥的 bean 只能是⼀个 ProducerBuilder，⽽不是 Producer，具体原因在上⽂已经阐述。

``` java
    /**
     * description:gRPC-SDK PushConsumer
     */
    @Bean(PUSH_CONSUMER_BEAN_NAME)
    @ConditionalOnMissingBean(PushConsumerBuilder.class)
    @ConditionalOnProperty(name = "rocketmq.sdk-version", havingValue = "grpc")
    //@ConditionalOnProperty(prefix = "rocketmq", value = {"producer.endpoints"})
    public PushConsumerBuilder pushConsumerBuilder(RocketMQProperties rocketMQProperties) {
        //此处getConsumer返回一个pushConsumer,getPullConsumer返回一个pullConsumer
        RocketMQProperties.PushConsumer rocketMQPushConsumer = rocketMQProperties.getConsumer();
        final ClientServiceProvider provider = ClientServiceProvider.loadService();
        SessionCredentialsProvider sessionCredentialsProvider =
                new StaticSessionCredentialsProvider(rocketMQPushConsumer.getAccessKey(), rocketMQPushConsumer.getSecretKey());
        ClientConfiguration clientConfiguration = ClientConfiguration.newBuilder()
                .setEndpoints(rocketMQPushConsumer.getEndpoints())
                .setCredentialProvider(sessionCredentialsProvider)
                .build();
        FilterExpression filterExpression = new FilterExpression(rocketMQPushConsumer.getTag(), FilterExpressionType.TAG);
        final PushConsumerBuilder builderImpl;
        builderImpl = provider.newPushConsumerBuilder()
                .setClientConfiguration(clientConfiguration)
                // Set the consumer group name.
                .setConsumerGroup(rocketMQPushConsumer.getGroup())
                // Set the subscription for the consumer.
                .setSubscriptionExpressions(Collections.singletonMap(rocketMQPushConsumer.getTopic(), filterExpression));
        return builderImpl;
    }
```

(4)配置加载执⾏的先后顺序<br />我们可以看到在RocketMQAutoConfiguration这个类上有以下注解，表明RocketMQAutoConfiguration 需要在TransactionConfiguration之前被加载和执⾏

``` java
@AutoConfigureBefore({TransactionConfiguration.class})
```

(5)引⼊事务处理注解<br />我们也需要⼀个⾃定义的 TransactionConfiguration 配置类，⽤来注⼊我们刚刚所说的，⽣产者所需要的 TransactionChecker 来执⾏回调函数。⾸先我们实现SmartInitializingSingleton 接⼝，在bean初始化完成后执⾏⼀些初始化操作：获取被@RocketMQTransactionListener 标记的类，这个类是需要⽤户实现<br />TransactionChecker 接⼝的，handleTransactionChecker ⽅法会⾃动将这个实现类给注⼊到 rocketmqTemplate 管理的 ProducerBuilder 对象中。<br />这⾥的 template 是根据 @RocketMQTransactionListener注解中的rocketMQTemplateBeanName 值来从容器上下⽂中提取的，template 是可拓展的，也就是我们所说的 @ExtTemplateConfiguration

``` java
        @Configuration
        public class TransactionConfiguration implements ApplicationContextAware,
                SmartInitializingSingleton {
            private ConfigurableApplicationContext applicationContext;
            @Override
            public void setApplicationContext(ApplicationContext applicationContex
t) throws BeansException {
                this.applicationContext = (ConfigurableApplicationContext) applica
                tionContext;
            }
            //获取被@RocketMQTransactionListener标记的类
            @Override
            public void afterSingletonsInstantiated() {
                Map<String, Object> beans = this.applicationContext.getBeansWithAn
                notation(TransactionListener.class)
                        .entrySet().stream().filter(entry -> !ScopedProxyUtils.isS
                                copedTarget(entry.getKey()))
                        .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::ge
                                tValue));
                beans.forEach(this::handleTransactionChecker);
            }
            public void handleTransactionChecker(String beanName, Object bean) {
                Class<?> clazz = AopProxyUtils.ultimateTargetClass(bean);
                if (!TransactionChecker.class.isAssignableFrom(bean.getClass())) {
                    throw new IllegalStateException(clazz + " is not instance of "
                            + TransactionChecker.class.getName());
                }
                TransactionListener annotation = clazz.getAnnotation(TransactionLi
                        stener.class);
                if (Objects.isNull(annotation)) {
                    throw new IllegalStateException("The transactionListener annot
                            ation is missing");
                }
                //获取注解上的template,默认为RocketMQGRpcTemplate
                RocketMQGRpcTemplate rocketMQTemplate = (RocketMQGRpcTemplate) app
                licationContext.getBean(annotation.rocketMQTemplateBeanName());
                if ((rocketMQTemplate.getProducerBuilder()) != null) {
                    rocketMQTemplate.getProducerBuilder().setTransactionChecker((T
                            ransactionChecker) bean);
                }
            }
        }
```

(6)⽣产者拓展的实现<br />我们可以⽤⼀个 @ExtTemplateConfiguration 注解来拓展我们的rocketmqTemplate，当⽤户需要多个不同的⽣产者时，可以使⽤这个注解来拓展装配不同的⽣产者。

``` java
        @Target(ElementType.TYPE)
        @Retention(RetentionPolicy.RUNTIME)
        @Documented
        @Component
        public @interface ExtTemplateConfiguration {
            /**
             * The component name of the Producer configuration.
             */
            String value() default "";
            /**
             * The property of "access-key".
             */
            String accessKey() default "${rocketmq.producer.accessKey:}";
            /**
             * The property of "secret-key".
             */
            String secretKey() default "${rocketmq.producer.secretKey:}";
            /**
             * Proxy address and port list
             */
            String endpoints() default "${rocketmq.producer.endpoints:}";
            /**
             * topic is used to prefetch the route
             */
            String topic() default "${rocketmq.producer.topic:}";
        }
```

(7)容器拓展类的注册<br />对于这个拓展 ExtRocketmqTemplate 的装配，我们可以在⾃定义配置类<br />ExtProducerResetConfiguration 中构造。与上⽂中 TransactionConfiguration 配置类似的，⾸先获取 @ExtTemplateConfiguration 注解下的所有 bean，然后获取@ExtTemplateConfiguration 注解的所有拓展属性值，装配出⼀个 ProducerBuilder并注⼊到⾃定义的拓展 ExtRocketmqTemplate 中。
<a name="auMY3"></a>

#### 新增字段

由于在 rocketmq-spring 的基于 4.x remoting 的源代码中，RocketMQProperties 类中的静态内部类 PushConsumer 是继承 PullConsumer 的，所以我在 PullConsumer 的字段中加入了以下两个属性：<br />**Tag属性：**设置消息 Tag，用于消费端根据指定Tag过滤消息<br />**EndPoints属性：**同上 Producer 的 Endpoints 介绍：我们知道 endpoints 是接入点地址，需要设置成 Proxy 的地址和端口列表。有了 RocketMQ Proxy 的出现，无疑解决了4.X版本客户端多语言客户端实现 Remoing 协议难度大、复杂、功能不一致、维护工作大的问题。使用业界熟悉的GRPC协议， 各个语言代码统一、简单，使得多语言使用RocketMQ更方便容易。<br />**FilterExpressionType属性：**消费者过滤表达式的类型。RocketMQ 支持两种过滤表达式类型：TAG 和 SQL92。如果将 FilterExpressionType 设置为 TAG，则消费者只会消费指定标签的消息TAG 表达式类型比较简单，性能相对较好，适用于只需要简单标签过滤的场景。如果将 FilterExpressionType 设置为 SQL92，则消费者可以使用 SQL92 的语法来过滤消息。SQL92 表达式类型更加灵活，可以支持更复杂的过滤规则，但是由于需要解析 SQL92 语法，所以性能相对较差。我们可以根据实际情况，选择合适的过滤表达式类型可以提高消费者的性能和效率。

``` java
        /**
         * Tag of consumer.
         */
        private String tag;

        /**
         * Proxy address and port list
         */
        private String endpoints;

        /**
         * filterExpressionType
         */
        private String filterExpressionType = "tag";
```

<a name="BQi9T"></a>

### 装配SimpleConsumer

<a name="tqnG1"></a>

#### 代码实现

``` java
    /**
     * description:gRPC-SDK SimpleConsumer
     */
    @Bean(SIMPLE_CONSUMER_BEAN_NAME)
    @ConditionalOnMissingBean(SimpleConsumer.class)
    //@ConditionalOnProperty(name = "rocketmq.sdk-version", havingValue = "grpc")
    @ConditionalOnExpression("${rocketmq.sdk-version} == 'grpc' && ${rocketmq.pull-consumer.endpoints}")
    //@ConditionalOnProperty(prefix = "rocketmq", value = {"producer.endpoints"})
    public SimpleConsumer simpleConsumer(RocketMQProperties rocketMQProperties) {
        //此处getConsumer返回一个pushConsumer,getPullConsumer返回一个pullConsumer
        RocketMQProperties.PullConsumer rocketMQPullConsumer = rocketMQProperties.getPullConsumer();
        final ClientServiceProvider provider = ClientServiceProvider.loadService();
        SessionCredentialsProvider sessionCredentialsProvider =
                new StaticSessionCredentialsProvider(rocketMQPullConsumer.getAccessKey(), rocketMQPullConsumer.getSecretKey());
        ClientConfiguration clientConfiguration = ClientConfiguration.newBuilder()
                .setEndpoints(rocketMQPullConsumer.getEndpoints())
                .setCredentialProvider(sessionCredentialsProvider)
                .build();
        Duration awaitDuration = Duration.ofSeconds(30);
        FilterExpressionType filterExpression = "tag".equals(rocketMQPullConsumer.getTag()) ? FilterExpressionType.TAG : FilterExpressionType.SQL92;
        FilterExpression expression = new FilterExpression(rocketMQPullConsumer.getTag(), filterExpression);
        SimpleConsumer simpleConsumer;
        try {
            simpleConsumer = provider.newSimpleConsumerBuilder()
                    .setClientConfiguration(clientConfiguration)
                    // Set the consumer group name.
                    .setConsumerGroup(rocketMQPullConsumer.getGroup())
                    // set await duration for long-polling.
                    .setAwaitDuration(awaitDuration)
                    // Set the subscription for the consumer.
                    .setSubscriptionExpressions(Collections.singletonMap(rocketMQPullConsumer.getTopic(), expression))
                    .build();
        } catch (ClientException e) {
            throw new RuntimeException(e);
        }
        return simpleConsumer;
    }
```

<a name="xXmQb"></a>

#### 新增字段

同PushConsumer
<a name="YgZ22"></a>

#### 注解控制实体装配

``` java
@ConditionalOnExpression("${rocketmq.sdk-version} == 'grpc' && ${rocketmq.pull-consumer.endpoints}")
```

@ConditionalOnExpression注解表示：只有当用户**自定义配置属性rocketmq.sdk-version为grpc时**，并且属性**rocketmq.pull-consumer.endpoints不为空**，才会将这个bean交给ioc容器管理，否则不会加载它。
<a name="AUiC5"></a>

# 封装消息发送方法

我们知道，RocketMQ-Spring 是遵循**Spring Messaging API**规范的，RocketMQ-Spring 通过实现 RocketMQTemplate 完成消息的发送。<br />RocketMQTemplate 继承 AbstractMessageSendingTemplate 抽象类，来支持 Spring Messaging API 标准的消息转换和发送方法，这些方法最终会代理给 doSend 方法，doSend 方法会最终调用 syncSend，由 DefaultMQProducer 实现。
<a name="tRrlr"></a>

### remoting SDK消息发送

首先我们先来分析一下基于rocketmq 4.x remoting SDK的消息发送封装类型：<br />![](https://cdn.nlark.com/yuque/0/2023/png/25620162/1697784558970-f902ba5f-0712-4ee6-9b84-e674c653300f.png#averageHue=%23ebedf2&clientId=ud3ed0db8-f6ac-4&from=paste&id=u4170e8c1&originHeight=1176&originWidth=1204&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=u90d94f72-aadf-45e9-b206-748676b8da1&title=)
<a name="TNV84"></a>

#### 同步消息的发送

``` java
//同步模式发送延迟消息（自定义超时时间）
private SendResult syncSend(String destination, Message<?> message, long timeout, long delayTime, DelayMode mode)
//同步模式发送延迟消息（自定义超时层级）
public SendResult syncSend(String destination, Message<?> message, long timeout, int delayLevel)
//同步模式发送批量消息
public <T extends Message> SendResult syncSend(String destination, Collection<T> messages, long timeout)
```

<a name="ae0vZ"></a>

#### 异步消息的发送

``` java
//异步模式发送延迟消息
public void asyncSend(String destination, Message<?> message, SendCallback sendCallback, long timeout, int delayLevel)
//异步模式发送延迟批量消息
public <T extends Message> void asyncSend(String destination, Collection<T> messages, SendCallback sendCallback, long timeout)
```

<a name="kubi9"></a>

#### 顺序消息的发送

``` java
//同步模式发送顺序延迟消息
public SendResult syncSendOrderly(String destination, Message<?> message, String hashKey, long timeout, int delayLevel)
//同步模式发送批量顺序延迟消息
public <T extends Message> SendResult syncSendOrderly(String destination, Collection<T> messages, String hashKey, long timeout) {
//异步模式发送顺序延迟消息
public void asyncSendOrderly(String destination, Message<?> message, String hashKey, SendCallback sendCallback, long timeout, int delayLevel)
```

<a name="I5gUv"></a>

#### 单向消息的发送

``` java
//单向模式的消息发送，忽略响应
public void sendOneWay(String destination, Message<?> message)
//单向模式的消息顺序发送，忽略响应
public void sendOneWayOrderly(String destination, Message<?> message, String hashKey)
```

<a name="u5WLJ"></a>

#### 事务消息的发送

``` java
//事务消息的发送
public TransactionSendResult sendMessageInTransaction(final String destination, final Message<?> message, final Object arg)
```

<a name="Tp5yJ"></a>

#### **Request-Reply 消息的发送**

Request-Reply 消息指的是上游服务投递消息后进入等待被通知的状态，直到消费端返回结果并返回给发送端。

``` java
//Request-Reply 消息发送
public <T> T sendAndReceive(String destination, Message<?> message, Type type, String hashKey,long timeout, int delayLevel)
```

<a name="GoGxN"></a>

### v5 SDK消息发送

基于上述remoting SDK消息的发送，我们需要进一步整合进 v5 SDK到 rocketmq-spring中。<br />在rocketmq-client-java项目中，我们依据Producer接口下的方法，首先可以粗略地将生产者发送消息的方法归结为以下**三类**：<br />（1）Producer发送同步消息：

``` java
SendReceipt send(Message message) throws ClientException;
```

（2）Producer发送事务消息：

``` java
SendReceipt send(Message message, Transaction transaction) throws ClientException;
```

（3）Producer发送异步消息：

``` java
CompletableFuture <SendReceipt> sendAsync(Message message);
```

但是，实际上在rocketmq-client-java中，我们可以发送不同类型的消息，**官方文档**有很清楚的记录：<br />![image.png](https://cdn.nlark.com/yuque/0/2023/png/25620162/1678815467073-0fbff046-26c7-4e6f-ad84-2cc726a8d4b4.png#averageHue=%23ededed&clientId=u4b1f5728-cd06-4&from=paste&height=155&id=M9gBW&originHeight=291&originWidth=421&originalType=binary&ratio=1.875&rotation=0&showTitle=false&size=19238&status=done&style=none&taskId=u024db69d-1e8b-46da-92bc-258cb2a3a3a&title=&width=224.53333333333333)![image.png](https://cdn.nlark.com/yuque/0/2023/png/25620162/1678815512595-73750cd8-0c5d-4cd3-b10f-7d14e494c6d6.png#averageHue=%23f0f0f0&clientId=u4b1f5728-cd06-4&from=paste&height=157&id=Q14Xn&originHeight=295&originWidth=438&originalType=binary&ratio=1.875&rotation=0&showTitle=false&size=16448&status=done&style=none&taskId=ub0ade7c0-8266-40c1-a42c-adad2a52ca3&title=&width=233.6)

基于rocketmq-client-java 5.0 SDK，我们需要整合的消息发送方法可以具体分为以下几类：<br />（1）Producer发送普通消息（NormalMessage）

``` java
final Message message = provider.newMessageBuilder()
            // Set topic for the current message.
            .setTopic(topic)
            // Message secondary classifier of message besides topic.
            .setTag(tag)
            // Key(s) of the message, another way to mark message besides message id.
            .setKeys("yourMessageKey-1c151062f96e")
            .setBody(body)
            .build();
final SendReceipt sendReceipt = producer.send(message);
```

（2）Producer发送顺序消息（FIFOMessage）<br />顺序消息和普通消息的不同之处就在于，需要**设置消息组、保证单一生产者**并且**串行发送。**

``` java
final Message message = provider.newMessageBuilder()
            // Set topic for the current message.
            .setTopic(topic)
            // Message secondary classifier of message besides topic.
            .setTag(tag)
            // Key(s) of the message, another way to mark message besides message id.
            .setKeys("yourMessageKey-1ff69ada8e0e")
            // Message group decides the message delivery order.
            .setMessageGroup("yourMessageGroup0")
            .setBody(body)
            .build();
final SendReceipt sendReceipt = producer.send(message);
```

（3）Producer发送延迟消息（DelayMessage）<br />对message设置触发时间即可，例如示例代码中我们可以对message进行如下处理：

``` java
final Message message = provider.newMessageBuilder()
            // Set topic for the current message.
            .setTopic(topic)
            // Message secondary classifier of message besides topic.
            .setTag(tag)
            // Key(s) of the message, another way to mark message besides message id.
            .setKeys("yourMessageKey-3ee439f945d7")
            // Set expected delivery timestamp of message.
            .setDeliveryTimestamp(System.currentTimeMillis() + messageDelayTime.toMillis())
            .setBody(body)
            .build();
final SendReceipt sendReceipt = producer.send(message);
```

（4）Producer发送事务消息（TransactionMessage）

``` java
final Transaction transaction = producer.beginTransaction();
        // Define your message body.
final Message message = provider.newMessageBuilder()
            // Set topic for the current message.
            .setTopic(topic)
            // Message secondary classifier of message besides topic.
            .setTag(tag)
            // Key(s) of the message, another way to mark message besides message id.
            .setKeys("yourMessageKey-565ef26f5727")
            .setBody(body)
            .build();
final SendReceipt sendReceipt = producer.send(message, transaction);
```

（5）Producer发送异步消息

``` java
final Message message = provider.newMessageBuilder()
            // Set topic for the current message.
            .setTopic(topic)
            // Message secondary classifier of message besides topic.
            .setTag(tag)
            // Key(s) of the message, another way to mark message besides message id.
            .setKeys("yourMessageKey-0e094a5f9d85")
            .setBody(body)
            .build();
        // Set individual thread pool for send callback.
final CompletableFuture<SendReceipt> future = producer.sendAsync(message);
```

那么接下来，我们将围绕以上的五种消息，将消息发送整合到现有的 rocketmq-spring 框架中来。我们可以首先做一个简略的基于v5 SDK整合的思维导图：<br />![](https://cdn.nlark.com/yuque/0/2023/png/25620162/1697785106955-91cf54f3-10cb-40ff-8cd2-50772e984d18.png#averageHue=%23eeeff3&clientId=ud3ed0db8-f6ac-4&from=paste&id=ue8f73455&originHeight=1118&originWidth=1190&originalType=url&ratio=2&rotation=0&showTitle=false&status=done&style=none&taskId=u48a17001-4cb0-4395-a0e2-0f3be6fe8d9&title=)
<a name="VwLJs"></a>

#### 同步消息的发送

在RocketmqTemplate中我们定义**syncSendGrpcMessage方法**，代码如下：<br />该方法主要是利用配置文件中的参数之前在ioc容器中管理的bean：grpcProducer，调用它的send方法达到发送同步消息的目的，其中message由spring的message转换为了client的message，作为grpcProducer.syncSendGrpcMessage方法的参数。

``` java
    /**
     * @param destination      formats: `topicName:tags`
     * @param message          {@link org.springframework.messaging.Message} the message to be sent.
     * @param messageDelayTime Time for message delay
     * @param messageGroup     message group name
     * @return SendReceipt Synchronous Task Results
     */
    public SendReceipt syncSendGrpcMessage(String destination, Message<?> message, Duration messageDelayTime, String messageGroup) {
        if (Objects.isNull(message) || Objects.isNull(message.getPayload())) {
            log.error("send request message failed. destination:{}, message is null ", destination);
            throw new IllegalArgumentException("`message` and `message.payload` cannot be null");
        }
        SendReceipt sendReceipt = null;
        try {
            org.apache.rocketmq.client.apis.message.Message rocketMsg = this.createRocketMqgRPCMessage(destination, message, messageDelayTime, messageGroup);
            try {
                sendReceipt = grpcProducer.send(rocketMsg);
                log.info("Send message successfully, messageId={}", sendReceipt.getMessageId());
            } catch (Throwable t) {
                log.error("Failed to send message", t);
            }
            grpcProducer.close();
        } catch (Exception e) {
            log.error("send request message failed. destination:{}, message:{} ", destination, message);
            throw new MessagingException(e.getMessage(), e);
        }
        return sendReceipt;
    }
```

这个方法与remoting SDK类似，remoting SDK调用了消息转换的方法createRocketMqMessage，而我们这里并不需要一个<org.apache.rocketmq.common.message.Message>类型的消息实体对象，而是需要一个<org.apache.rocketmq.client.apis.message.Message>类型的消息实体对象，所以这里我又自定义了一个构建基于 v5-SDK 的消息构造方法：**createRocketMqMessage**。<br />下面是 **createRocketMqMessage **方法具体源码：

``` java
    private org.apache.rocketmq.client.apis.message.Message createRocketMQMessage(String destination, Message<?> message, Duration messageDelayTime, String messageGroup) {
        Message<?> msg = this.doConvert(message.getPayload(), message.getHeaders(), null);
        return RocketMQUtil.convertToClientMessage(getMessageConverter(), charset,
                destination, msg, messageDelayTime, messageGroup);
    }
```

这里面调用了 RocketMQUtil 中的静态方法 convertToClientMessage 用来构造一个真正的<org.apache.rocketmq.client.apis.message>消息实例。<br />下面是** convertToClientMessage **方法代码：

``` java
    public static org.apache.rocketmq.client.apis.message.Message convertToClientMessage(
            MessageConverter messageConverter, String charset,
            String destination, org.springframework.messaging.Message<?> message, Duration messageDelayTime, String messageGroup) {
        Object payloadObject = message.getPayload();
        byte[] payloads;
        try {
            payloads = getPayloadBytes(payloadObject, messageConverter, charset, message);
        } catch (Exception e) {
            throw new RuntimeException("convert to gRPC message failed.", e);
        }
        return getAndWrapMessage(destination, message.getHeaders(), payloads, messageDelayTime, messageGroup);
    }
```

在获取了消息负载的byte数组后，我们需要封装消息实体了。<br />下面是方法 **getAndWrapMessage **源码：<br />在这个方法里面，我们首先进行了判空操作解析了 destination 中的 topic 和 tag，取出MessageHeaders 中的属性并放到 message 的 properties 属性中，最后建造者模式调用build 方法构造了我们所需要的 message 类。

``` java
    public static org.apache.rocketmq.client.apis.message.Message getAndWrapMessage(
            String destination, MessageHeaders headers, byte[] payloads, Duration messageDelayTime, String messageGroup) {
        if (payloads == null || payloads.length < 1) {
            return null;
        }
        if (destination == null || destination.length() < 1) {
            return null;
        }
        String[] tempArr = destination.split(":", 2);
        final ClientServiceProvider provider = ClientServiceProvider.loadService();
        org.apache.rocketmq.client.apis.message.MessageBuilder messageBuilder = null;
        // resolve header
        if (Objects.nonNull(headers) && !headers.isEmpty()) {
            Object keys = headers.get(RocketMQHeaders.KEYS);
            if (ObjectUtils.isEmpty(keys)) {
                keys = headers.get(toRocketHeaderKey(RocketMQHeaders.KEYS));
            }
            messageBuilder = provider.newMessageBuilder()
                    .setTopic(tempArr[0]);
            if (tempArr.length > 1) {
                messageBuilder.setTag(tempArr[1]);
            }
            if (StringUtils.hasLength(messageGroup)) {
                messageBuilder.setMessageGroup(messageGroup);
            }
            if (!ObjectUtils.isEmpty(keys)) {
                messageBuilder.setKeys(keys.toString());
            }
            if (Objects.nonNull(messageDelayTime)) {
                messageBuilder.setDeliveryTimestamp(System.currentTimeMillis() + messageDelayTime.toMillis());
            }
            messageBuilder.setBody(payloads);
            org.apache.rocketmq.client.apis.message.MessageBuilder builder = messageBuilder;
            headers.forEach((key, value) -> builder.addProperty(key, String.valueOf(value)));
        }
        return messageBuilder.build();
    }
```

到这里我们需要的 message 实体类就构建完成了，之后交给 rocketmq-client-java 原生的send 方法就好了。syncSend(String destination, Message<?> message, Duration messageDelayTime, String messageGroup) 这个方法的参数可以看出，它既支持 Normal消息的发送，同时也支持 Delay 消息(messageDelayTime参数可配置)和 FIFO 消息（messageGroup可配置）的发送。<br />**方法重载：**<br />相比于原生客户端需要自己去构建 RocketMQ Message（比如将对象序列化成 byte 数组放入 Message 对象），我们需要让 RocketMQTemplate 可以直接将对象、字符串或者 byte 数组作为参数发送出去【对象序列化操作由 RocketMQ-Spring 内置完成】<br />我们可以定义如下重载方法：<br />（1）Normal Message

``` java
    public SendReceipt syncSendNormalMessage(String destination, Object payload) {
        Message<?> message = MessageBuilder.withPayload(payload).build();
        return syncSendGrpcMessage(destination, message, null, null);
    }

    public SendReceipt syncSendNormalMessage(String destination, String payload) {
        Message<?> message = MessageBuilder.withPayload(payload).build();
        return syncSendGrpcMessage(destination, message, null, null);
    }

    public SendReceipt syncSendNormalMessage(String destination, Message<?> message) {
        return syncSendGrpcMessage(destination, message, null, null);
    }

    public SendReceipt syncSendNormalMessage(String destination, byte[] payload) {
        Message<?> message = MessageBuilder.withPayload(payload).build();
        return syncSendGrpcMessage(destination, message, null, null);
    }
```

（2）FIFO Message

``` java
    public SendReceipt syncSendFifoMessage(String destination, Object payload, String messageGroup) {
        Message<?> message = MessageBuilder.withPayload(payload).build();
        return syncSendGrpcMessage(destination, message, null, messageGroup);
    }

    public SendReceipt syncSendFifoMessage(String destination, String payload, String messageGroup) {
        Message<?> message = MessageBuilder.withPayload(payload).build();
        return syncSendGrpcMessage(destination, message, null, messageGroup);
    }

    public SendReceipt syncSendFifoMessage(String destination, byte[] payload, String messageGroup) {
        Message<?> message = MessageBuilder.withPayload(payload).build();
        return syncSendGrpcMessage(destination, message, null, messageGroup);
    }

    public SendReceipt syncSendFifoMessage(String destination, Message<?> message, String messageGroup) {
        return syncSendGrpcMessage(destination, message, null, messageGroup);
    }

```

（3）Delay Message

``` java
    public SendReceipt syncSendDelayMessage(String destination, Object payload, Duration messageDelayTime) {
        Message<?> message = MessageBuilder.withPayload(payload).build();
        return syncSendGrpcMessage(destination, message, messageDelayTime, null);
    }

    public SendReceipt syncSendDelayMessage(String destination, String payload, Duration messageDelayTime) {
        Message<?> message = MessageBuilder.withPayload(payload).build();
        return syncSendGrpcMessage(destination, message, messageDelayTime, null);
    }

    public SendReceipt syncSendDelayMessage(String destination, byte[] payload, Duration messageDelayTime) {
        Message<?> message = MessageBuilder.withPayload(payload).build();
        return syncSendGrpcMessage(destination, message, messageDelayTime, null);
    }

    public SendReceipt syncSendDelayMessage(String destination, Message<?> message, Duration messageDelayTime) {
        return syncSendGrpcMessage(destination, message, messageDelayTime, null);
    }
```

他们底层都调用了**syncSendGrpcMessage**方法。
<a name="Vxu7i"></a>

#### 异步消息的发送

我们定义了如下一个异步消息的发送方法 asyncSend。<br />类似的，他的核心方法调用了 rocketmq-client-java 原生的sendAsync异步发送消息方法：future = grpcProducer.sendAsync(rocketMsg);

``` java
    public CompletableFuture<SendReceipt> asyncSend(String destination, Message<?> message, Duration messageDelayTime, String messageGroup, CompletableFuture<SendReceipt> future) {
        if (Objects.isNull(message) || Objects.isNull(message.getPayload())) {
            log.error("send request message failed. destination:{}, message is null ", destination);
            throw new IllegalArgumentException("`message` and `message.payload` cannot be null");
        }
        Producer grpcProducer = this.getProducer();
        try {
            org.apache.rocketmq.client.apis.message.Message rocketMsg = this.createRocketMQMessage(destination, message, messageDelayTime, messageGroup);
            future = grpcProducer.sendAsync(rocketMsg);
        } catch (Exception e) {
            log.error("send request message failed. destination:{}, message:{} ", destination, message);
            throw new MessagingException(e.getMessage(), e);
        }
        return future;
    }
```

<a name="qppyQ"></a>

#### 事务消息的发送

与普通消息发送不同的是，我们需要在事务消息中调用grpcProducer.beginTransaction()开启事务，并且还要提交事务transaction.commit()。

``` java
    public Pair<SendReceipt, Transaction> sendMessageInTransaction(String destination, Object payload) throws ClientException {
        Message<?> message = MessageBuilder.withPayload(payload).build();
        return sendTransactionMessage(destination, message);
    }

    public Pair<SendReceipt, Transaction> sendMessageInTransaction(String destination, String payload) throws ClientException {
        Message<?> message = MessageBuilder.withPayload(payload).build();
        return sendTransactionMessage(destination, message);
    }

    public Pair<SendReceipt, Transaction> sendMessageInTransaction(String destination, byte[] payload) throws ClientException {
        Message<?> message = MessageBuilder.withPayload(payload).build();
        return sendTransactionMessage(destination, message);
    }
    /**
     * @param destination formats: `topicName:tags`
     * @param message     {@link Message} the message to be sent.
     * @return CompletableFuture<SendReceipt> Asynchronous Task Results
     */
    public Pair<SendReceipt, Transaction> sendTransactionMessage(String destination, Message<?> message) {
        if (Objects.isNull(message) || Objects.isNull(message.getPayload())) {
            log.error("send request message failed. destination:{}, message is null ", destination);
            throw new IllegalArgumentException("`message` and `message.payload` cannot be null");
        }
        final SendReceipt sendReceipt;
        Producer grpcProducer = this.getProducer();
        org.apache.rocketmq.client.apis.message.Message rocketMsg = this.createRocketMQMessage(destination, message, null, null);
        final Transaction transaction;
        try {
            transaction = grpcProducer.beginTransaction();
            sendReceipt = grpcProducer.send(rocketMsg, transaction);
            log.info("Send transaction message successfully, messageId={}", sendReceipt.getMessageId());
        } catch (ClientException e) {
            log.error("send request message failed. destination:{}, message:{} ", destination, message);
            throw new RuntimeException(e);
        }
        return new Pair<>(sendReceipt, transaction);
    }
```

<a name="tJBMW"></a>

# 封装消息接收方法

<a name="TgH4l"></a>

### PushConsumer消息接收

因为之前在RocketmqProperties中已经将bean：pushConsumerBuilder 注入了ioc容器中，现在只需要让用户定义消息监听处理方式，然后在初始化时将 pushConsumerBuilder所需要的参数传入就可以了。

``` java
     private void initRocketMQPushConsumer() {
        if (rocketMQMessageListener == null) {
            throw new IllegalArgumentException("Property 'rocketMQMessageListener' is required");
        }
        Assert.notNull(consumerGroup, "Property 'consumerGroup' is required");
        Assert.notNull(topic, "Property 'topic' is required");
        Assert.notNull(tag, "Property 'tag' is required");
        FilterExpression filterExpression = null;
        final ClientServiceProvider provider = ClientServiceProvider.loadService();
        if (StringUtils.hasLength(this.getTag())) {
            filterExpression = RocketMQUtil.createFilterExpression(this.getTag(),this.getType());
        }
        ClientConfiguration clientConfiguration = RocketMQUtil.createClientConfiguration(this.getAccessKey(), this.getSecretKey(), this.getEndpoints(), this.getRequestTimeout());

        PushConsumerBuilder pushConsumerBuilder = provider.newPushConsumerBuilder()
                .setClientConfiguration(clientConfiguration);
        // Set the consumer group name.
        if (StringUtils.hasLength(this.getConsumerGroup())) {
            pushConsumerBuilder.setConsumerGroup(this.getConsumerGroup());
        }
        // Set the subscription for the consumer.
        if (StringUtils.hasLength(this.getTopic()) && Objects.nonNull(filterExpression)) {
            pushConsumerBuilder.setSubscriptionExpressions(Collections.singletonMap(this.getTopic(), filterExpression));
        }
        pushConsumerBuilder
                .setConsumptionThreadCount(this.getConsumptionThreadCount())
                .setMaxCacheMessageSizeInBytes(this.getMaxCacheMessageSizeInBytes())
                .setMaxCacheMessageCount(this.getMaxCachedMessageCount())
                .setMessageListener(rocketMQListener);
        this.setPushConsumerBuilder(pushConsumerBuilder);
    }
```

<a name="DOlvg"></a>

### SimpleConsumer消息接收

<a name="IzPVR"></a>

#### 同步接收消息

我们定义了一个 receive 方法作为 SimpleConsumer 的消息接收方法，并且传入参数maxMessageNum指定了每次长轮询的最大消息数，invisibleDuration 指定了设置消息接收后的不可见持续时间。

``` java
    public List<MessageView> receive(int maxMessageNum, Duration invisibleDuration) throws ClientException {
        SimpleConsumer simpleConsumer = this.getSimpleConsumer();
        return simpleConsumer.receive(maxMessageNum, invisibleDuration);
    }
```

进行消息确认：

``` java
    public void ack(MessageView message) throws ClientException {
        SimpleConsumer simpleConsumer = this.getSimpleConsumer();
        simpleConsumer.ack(message);
    }
```

<a name="dJE3A"></a>

#### 异步接收消息

receiveAsync方法返回一个CompletableFuture类型的异步返回结果，对于这个结果的处理，用户可以在调用rocketmqTemplate.receiveAsync之后自行处理。

``` java
    public CompletableFuture<List<MessageView>> receiveAsync(int maxMessageNum, Duration invisibleDuration) throws ClientException, IOException {
        SimpleConsumer simpleConsumer = this.getSimpleConsumer();
        CompletableFuture<List<MessageView>> listCompletableFuture = simpleConsumer.receiveAsync(maxMessageNum, invisibleDuration);
        simpleConsumer.close();
        return listCompletableFuture;
    }
```

进行消息确认：

``` java
    public CompletableFuture<Void> ackAsync(MessageView messageView) {
        SimpleConsumer simpleConsumer = this.getSimpleConsumer();
        return simpleConsumer.ackAsync(messageView);
    }
```
---
title: "How to Catch Deserialization Errors in Spring-Kafka?"
source: "https://www.baeldung.com/spring-kafka-deserialization-errors"
author:
  - "[[Written by: 											Emanuel Trandafir]]"
  - "[[Emanuel Trandafir]]"
published: 2024-01-24
created: 2025-05-29
description: "Learn about Spring-Kafka's RecordDeserializationException."
tags:
  - "clippings"
---
## 1\. Overview

In this article, we’ll learn about [Spring-Kafka](https://www.baeldung.com/spring-kafka) ‘s *RecordDeserializationException*. After that, we’ll create a custom error handler to catch this exception and skip the invalid message, allowing the consumer to continue processing the next events.

This article relies on Spring Boot’s Kafka modules, which offer convenient tools for interacting with the broker. For a deeper grasp of Kafka internals, we can revisit the [fundamental concepts](https://www.baeldung.com/apache-kafka) of the platform.

## 2\. Creating a Kafka Listener

**For the code examples in this article, we’ll use a small application that listens to the topic “ *baeldung.articles.published”* and processes the incoming messages. To showcase the custom error handling, our application should continue consuming messages after encountering deserialization exceptions.**

Spring-Kafka’s version will be resolved automatically by the [parent Spring Boot](https://www.baeldung.com/spring-boot-dependency-management-custom-parent) pom. Therefore, we simply need to add the module dependency:

```xml
<dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
</dependency>
```

This module enables us to use the *@KafkaListener* annotation, an abstraction over [Kafka’s Consumer API](https://www.baeldung.com/kafka-create-listener-consumer-api). Let’s leverage this annotation to create the *ArticlesPublishedListener* component. Additionally, we’ll introduce another component, *EmailService*, that will perform an action for each of the incoming messages:

```java
@Component
class ArticlesPublishedListener {
    private final EmailService emailService;

    // constructor

    @KafkaListener(topics = "baeldung.articles.published")
    public void onArticlePublished(ArticlePublishedEvent event) {
        emailService.sendNewsletter(event.article());
    }
}

record ArticlePublishedEvent(String article) {
}
```

For the [consumer configuration](https://www.baeldung.com/spring-kafka#1-consumer-configuration), we’ll focus on defining only the properties crucial to our example. When we’re developing a production application, we can adjust these properties to suit our particular needs or externalize them to a separate configuration file:

```java
@Bean
ConsumerFactory<String, ArticlePublishedEvent> consumerFactory(
  @Value("${spring.kafka.bootstrap-servers}") String bootstrapServers
) {
    Map<String, Object> props = new HashMap<>();
    props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);
    props.put(ConsumerConfig.GROUP_ID_CONFIG, "baeldung-app-1");
    props.put(ConsumerConfig.AUTO_OFFSET_RESET_CONFIG, "earliest");
    props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
    props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, JsonDeserializer.class);
    return new DefaultKafkaConsumerFactory<>(
      props,
      new StringDeserializer(),
      new JsonDeserializer<>(ArticlePublishedEvent.class)
    );
}

@Bean
KafkaListenerContainerFactory<String, ArticlePublishedEvent> kafkaListenerContainerFactory(
  ConsumerFactory<String, ArticlePublishedEvent> consumerFactory
) {
    var factory = new ConcurrentKafkaListenerContainerFactory<String, ArticlePublishedEvent>();
    factory.setConsumerFactory(consumerFactory);
    return factory;
}
```

## 3\. Setting up the Test Environment

**To set up our testing environment, we can leverage a Kafka [Testcontainer](https://www.baeldung.com/docker-test-containers) that will seamlessly spin up a Kafka Docker container for testing:**

```java
@Testcontainers
@SpringBootTest(classes = Application.class)
class DeserializationExceptionLiveTest {

    @Container
    private static KafkaContainer KAFKA = new KafkaContainer(DockerImageName.parse("confluentinc/cp-kafka:latest"));

    @DynamicPropertySource
    static void setProps(DynamicPropertyRegistry registry) {
        registry.add("spring.kafka.bootstrap-servers", KAFKA::getBootstrapServers);
    }

    // ...
}
```

**Alongside this, we’ll need a *KafkaProducer* and an *EmailService* to validate our listener’s functionality. These components will send messages to our listener and verify their accurate processing.** To simplify the tests and avoid mocking, let’s persist all incoming articles in a list, in memory, and later access them using a getter:

```java
@Service
class EmailService { 
    private final List<String> articles = new ArrayList<>();
   
    // logger, getter

    public void sendNewsletter(String article) {
        log.info("Sending newsletter for article: " + article);
        articles.add(article);
    }
}
```

As a result, we simply need to inject the *EmailService* into our test class. Let’s continue by creating *testKafkaProducer*:

```java
@Autowired
EmailService emailService;

static KafkaProducer<String, String> testKafkaProducer;

@BeforeAll
static void beforeAll() {
    Properties props = new Properties();
    props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, KAFKA.getBootstrapServers());
    props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
    props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class.getName());
    testKafkaProducer = new KafkaProducer<>(props);
}
```

With this setup, we can already test the happy flow. Let’s publish two articles with a valid JSON and verify that our application successfully called the *emailService* for each of them:

```java
@Test
void whenPublishingValidArticleEvent_thenProcessWithoutErrors() {
    publishArticle("{ \"article\": \"Kotlin for Java Developers\" }");
    publishArticle("{ \"article\": \"The S.O.L.I.D. Principles\" }");

    await().untilAsserted(() -> 
      assertThat(emailService.getArticles())
        .containsExactlyInAnyOrder(
          "Kotlin for Java Developers", 
          "The S.O.L.I.D. Principles"
        ));
}
```

## 4\. Causing a RecordDeserializationException

**Kafka throws *RecordDeserializationException* if the configured deserializer cannot properly parse the key or value of the message. To reproduce this error, we simply need to publish a message containing an invalid JSON body**:

```java
@Test
void whenPublishingInvalidArticleEvent_thenCatchExceptionAndContinueProcessing() {
    publishArticle("{ \"article\": \"Introduction to Kafka\" }");
    publishArticle(" !! Invalid JSON !! ");
    publishArticle("{ \"article\": \"Kafka Streams Tutorial\" }");

    await().untilAsserted(() -> 
      assertThat(emailService.getArticles())
        .containsExactlyInAnyOrder(
          "Kotlin for Java Developers",
          "The S.O.L.I.D. Principles"
        ));
}
```

If we run this test and check the console, we’ll observe a recurring error being logged:

```markdown
ERROR 7716 --- [ntainer#0-0-C-1] o.s.k.l.KafkaMessageListenerContainer    : Consumer exception

**java.lang.IllegalStateException: This error handler cannot process 'SerializationException's directly; please consider configuring an 'ErrorHandlingDeserializer' in the value and/or key deserializer**
   at org.springframework.kafka.listener.DefaultErrorHandler.handleOtherException(DefaultErrorHandler.java:151) ~[spring-kafka-2.8.11.jar:2.8.11]
   at org.springframework.kafka.listener.KafkaMessageListenerContainer$ListenerConsumer.handleConsumerException(KafkaMessageListenerContainer.java:1815) ~[spring-kafka-2.8.11.jar:2.8.11]
   ...
**Caused by: org.apache.kafka.common.errors.RecordDeserializationException: Error deserializing key/value for partition baeldung.articles.published-0 at offset 1. If needed, please seek past the record to continue consumption.**
   at org.apache.kafka.clients.consumer.internals.Fetcher.parseRecord(Fetcher.java:1448) ~[kafka-clients-3.1.2.jar:na]
   ...
**Caused by: org.apache.kafka.common.errors.SerializationException: Can't deserialize data** [[32, 33, 33, 32, 73, 110, 118, 97, 108, 105, 100, 32, 74, 83, 79, 78, 32, 33, 33, 32]] from topic [baeldung.articles.published]
   at org.springframework.kafka.support.serializer.JsonDeserializer.deserialize(JsonDeserializer.java:588) ~[spring-kafka-2.8.11.jar:2.8.11]
   ...
**Caused by: com.fasterxml.jackson.core.JsonParseException: Unexpected character ('!' (code 33))**: expected a valid value (JSON String, Number, Array, Object or token 'null', 'true' or 'false')
   **at [Source: (byte[])" !! Invalid JSON !! "; line: 1, column: 3]**
   at com.fasterxml.jackson.core.JsonParser._constructError(JsonParser.java:2391) ~[jackson-core-2.13.5.jar:2.13.5]
   ...
```

Then, the test will eventually time out and fail. If we check the assertion error, we’ll notice that only the first message was successfully processed:

```markdown
org.awaitility.core.ConditionTimeoutException: Assertion condition 
Expecting actual:
  ["Introduction to Kafka"]
to contain exactly in any order:
  ["Introduction to Kafka", "Kafka Streams Tutorial"]
but could not find the following elements:
  ["Kafka Streams Tutorial"]
 within 5 seconds.
```

As expected, the deserialization failed for the second message. Consequently, the listener continued attempting to consume the same message, leading to the repeated occurrence of the error.

If we carefully analyze the failure logs, we’ll notice two suggestions:

- consider configuring an ‘ *ErrorHandlingDeserializer* ‘;
- if needed, please seek past the record to continue consumption;

**In other words, we can create a custom error handler that will handle the deserialization exception and increase the consumer offset. This will allow us to skip the invalid message and proceed with the consumption.**

### 5.1. Implementing CommonErrorHandler

To implement the *CommonErrorHandler* interface, we’ll have to override two public methods without a *default* implementation:

- *handleOne()* – called to handle a single failed record;
- *handleOtherException()* – called when an exception is thrown, but not for a particular record;

We can handle both cases using a similar approach. Let’s start by catching the exception and logging an error message:

```java
class KafkaErrorHandler implements CommonErrorHandler {

    @Override
    public void handleOne(Exception exception, ConsumerRecord<?, ?> record, Consumer<?, ?> consumer, MessageListenerContainer container) {
        handle(exception, consumer);
    }

    @Override
    public void handleOtherException(Exception exception, Consumer<?, ?> consumer, MessageListenerContainer container, boolean batchListener) {
        handle(exception, consumer);
    }

    private void handle(Exception exception, Consumer<?, ?> consumer) {
        log.error("Exception thrown", exception);
        // ...
    }
}
```

### 5.2. Kafka Consumer’s seek() and commitSync()

**We can use the *seek()* method from the *Consumer* interface to manually change the current offset position for a particular partition within a topic**. Simply put, we can use it to reprocess or skip messages as needed based on their offsets.

In our case, if the exception is an instance of *RecordDeserializationException,* we’ll call the *seek()* method with the topic partition and the next offset:

```java
void handle(Exception exception, Consumer<?, ?> consumer) {
    log.error("Exception thrown", exception);
    if (exception instanceof RecordDeserializationException ex) {
        consumer.seek(ex.topicPartition(), ex.offset() + 1L);
        consumer.commitSync();
    } else {
        log.error("Exception not handled", exception);
    }
}
```

**As we can notice, we need to call the *commitSync()* from the *Consumer* interface. This will commit the offset and ensure that the new position is acknowledged and persisted by the Kafka broker.** This step is crucial, as it updates the offset committed by the consumer group, indicating that messages up to the adjusted position have been successfully processed.

### 5.3. Updating the Configuration

**Finally, we need to add the custom error handler to our consumer configuration.** Let’s start by declaring it as a *@Bean*:

```java
@Bean
CommonErrorHandler commonErrorHandler() {
    return new KafkaErrorHandler();
}
```

After that, we’ll add the new bean to the *ConcurrentKafkaListenerContainerFactory* using its dedicated setter:

```java
@Bean
ConcurrentKafkaListenerContainerFactory<String, ArticlePublishedEvent> kafkaListenerContainerFactory(
  ConsumerFactory<String, ArticlePublishedEvent> consumerFactory,
  CommonErrorHandler commonErrorHandler
) {
    var factory = new ConcurrentKafkaListenerContainerFactory<String, ArticlePublishedEvent>();
    factory.setConsumerFactory(consumerFactory);
    factory.setCommonErrorHandler(commonErrorHandler);
    return factory;
}
```

That’s it! We can re-run the tests now and expect the listener to skip the invalid message and continue consuming messages.

## 6\. Conclusion

In this article, we discussed Spring Kafka’s *RecordDeserializationException* and we discovered that, if not handled correctly, it can block the consumer group for the given partition.

Following that, we delved into Kafka’s *CommonErrorHandler* interface and implemented it to enable our listener to handle deserialization failures while continuing to process messages. We leveraged the Consumer’s API methods, namely *seek()* and *commitSync(),* to bypass invalid messages by adjusting the consumer offset accordingly.
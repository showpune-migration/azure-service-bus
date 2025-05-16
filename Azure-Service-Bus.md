# Azure Service Bus

Contains samples, instructions, and other information that is useful for generating code that uses Azure Service Bus.

**NOTE**:
- Use Azure Managed Identity solution for Azure Service Bus connection, **DO NOT** use connection string.
- Refer to the samples in the knowledge when generate new service bus code.

## 1. Collect project context

Collect some project context:
- The spring-boot version used.
- The service bus resource names.

Before update, generate the topology info for the service bus resources.

**Note**:
- Create service bus subscription for entity type topic, **DO NOT** create service bus queue with same subscription name.
- Avoid new class name conflict with existing annotations, interfaces and classes.
- Remove the old files when new equivalent files are created.

## 2. Import Dependencies for Azure Service Bus and Azure Managed Identity

### Add the Azure Service Bus Dependencies

1. Managed Dependency:

   ```xml
   <dependency>
       <groupId>com.azure.spring</groupId>
       <artifactId>spring-cloud-azure-dependencies</artifactId>
       <version>5.22.0</version>
       <scope>import</scope>
       <type>pom</type>
   </dependency>
   ```

   **Note**:
   - If the code is using Spring Boot 2.x, set the `spring-cloud-azure-dependencies` version to `4.20.0`.
   - If the code is using Spring Boot 3.x, check the latest version of the dependency and update the version if possible.
   - Define a new property named `spring-cloud-azure.version` for `spring-cloud-azure-dependencies`.
   - This Bill of Material (BOM) should be configured in the `<dependencyManagement>` section of `pom.xml`.

2. Dependencies:

   ```xml
   <dependency>
       <groupId>com.azure.spring</groupId>
       <artifactId>spring-cloud-azure-starter</artifactId>
   </dependency>
   <dependency>
       <groupId>com.azure.spring</groupId>
       <artifactId>spring-messaging-azure-servicebus</artifactId>
   </dependency>
   ```

---

## 3. Use Azure Managed Identity for App to Connect to Azure Service Bus

### Add Service Bus Connection Settings

  ```properties
  spring.cloud.azure.credential.managed-identity-enabled=true
  spring.cloud.azure.credential.client-id=${AZURE_CLIENT_ID}
  spring.cloud.azure.servicebus.entity-type=queue
  spring.cloud.azure.servicebus.namespace=${SERVICE_BUS_NAMESPACE}
  ```

  **NOTE**:
  - If the topoloy of Service Bus is Topic + Subscriptions, set the entity-type to `topic`.

**Note**:
- We use Azure Managed Identity for Azure Service Bus, do not use connection string to init the beans.

---

## 4. Listen to Azure Service Bus Resources

### Configure `@ServiceBusListener` Parameters

Check the Azure Service Bus Topology and set the correct listener parameter:

- For Topic and Subscription:
  Set `@ServiceBusListener(destination=<topic name>, group=<subscription name>)`.

- For Queue:
  Set `@ServiceBusListener(destination=<queue name>)`.

### Add `@EnableAzureMessaging`

Add the Annotation `@EnableAzureMessaging` to one of the configuration class.

---

## 5. Send Azure Service Bus message

When constructing the service bus message, check if there is a routing key in the context. If a routing key is present, set it as the message header named ServiceBusMessageHeaders.SUBJECT. If there is no routing key, do not set the SUBJECT header.

- With Routing key

```java
Message<String> message = MessageBuilder.withPayload(strMessage)
    .setHeader(ServiceBusMessageHeaders.SUBJECT, "routing.key").build();
serviceBusTemplate.send("topic", message);
```

- Without Routing key

```java
Message<String> message = MessageBuilder.withPayload(strMessage).build();
serviceBusTemplate.send("queueName", message);
```

---

## 6. Azure Service Bus Sample Code

### Initialize a `ServiceBusAdministrationClient` Bean

This bean is required to create Service Bus resources like Topic, Subscription, Queue.

**Note**: Use Azure Managed Identity to init the Admin Client.
- The starter `spring-cloud-azure-starter` will create the `TokenCredential` bean.
- Remove unnecessary Service Bus Factory Beans and clean up any redundant configurations related to them.
- Use `AzureServiceBusProperties.getFullyQualifiedNamespace()` for `ServiceBusAdministrationClient` credential.

```java
@Bean
public ServiceBusAdministrationClient adminClient(AzureServiceBusProperties properties, TokenCredential credential) {
    return new ServiceBusAdministrationClientBuilder()
            .credential(properties.getFullyQualifiedNamespace(), credential)
            .buildClient();
}
```

### Create Azure Service Bus Topic and Subscription with Rule Options

**Note**:
- use `CorrelationRuleFilter().setLabel(routeKey)` to set `SUBJECT` in `CorrelationRuleFilter`.
- import `ResourceNotFoundException` from package `com.azure.core.exception`.

```java
@Bean
public TopicProperties topicProperties(ServiceBusAdministrationClient adminClient, String topicName) {
    try {
        return adminClient.getTopic(topicName);
    } catch (ResourceNotFoundException e) {
        return adminClient.createTopic(topicName);
    }
}

@Bean
@DependsOn("topicProperties")
public SubscriptionProperties subscription(
        ServiceBusAdministrationClient adminClient,
        String topicName,
        String subscriptionName,
        String routeKey) {

    try {
        return adminClient.getSubscription(topicName, subscriptionName);
    } catch (ResourceNotFoundException e) {
        CreateSubscriptionOptions subOptions = new CreateSubscriptionOptions();
        CorrelationRuleFilter filter = new CorrelationRuleFilter().setLabel(routeKey);
        CreateRuleOptions ruleOptions = new CreateRuleOptions().setFilter(filter);
        return adminClient.createSubscription(topicName, subscriptionName, "RouteKey", subOptions, ruleOptions);
    }
}
```

### Create Queue in Azure Service Bus

```java
@Bean
public QueueProperties queue(ServiceBusAdministrationClient adminClient, String queueName) {
    try {
        return adminClient.getQueue(queueName);
    } catch (ResourceNotFoundException e) {
        return adminClient.createQueue(queueName);
    }
}
```

### Listener with Payload

```java
public void listener(T payload) {
    // Handle payload
}
```

### Listener with Message

```java
public void listener(Message<T> message) {
    T body = message.getPayload();
    // Handle message
}
```

### Listener with Payload and Headers

```java
public void listener(T msg, @Headers Map<String, Object> headers) {
    // Handle payload and headers
}
```

### Listener with Payload, Message, and Context

```java
public void listener(T payload, Message<T> message, @Header(ServiceBusMessageHeaders.RECEIVED_MESSAGE_CONTEXT) ServiceBusReceivedMessageContext context) {
    try {
      // Handle payload, message, and context
      context.complete();
    } catch ( ... ) {
      context.abandon();
    }
}
```

### `@ServiceBusListener` on Topic and Subscription

```java
@ServiceBusListener(destination = "demoTopic", group = "demoSubscription")
public void listener(T message) {
    // Handle message
}
```

### `@ServiceBusListener` on Queue

```java
@ServiceBusListener(destination = "queueName")
public void listener(T message) {
    // Handle message
}
```

---

## 7. Key Information of Service Bus Classes, Interfaces, and Annotations

### Annotations

- `@ServiceBusListener`
  - Package: `com.azure.spring.messaging.servicebus.implementation.core.annotation`
  - Properties:
    - `id`: The unique identifier of the container managing this endpoint.
    - `containerFactory`: The bean name of the `MessageListenerContainerFactory` to use.
    - `destination`: The destination name for this listener.
    - `group`: The name for the durable group, if any.
    - `concurrency`: Override the container factory's concurrency setting.

- `@EnableAzureMessaging`
  - Package: `com.azure.spring.messaging.implementation.annotation`

### Interfaces and Classes

- `Message<T>`
  - Package: `org.springframework.messaging`
  - Methods:
    - `getPayload()`: Returns the message payload.
    - `getHeaders()`: Returns the message headers.

- `ServiceBusTemplate`
  - Package: `com.azure.spring.messaging.servicebus.core`
  - Methods:
    - `setMessageConverter(AzureMessageConverter<ServiceBusReceivedMessage, ServiceBusMessage> messageConverter)`
    - `send(String destination, Message<T> message)`
    - `sendAsync(String destination, Message<T> message)`

- `MessageBuilder<T>`
  - Package: `org.springframework.messaging.support`
  - Description: `A builder for creating GenericMessage, where T is the message payload type`
  - Methods:
    - `setHeader(String headerName, @Nullable Object headerValue)`
      - Description: Set the value for the given header name. If the provided value is null, the header will be removed.
      - Parameters:
        - headerName: the name of the header
        - headerValue: the value of the header
      - Returns: MessageBuilder<T>
    - `withPayload(<T> payload)`
      - Description: Create a new builder for a message with the given payload.
      - Parameters:
        - payload: the payload
      - Returns: MessageBuilder<T>
    - `build()`
      - Description: build a message
      - Returns: Message<T>

### Package Classes

- Package: `com.azure.messaging.servicebus.administration`
  - Classes:
    - `ServiceBusAdministrationClient`
    - `ServiceBusAdministrationClientBuilder`

- Package: `com.azure.messaging.servicebus.administration.models`
  - Classes:
    - `CorrelationRuleFilter`
    - `CreateRuleOptions`
    - `CreateQueueOptions`
    - `CreateSubscriptionOptions`
    - `CreateTopicOptions`
    - `QueueProperties`
    - `SqlRuleFilter`
    - `SubscriptionProperties`
    - `TopicProperties`

- Package: `com.azure.spring.cloud.autoconfigure.implementation.servicebus.properties`
  - Classes:
    - `AzureServiceBusProperties`

- Package: `com.azure.core.exception`
  - Classes:
    - `ResourceNotFoundException`: Represents a common Azure SDK exception thrown on attempts to access an Azure Resource that does not exist.

- Package: `com.azure.core.credential`
  - Classes:
    - `TokenCredential`

### Constants

- `MessageHeaders`
  - Class: `org.springframework.messaging`
  - Constants:
    - `CONTENT_TYPE`
    - `REPLY_CHANNEL`

- `ServiceBusMessageHeaders`
  - Class: `com.azure.spring.messaging.servicebus.support`
  - Constants:
    - `SUBJECT`
    - `SCHEDULED_ENQUEUE_TIME`
    - `CORRELATION_ID`
    - `REPLY_TO_SESSION_ID`
    - `SESSION_ID`
    - `TO`

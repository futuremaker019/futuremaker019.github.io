---
layout: post
tags: [Kafka, Infra, blog]
toc: true
---

## Kafka를 이용한 책임분리 및 Transactional Outbox Pattern 구현


### Transactional Outbox pattern 이란

- Outbox는 `보낸 편지함`  을 뜻하며, 발생된 message 를 `Message Broker`로 발행되었는지를 확인하여 신뢰성을 보장해주는 패턴이다.
- 이러한 패턴은 왜 필요한가
    - 트랜잭션이 커밋된 이 후 카프카 등의 `Message Broker`에서 메시지 발행이 되지 않았다면 반드시 처리되야 하는 메시지는 소실 될 것이다.
    - 또한 DB 트랜잭션은 데이터 베이스 차원의 원자성을 보장하지만 비동기로 이루어지는 `Event Driven Architecture` 시스템에서는 메시지 발행의 유무는 `Consumer`를 통해 메시지를 직접 확인하지 않는다면 메시지 유실 유무를 확인할 수 없다.
- 위와 같은 이유로 도메인 로직의 성공 시, 발행되는 메시지를 `outbox table` 이라는 별도의 테이블에 저장하여 함께 커밋한다. 미발행된 메시지를 outbox table에서 확인하여 배치 시스템등으로 재발행을 유도한다.

### 프로젝트 아키텍처

```
├── application/
│   └── event/
│       └── payment/
│           └── PaymentEventListener
├── domain/
│   ├── common/
│   │   ├── dataplatform/
│   │   │   └── DataPlatformClient          (interface)
│   │   ├── notification/
│   │   │   ├── Notification
│   │   │   └── NotificationSender          (interface)
│   │   └── outbox/
│   │       ├── Outbox
│   │       ├── OutboxMessageWriter         (interface)
│   │       └── OutboxRepository            (interface)
│   └── payment/
│       ├── event/
│       │   ├── PaymentEvent
│       │   └── PaymentEventPublisher       (interface)
│       └── message/
│           ├── PaymentMessage
│           └── PaymentMessageSender        (interface)
├── infrastructures/
│   ├── kafka/
│   │   ├── KafkaMessage
│   │   ├── payment/
│   │   │   └── PaymentKafkaMessageSender   (PaymentMessageSender 구현체)
│   │   └── notification/
│   │       └── SlackNotificationSender     (NotificationSener 구현체)
│   └── spring/
│       └── payment/
│           └── PaymentSpringEventPublisher (PaymentEventPublisher 구현체)
├── interfaces/
│   └── consumer/
│       └── payment/
│           └── PaymentMessageConsumer
└── support/
    └── KafkaConfig
```

- Layered Architecture 를 구성하여 각 계층별로 Kafka의 메시지 pulisher와 consumer를 적절히 위치 시켰으며, 도메인에 `PaymentMessageSender` interface를 생성하여 DIP(Dependency Inversion Principle)을 만족할 수 있는 형태로 구성하였다.
- 또한 `PaymentEventListener` 에서는 Spring 에서 제공하는 Application Event를 이용하여 도메인 로직이 commit 된 후, message 를 발행할 수 있도록 하여 트랜잭션이 완료되지 않거나 롤백되는 경우 이벤트가 실행되지 않으므로 데이터의 일관성을 보장할 수 있도록 하였다.

<br>

- PaymentEventListener

```java
@Component
@RequiredArgsConstructor
public class PaymentEventListener {

    private final OutboxRepository outboxRepository;
    private final PaymentMessageSender paymentMessageSender;

    @TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
    public void saveOutbox(PaymentEvent.Send send) {
        OutboxStatement.Save outbox =
                OutboxStatement.Save.of(send.outboxId().toString(), send.paymentId().toString());
        outboxRepository.save(outbox);
    }

    @Async
    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    public void sendMessage(PaymentEvent.Send send) {
        PaymentMessage.Send paymentMessage = PaymentMessage.Send.of(send.paymentId());
        KafkaMessage<PaymentMessage.Send> message =
                KafkaMessage.of(send.outboxId().toString(), paymentMessage);
        paymentMessageSender.send(message);
    }

}
```
- saveOutbox
    - 도메인 로직의 트랜잭션이 이루어질때, 전송될 메시지를 Outbox table에 저장하여 Outbox 패턴을 구성하기 위한 데이터를 적재한다.
- sendMessage
    - 메시지를 발행하여 `Topic`에 전달한다.

<br>

- PaymentMessageConsumer


```java
@Slf4j
@Component
@RequiredArgsConstructor
public class PaymentMessageConsumer {

    private final DataPlatformClient dataPlatformClient;
    private final NotificationSender notificationSender;

    private final OutboxMessageWriter outboxMessageWriter;

    private final ObjectMapper objectMapper;

    /**
     * outbox 검증 로직
     * */
    @KafkaListener(topics = "${kafka.topic.name}", groupId = "group1")
    public void complete(KafkaMessage<?> message) {
        outboxMessageWriter.complete(message.getId());
    }

    /**
     * 데이터 플랫폼 호출 (외부 API)
     * */
    @KafkaListener(topics = "${kafka.topic.name}", groupId = "group2")
    public void sendData(KafkaMessage<?> message) {
        LinkedHashMap<?, ?> map = (LinkedHashMap<?, ?>) message.getMessage();
        PaymentMessage.Get get = objectMapper.convertValue(map, PaymentMessage.Get.class);
        callDataPlatform(get);
    }

    /**
     * slack 알림 연동
     * */
    @KafkaListener(topics = "${kafka.topic.name}", groupId = "group3")
    public void sendNotification(KafkaMessage<?> message) {
        LinkedHashMap<?, ?> map = (LinkedHashMap<?, ?>) message.getMessage();
        PaymentMessage.Get get = objectMapper.convertValue(map, PaymentMessage.Get.class);
        sendNotification(get);
    }
}
```
- Topic에 전달된 메시지를 Consumer 를 이용하여 메시지가 성공적으로 생성된 것을 확인했다면 outbox table에 저장된 데이터의 상태값을 PUBLISHED 로 변경한다.

```java
@Component
@RequiredArgsConstructor
public class SchedulerManager {

    @Scheduled(cron = "5 * * * * *", zone = "Asia/Seoul")
    public void resendMessage() {
        outboxMessageWriter.resend();
    }
}

@Component
@RequiredArgsConstructor
public class OutboxMessageWriterImpl implements OutboxMessageWriter {

    @Override
    @Transactional
    public void complete(String outboxId) {
        outboxRepository.updateStatus(outboxId, OutboxType.PUBLISHED);
    }
    
    @Override
    public void resendPaymentMessage() {
        List<OutboxEntity> outboxes = outboxRepository.getOutboxByOutboxType(OutboxType.INIT);
        if (!outboxes.isEmpty()) {
            outboxes.forEach(outbox -> {
                if (outbox.getCreatedAt().plusMinutes(5).isBefore(LocalDateTime.now())) {
                    PaymentMessage.Send paymentMessage =
                            objectMapper.convertValue(outbox.getMessage(), PaymentMessage.Send.class);
                    KafkaMessage<PaymentMessage.Send> message = KafkaMessage.of(outbox.getId(), paymentMessage);
                    paymentMessageSender.send(message);
                }
            });
        }
    }
}
```
- 상태값이 PUBLISHED 로 변경되지 못한 outbox table의 데이터를 유실된 메시지로 판단하여 스케줄러를 통해 일정시간의 간격으로 다시 발행하도록 한다

## 결론

- Message Broker를 적용하여 메시지를 발행하고 소비하는 방식의 구성으로 시스템의 느슨한 결합을 유도하여, 구성의 변경 시 다른 로직에 영향을 미치지 않게 만들 수 있다.
- 이로 인하여 유연성 및 확장성이 증가한다.
- 또한 Transactional Outbox Pattern을 적용하여 메시지 발행에 대한 신뢰성 및 안정성을 유지할 수 있어 메시지 유실을 방지하여 데이터의 일관성을 유지할 수 있다.

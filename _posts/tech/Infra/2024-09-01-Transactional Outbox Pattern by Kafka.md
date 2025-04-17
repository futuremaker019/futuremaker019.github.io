---
title: Transactional Outbox Pattern by Kafka
categories: [Tech, Infra]
tags: [Kafka, Infra, blog]
toc: true
---

## 살펴보기

메시지 브로커를 활용한 마이크로서비스 아키텍처에서는, 하나의 비즈니스 트랜잭션이 여러 서비스에 걸쳐 분산되어 처리되는 경우가 많습니다. 이때 각 서비스는 자신의 로컬 데이터베이스에 데이터를 저장하는 동시에, 다른 서비스로의 이벤트 전파를 위해 메시지를 발행해야 합니다.

하지만 이러한 구조에서 자주 발생하는 문제는 다음과 같습니다

- DB 저장은 성공했지만 메시지 발행이 실패
- 메시지는 발행됐지만 DB 트랜잭션은 롤백됨

이처럼 DB 상태와 메시지 전파 상태가 불일치하게 되면, 전체 시스템의 데이터 정합성이 깨질 수 있습니다. 이 문제를 해결하기 위해 등장한 개념이 바로 Transactional Messaging입니다.

### Transactional Messaging

Transactional Messaging은 **데이터 저장(DB 트랜잭션)**과 **메시지 브로커 전송(Event 발행)**을 하나의 논리적 트랜잭션처럼 묶어 처리하여, 시스템 간 **데이터 정합성(consistency)**과 **신뢰성(reliability)**을 확보하는 아키텍처 패턴입니다.

Transactional Messaging은 마이크로서비스 간 느슨한 결합을 유지하면서도, 이벤트 흐름과 상태 전파가 정확하게 일치하도록 만들어주는 핵심 전략입니다.

## Transactional Messaging 구현 전략

> 대표적인 방식인 Transactional Outbox Pattern을 살펴보고자 합니다.

### Transactional Outbox pattern

- Outbox는 `보낸 편지함`  을 뜻하며, 발생된 message 를 `Message Broker`로 발행되었는지를 확인하여 신뢰성을 보장해주는 패턴입니다.

🔍 이러한 패턴은 왜 필요한가
- 트랜잭션이 커밋된 이 후 카프카 등의 `Message Broker`에서 메시지 발행이 되지 않았다면 반드시 처리되야 하는 메시지는 소실 될 것입니다.
- 또한 DB 트랜잭션은 데이터 베이스 차원의 원자성을 보장하지만 비동기로 이루어지는 `Event Driven Architecture` 시스템에서는 메시지 발행의 유무는 `Consumer`를 통해 메시지를 직접 확인하지 않는다면 메시지 유실 유무를 확인할 수 없습니다.

위와 같은 이유로 도메인 로직의 성공 시, 발행되는 메시지를 `outbox table` 이라는 별도의 테이블에 저장하여 함께 커밋합니다. 미발행된 메시지를 outbox table에서 확인하여 배치 시스템등으로 재발행을 유도합니다.

### 그림으로 보는 Outbox pattern의 흐름

![Desktop View](/assets/images/infra/outbox.png){: width="972" height="589" }

①	도메인 데이터와 메시지를 Outbox에 함께 저장 (단일 트랜잭션) <br>
②	Outbox 테이블에서 메시지를 Kafka로 발행 <br>
③	Kafka 발행 성공 시 메시지 상태 업데이트 <br>
④ INIT 상태의 메시지를 주기적으로 조회하여 Kafka에 재전송하여, 전송 실패나 누락된 이벤트를 보완<br>

### 프로젝트 구성

#### 프로젝트 아키텍처

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

- Layered Architecture 를 구성하여 각 계층별로 Kafka의 메시지 pulisher와 consumer를 적절히 위치 시켰으며, 도메인에 `PaymentMessageSender` interface를 생성하여 DIP(Dependency Inversion Principle)을 만족할 수 있는 형태로 구성하였습니다.
- 또한 `PaymentEventListener` 에서는 Spring 에서 제공하는 Application Event를 이용하여 도메인 로직이 commit 된 후, message 를 발행할 수 있도록 하여 트랜잭션이 완료되지 않거나 롤백되는 경우 이벤트가 실행되지 않으므로 데이터의 일관성을 보장할 수 있도록 하였습니다.



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
- Topic에 전달된 메시지를 Consumer 를 이용하여 메시지가 성공적으로 생성된 것을 확인했다면 outbox table에 저장된 데이터의 상태값을 PUBLISHED 로 변경합니다.

```java
@Component
@RequiredArgsConstructor
public class OutboxMessageWriterImpl implements OutboxMessageWriter {

    @Override
    @Transactional
    public void complete(String outboxId) {
        outboxRepository.updateStatus(outboxId, OutboxType.PUBLISHED);
    }
    

    /**
       재발행 로직
        상태값이 `INIT`인 상태의 outbox 데이터를 찾아 메시지를 재발행함
     */
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

@Component
@RequiredArgsConstructor
public class SchedulerManager {

    /** 
       스케줄러를 이용한 재발행을 유도함
     */
    @Scheduled(cron = "5 * * * * *", zone = "Asia/Seoul")
    public void resendMessage() {
        outboxMessageWriter.resendPaymentMessage();
    }
}
```
- 상태값이 `PUBLISHED` 로 변경되지 못한 `outbox table`의 데이터를 유실된 메시지로 판단하여 스케줄러 or batch 시스템을 통해 일정시간의 간격으로 다시 발행하도록 합니다

## 결론

- 메시지 브로커를 활용한 이벤트 기반 구조는 서비스 간 느슨한 결합을 가능하게 하여, 시스템 구성 변경 시 다른 서비스에 영향을 최소화할 수 있습니다.
- 이를 통해 유연성과 확장성이 크게 향상되며, 복잡한 통합 시나리오에서도 안정적인 연동이 가능합니다.
- 특히, Transactional Outbox Pattern을 적용하면 메시지 발행의 신뢰성과 안정성을 확보할 수 있어, 이벤트 유실 없이 데이터 일관성을 유지할 수 있습니다.

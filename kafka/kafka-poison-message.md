# 역직렬화 안 되는 Kafka 메시지 하나가 파티션 전체를 멈춘다

> [ticketing-simulator](https://github.com/jun0x2dev/ticketing-simulator)에서 겪은 일이다.

## 문제

`entry.requested` 토픽에 JSON이 아닌 메시지(`not-json`) 하나를 넣고, 바로 뒤에 정상 메시지를 넣었다.

- 30초 동안 `RecordDeserializationException`이 약 66만 번 반복됐다.
- 뒤의 정상 메시지는 끝내 처리되지 않았다.
- 재시도 3회 후 DLT로 보내는 `DefaultErrorHandler`를 설정해 뒀는데도 DLT에는 아무것도 쌓이지 않았다.

실서비스였다면 그 파티션으로 들어온 요청이 전부 멈추고, 로그와 CPU가 폭주했을 것이다.

## 원인

값 역직렬화기로 `JsonDeserializer`를 그대로 썼다.

역직렬화는 리스너가 아니라 `KafkaConsumer.poll()` 안에서 일어난다. 그래서 실패하면 레코드가 리스너나 에러 핸들러에 전달되기 전에 예외가 난다.

```
poll() ── 역직렬화 실패 ──> 예외 (에러 핸들러는 이 레코드를 본 적 없음)
  ↑                              │
  └──── 오프셋 그대로, 같은 메시지 다시 읽음 ─┘
```

오프셋이 넘어가지 않으니 다음 poll에서 같은 메시지를 다시 읽고 다시 실패한다. 재시도 횟수도 DLT도 끼어들 틈이 없다.

## 해결

값 역직렬화기를 `ErrorHandlingDeserializer`로 감싸고, 실제 변환은 `JsonDeserializer`에 맡긴다.

```properties
spring.kafka.consumer.value-deserializer=org.springframework.kafka.support.serializer.ErrorHandlingDeserializer
spring.kafka.consumer.properties.spring.deserializer.value.delegate.class=org.springframework.kafka.support.serializer.JsonDeserializer
```

- 역직렬화에 실패해도 예외를 던지지 않는다. 대신 실패를 헤더에 담아 레코드를 넘긴다. 그러면 에러 핸들러가 이 레코드를 받아 처리할 수 있다.
- `DeserializationException`은 재시도해도 결과가 같으므로 재시도 없이 바로 DLT로 간다(`DefaultErrorHandler`의 기본 동작).

같이 챙긴 것이 두 가지 있다.

**1. 깨진 메시지는 원본 바이트 그대로 DLT에 보관한다.**
역직렬화에 실패한 레코드는 값이 `null`이고, 원본 바이트는 예외와 함께 헤더(`springDeserializerExceptionValue`)에 담긴다. `DeadLetterPublishingRecoverer`는 DLT로 보낼 때 이 헤더에서 원본 `byte[]`를 꺼내 값으로 되돌린다. 이걸 JSON 직렬화기로 보내면 base64 문자열이 되어 원본을 알아보기 어렵다. 그래서 `byte[]` 전용 템플릿을 따로 줬다.

```java
Map<Class<?>, KafkaOperations<?, ?>> templates = new LinkedHashMap<>();
templates.put(byte[].class, bytesTemplate);   // ByteArraySerializer
templates.put(Object.class, kafkaTemplate);   // JsonSerializer
DeadLetterPublishingRecoverer recoverer = new DeadLetterPublishingRecoverer(templates, (record, ex) -> ...);
```

**2. DLT 처리에 실패한 메시지를 `*.DLT.DLT`로 보내지 않는다.**
DLT에서 다시 실패한 메시지를 또 DLT로 보내면 `.DLT.DLT` 토픽이 생기고, 같은 일이 반복될 수 있다. 원본 토픽이 이미 DLT면 목적지를 `null`로 돌려 error 로그만 남기고 건너뛴다.

```java
(record, ex) -> {
    if (record.topic().endsWith(DLT_SUFFIX)) {   // ".DLT"
        log.error("DLT 메시지 처리 실패, 건너뜀 - topic: {}, partition: {}, offset: {}",
                record.topic(), record.partition(), record.offset(), ex);
        return null;
    }
    return new TopicPartition(record.topic() + DLT_SUFFIX, record.partition());
}
```

## 느낀 점

- 재시도와 DLT를 설정해 뒀으니 괜찮을 줄 알았다. 그런데 에러 핸들러는 리스너까지 도달한 레코드만 처리한다. 그 앞의 역직렬화에서 실패하면 소용이 없다.
- 정상 메시지만 넣는 테스트로는 안 보이는 문제다. 깨진 메시지를 일부러 넣는 테스트를 하나 두는 게 좋다.

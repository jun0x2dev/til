# Kafka 리스너에서 InterruptedException을 삼키면 처리하지 않은 메시지가 커밋된다

> [ticketing-simulator](https://github.com/jun0x2dev/ticketing-simulator)에서 겪은 일이다.

## 문제

DLT Consumer는 메시지가 도착하고 5초가 지난 뒤 재처리한다. 일시적 장애라면 바로 다시 보내도 같은 이유로 실패하기 때문이다.

```java
@KafkaListener(topics = KafkaTopicConfig.ENTRY_REQUESTED_DLT, groupId = "${spring.kafka.consumer.group-id}-dlt")
public void consume(ConsumerRecord<String, EntryRequestedEvent> record) {
    // (값 없는 메시지를 건너뛰는 부분은 생략)
    long waitMillis = record.timestamp() + redriveDelayMillis - System.currentTimeMillis();   // 기본 5초
    try {
        if (waitMillis > 0) {
            Thread.sleep(waitMillis);
        }
        entryService.handleDeadLetter(record.value());
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();   // "정석"대로 플래그만 다시 세움
    } catch (RuntimeException e) {
        log.error("DLT 처리 실패", e);
    }
}
```

대기 중에 리스너 스레드가 인터럽트되면 `sleep`이 끊기고, 메서드는 재처리를 하지 않은 채 정상 반환한다. 그러면 처리하지 않은 메시지의 오프셋이 커밋된다. 재투입도 최종 실패 처리도 되지 않으므로, 사용자는 실패 안내를 받지 못한다. 토큰 TTL(10분) 동안 순번이 없는 대기 화면에 머문다.

## 원인

`InterruptedException`을 잡아 플래그를 세우고 메서드는 정상 반환했다.

Spring Kafka는 리스너가 정상 반환하면 그 레코드를 처리한 것으로 보고 오프셋을 커밋한다. 리스너 쪽에서는 플래그를 다시 세운 게 맞는 처리였지만, 컨테이너는 이걸 처리 완료와 구분하지 못한다.

## 해결

`sleep`을 try 밖으로 빼고, `InterruptedException`은 그대로 던진다.

```java
@KafkaListener(topics = KafkaTopicConfig.ENTRY_REQUESTED_DLT, groupId = "${spring.kafka.consumer.group-id}-dlt")
public void consume(ConsumerRecord<String, EntryRequestedEvent> record) throws InterruptedException {
    // (값 없는 메시지를 건너뛰는 부분은 생략)
    long waitMillis = record.timestamp() + redriveDelayMillis - System.currentTimeMillis();
    if (waitMillis > 0) {
        Thread.sleep(waitMillis);            // 인터럽트되면 예외로 끝남 → 커밋 안 됨
    }
    try {
        entryService.handleDeadLetter(record.value());
    } catch (RuntimeException e) {
        log.error("DLT 입장 요청 처리 실패 - requestId: {}", record.value().requestId(), e);
    }
}
```

리스너가 예외로 끝나면 `DefaultErrorHandler`가 재시도를 준비한다. 이때 컨테이너가 멈추는 중이면 재시도 대기(`stoppableSleep`)가 바로 끝난다. 레코드는 처리하지 않은 것으로 남고 그 위치로 되감긴다. 오프셋이 커밋되지 않으니, 재시작한 뒤 같은 메시지를 다시 받는다.

## 그런데 정상 종료에서 정말 인터럽트될까?

처음에는 "배포로 앱이 종료되면 컨테이너가 리스너 스레드를 인터럽트해 메시지가 사라진다"고 생각했다. 그런데 spring-kafka 4.1.2의 컨테이너 코드를 바이트코드(`javap -c`)로 확인해 보니 그렇지 않았다.

- `stop()`이 하는 일은 세 가지다. running 플래그를 내리고, Consumer를 깨우고(`wakeIfNecessaryForStop`), `shutdownTimeout`(기본 10초)까지 기다린다.
- 코드 안의 `Thread.interrupt()` 호출은 모두 `InterruptedException`을 잡은 뒤 자기 스레드의 플래그를 다시 세우는 용도였다. 다른 스레드(리스너 스레드)를 인터럽트하는 코드는 없었다.

즉 정상 종료에서는 최대 5초인 대기를 끝까지 마치고, 메시지를 처리한 뒤 커밋한다. 이 수정은 실제로 재현된 버그의 해결이 아니다. 리스너 스레드를 인터럽트하는 다른 주체(스레드 풀 강제 종료 등)가 생길 때를 대비한 방어 코드다.

## 느낀 점

- "`InterruptedException`은 삼키지 말고 플래그를 다시 세워라"라고 배웠는데, 반환 여부로 성공을 판단하는 콜백 안에서는 그것만으로 부족했다. 플래그만 세우고 반환하면 성공으로 처리된다.
- "종료하면 인터럽트된다"는 시나리오를 확인하지 않고 버그로 기록했다가 나중에 바로잡았다. 프레임워크가 이렇게 동작할 거라고 가정했다면 코드로 확인해 봐야 한다. 소스가 없으면 바이트코드로라도 확인할 수 있다.
- 테스트는 간단하다. 대기 중인 리스너 스레드를 인터럽트하고, 메서드가 예외로 끝나는지와 재처리가 호출되지 않았는지를 확인하면 된다. 다만 이 테스트는 "인터럽트되면 어떻게 되는가"를 검증할 뿐이다. "종료 때 인터럽트된다"는 가정까지 검증해 주지는 않는다.

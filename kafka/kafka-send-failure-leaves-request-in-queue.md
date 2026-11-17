# Kafka 발행에 실패한 입장 요청이 대기 줄에 그대로 남았다

> [ticketing-simulator](https://github.com/jun0x2dev/ticketing-simulator)에서 겪은 일이다.

## 문제

입장 요청은 이렇게 흘러간다.

```
POST /entries → Redis 대기 줄(ZSET)에 세움 → entry.requested 발행 → 202 응답
Queue Worker가 entry.requested를 소비 → 입장 토큰 발급 → 줄에서 뺌
```

브로커 장애 중에 입장을 요청하면 API는 202를 응답하고 대기 화면이 뜬다. 그런데 순번이 줄지 않은 채 멈추고, "다시 줄 서기" 버튼도 나타나지 않는다. 대기 줄 키의 TTL(1시간)이 지나야 줄에서 사라진다.

장애가 실제로 난 것은 아니다. outbox를 적용하면서 "이 토픽은 유실돼도 회복된다"는 근거를 코드로 확인하다가 이 경로를 발견했다. DLT 재투입 발행이 실패할 때도 같은 일이 생겼다.

## 원인

발행은 비동기였고, 실패는 로그로만 남겼다.

```java
public EntryRequestResponse requestEntry(Long eventId) {
    ...
    // Kafka 발행보다 먼저 줄에 세워야, Consumer가 먼저 처리해 버려 줄에 영원히 남는 요청이 생기지 않는다.
    waitingLineManager.join(eventId, event.requestId());
    entryRequestProducer.send(event);
    return new EntryRequestResponse(event.requestId());
}

public void send(EntryRequestedEvent event) {
    kafkaTemplate.send(KafkaTopicConfig.ENTRY_REQUESTED, key, event)
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("입장 요청 이벤트 발행 실패 - requestId: {}", event.requestId(), ex);
                }
                ...
            });
}
```

- 줄에 먼저 세우는 순서는 의도한 것이다. 발행을 먼저 하면 Consumer가 먼저 처리해 버린 뒤 줄에 서서, 영원히 빠지지 않는 요청이 생긴다.
- 그 대신 발행이 실패하면 줄에는 있지만 Kafka에는 없는 요청이 생긴다. Queue Worker가 이 요청을 꺼낼 일이 없으니 순번이 줄지 않는다.
- 대기 화면의 "다시 줄 서기"는 상태가 `FAILED`일 때만 나오는데, 이 경로에서는 아무도 실패로 표시하지 않았다.

## 첫 번째 해결: 브로커 확인을 기다린다

처음에는 브로커 확인을 기다린 뒤 202를 주고, 실패하면 503을 주는 동기 방식으로 고쳤다.

이 PC에서 동시 50건을 보내 보니 p50이 15ms(브로커를 쓰지 않는 기준선)에서 220ms로 늘었다. 확인을 기다리는 10~20ms 동안 요청 스레드가 묶여 줄이 밀린 것이다. 입장 요청은 오픈 순간의 트래픽을 싸게 받아내야 하는 구간이라 이 방식은 버렸다.

## 두 번째 해결: 응답은 비동기, 실패만 콜백으로 처리한다

Producer가 발행 결과 future를 돌려주고, 서비스가 실패했을 때만 정리한다.

```java
public CompletableFuture<?> send(EntryRequestedEvent event) {
    return kafkaTemplate.send(KafkaTopicConfig.ENTRY_REQUESTED, key, event)
            .whenComplete(...);   // 성공 로그만 남긴다
}
```

```java
private void publish(EntryRequestedEvent event) {
    CompletableFuture<?> result;
    try {
        result = entryRequestProducer.send(event);
    } catch (RuntimeException e) {
        result = CompletableFuture.failedFuture(e);    // send() 호출 자체가 던지는 경우
    }
    result.whenComplete((ignored, ex) -> {
        if (ex != null) {
            failRequest(event, ex);
        }
    });
}

private void failRequest(EntryRequestedEvent event, Throwable cause) {
    waitingLineManager.leave(event.eventId(), event.requestId());
    entryTokenManager.markFailed(event.requestId());   // entry:failed:{requestId}
    log.error("입장 요청 최종 실패 - requestId: {}, ...", event.requestId(), cause);
}
```

- 실패하면 요청을 줄에서 빼고 실패로 표시한다. 상태 조회가 `FAILED`를 돌려주므로, 1초마다 조회하는 대기 화면이 바로 "다시 줄 서기"를 안내한다.
- `KafkaTemplate.send()`는 future로 실패를 알리기도 하지만, 메타데이터를 못 가져오는 경우처럼 호출 자체에서 예외를 던질 수도 있다. 두 경우를 같은 경로로 모았다.
- DLT 재투입도 같은 `publish()`를 타게 했다.
- 대기 화면은 입장 요청 API 자체가 실패했을 때(네트워크 오류 등)도 "다시 줄 서기" 버튼을 보여 주게 했다. 전에는 오류 문구만 있고 다시 시도할 방법이 없었다.

## 남는 한계

- 브로커가 처음부터 없으면 `max.block.ms`(이 프로젝트는 5초로 설정) 뒤에 실패로 표시된다.
- 브로커가 메시지를 받는 도중에 끊기면, Producer가 `delivery.timeout.ms`(기본 120초)까지 재시도한 뒤에야 실패를 알린다. 그동안 사용자는 대기 화면에 머문다.
- 실패로 표시된 뒤 메시지가 늦게 도착해 토큰이 발급될 수 있다. 상태 조회가 토큰을 먼저 확인하므로 이 경우는 `GRANTED`로 정상 입장한다.

## 느낀 점

- 발행 실패는 `whenComplete`의 에러 로그로만 남고 있었다. 실패했을 때 사용자 화면이 어떻게 되는지까지 따라가 보고서야 멈춘 화면이 보였다.
- 처음 고친 동기 방식은 재 보니 응답 시간이 크게 늘었다. 고치는 방법도 같은 조건에서 재 보고 골랐다.
- Redis 작업과 Kafka 발행은 원자적으로 묶을 수 없다. 그래서 묶는 대신 실패했을 때 되돌리는 경로를 만들었다.

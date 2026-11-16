# 커밋 뒤에 바로 발행하면 그 사이에 좌석 이벤트가 사라질 수 있었다

> [ticketing-simulator](https://github.com/jun0x2dev/ticketing-simulator)에서 겪은 일이다.

## 문제

결제가 끝나면 좌석을 확정하고, `seat.confirmed` 이벤트를 Kafka로 보낸다. 통계 Consumer가 이 이벤트로 회차별 판매 수를 센다.

처음 구조에서는 DB 트랜잭션을 커밋한 다음에 이벤트를 보냈다.

```java
Payment payment = transactionTemplate.execute(status -> {
    seatService.confirmSeat(seatId, entryToken);
    return paymentRepository.save(Payment.create(seatId, PaymentStatus.SUCCESS));
});
seatEventProducer.sendConfirmed(SeatConfirmedEvent.of(seat.eventId(), seatId, payment.getId()));
```

커밋과 발행 사이에 서버가 죽거나 브로커가 응답하지 않으면 이벤트가 사라진다. 좌석은 확정됐는데 통계에서는 빠지고, 다시 보낼 근거도 어디에도 남지 않는다.

## 고려한 방법

- Kafka 트랜잭션과 DB 트랜잭션 동기화: DB와 Kafka를 원자적으로 묶지 못한다(최선 노력 1PC). DB 커밋 후 Kafka 커밋이 실패하는 틈이 남고, Consumer도 `read_committed`로 바꿔야 한다.
- CDC(Debezium): binlog를 읽어 발행한다. Kafka Connect와 Debezium 운영이 따로 필요해서, 모놀리식 MVP 범위에 비해 과했다.
- Transactional Outbox + 폴링 발행기: 이벤트를 같은 트랜잭션 안에서 DB 테이블에 기록하고, 스케줄러가 읽어서 발행한다. 테이블 하나와 스케줄러 하나로 "DB에 기록되면 언젠가 발행된다"를 만들 수 있다.

결제는 이미 게이트웨이 호출 뒤에 짧은 트랜잭션(`TransactionTemplate`)이 있었다. TTL 복구도 트랜잭션 안이다. 그래서 행 하나만 더 쓰면 되는 outbox를 골랐다.

## 해결

### 같은 트랜잭션에 기록

좌석 확정, 결제 기록, outbox 행을 한 트랜잭션에 쓴다. 커밋되면 이벤트도 발행되고, 롤백되면 셋 다 남지 않는다.

```java
Payment payment = transactionTemplate.execute(status -> {
    seatService.confirmSeat(seatId, entryToken);
    Payment saved = paymentRepository.save(Payment.create(seatId, PaymentStatus.SUCCESS));
    outboxService.append(KafkaTopicConfig.SEAT_CONFIRMED, String.valueOf(seat.eventId()),
            SeatConfirmedEvent.of(seat.eventId(), seatId, saved.getId()));
    return saved;
});
```

`append`는 `Propagation.MANDATORY`로 걸었다. 비즈니스 변경과 같은 트랜잭션이어야 의미가 있으므로, 트랜잭션 없이 호출되면 새로 만들지 않고 실패시킨다.

```java
@Transactional(propagation = Propagation.MANDATORY)
public void append(String topic, String key, Object payload) {
    outboxEventRepository.save(OutboxEvent.create(
            topic, key, payload.getClass().getName(), jsonMapper.writeValueAsString(payload)));
}
```

### 발행기는 SKIP LOCKED로 가져간다

발행기는 1초마다 발행 대기 행을 100건씩 가져와 보낸다. 서버가 여러 대여도 같은 행을 동시에 보내지 않도록 `FOR UPDATE SKIP LOCKED`를 썼다. 다른 인스턴스가 잠근 행은 기다리지 않고 건너뛴다.

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
// 잠금 대기 시간 -2는 Hibernate에서 SKIP LOCKED를 뜻한다.
@QueryHints(@QueryHint(name = "jakarta.persistence.lock.timeout", value = "-2"))
@Query("select o from OutboxEvent o where o.publishedAt is null order by o.id")
List<OutboxEvent> findPendingForUpdate(Pageable pageable);
```

다른 선택지는 리더 선출(ShedLock 등)로 한 인스턴스만 발행하거나, 잠금 없이 모두 발행하고 중복은 Consumer에 맡기는 방법이었다. 리더 선출은 라이브러리가 하나 늘고 한 대만 일하게 된다. 잠금이 없으면 인스턴스 수만큼 중복 발행된다. `SKIP LOCKED`는 MySQL 8 기본 기능이라 의존성이 늘지 않는다. 인스턴스가 늘면 처리량도 늘고, 한 인스턴스가 죽어 잠금이 풀리면 다른 인스턴스가 이어받는다.

### 브로커가 확인한 건만 완료로 표시

한 건씩 응답을 기다리지 않고 모두 보낸 뒤 결과를 모은다. 브로커가 확인한 행만 `publishedAt`을 채우고, 실패한 행은 다음 주기에 다시 시도한다. 발행기는 행 잠금과 DB 커넥션을 잡은 채 기다리므로, 전체 대기 시간을 5초로 제한했다.

```java
List<CompletableFuture<?>> results = pending.stream().<CompletableFuture<?>>map(this::send).toList();

long deadline = System.nanoTime() + sendTimeout.toNanos();
for (int i = 0; i < pending.size(); i++) {
    try {
        results.get(i).get(Math.max(0, deadline - System.nanoTime()), TimeUnit.NANOSECONDS);
        pending.get(i).markPublished();
    } catch (ExecutionException | TimeoutException e) {
        pending.get(i).recordFailure();
    }
}
```

발행 완료 행은 24시간 뒤 1000건씩 지우고, 발행 대기 행은 기간과 관계없이 남긴다.

## 감수한 점

- 발행은 "적어도 한 번"이라 중복이 생길 수 있다. 브로커가 받았지만 대기 시간 안에 확인이 오지 않은 건이다. 통계 Consumer가 이미 `paymentId`와 `seatId + occurredAt`으로 멱등 집계하고 있었다. outbox는 저장해 둔 payload를 그대로 다시 보내므로 멱등 키도 바뀌지 않는다.
- 발행이 커밋 뒤 최대 약 1초 늦어진다. 좌석 화면은 커밋 후 WebSocket 브로드캐스트로 갱신하므로 영향이 없고, 통계만 조금 늦게 반영된다.
- 계속 실패하는 행(예: DTO 클래스 이름 변경)은 묶음 맨 앞에 남아 매 주기 실패한다. 같은 묶음의 다른 이벤트 발행은 막지 않지만, `attempts`와 에러 로그로 찾아 직접 처리해야 한다.
- 입장 요청(`entry.requested`)과 입장 허용(`entry.granted`)은 Redis 작업(대기 줄, 토큰 발급)과 묶여 있어서 DB outbox로는 원자성을 만들 수 없다. 이 둘은 outbox 대상에서 빼고 다른 방법으로 유실에 대비했다.

## 느낀 점

커밋 후 직접 발행하던 코드는 평소에는 문제없이 동작했다. 커밋과 발행 사이의 틈은 장애가 나야 드러나는 종류라, 이번에는 기능을 바꾸기 전에 선택지를 문서로 비교하고 골랐다. outbox를 적용하면서 다른 토픽도 같은 틈이 있는지 하나씩 확인했는데, 그 과정에서 입장 요청 발행 실패 시 대기 화면이 멈추는 문제를 찾았다.

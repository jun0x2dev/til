# TIL

개발하며 겪은 문제와 해결을 기록합니다.

## 작성 형식

```
# 제목

## 문제
## 원인
## 해결
## 느낀 점
```

## 목차

### [spring](./spring)

- [잘못된 요청과 없는 경로가 500으로 응답됐다](./spring/client-errors-returned-as-500.md)
- [WebSocket(STOMP)을 켜면 @Scheduled가 STOMP 스레드에서 돈다](./spring/scheduled-runs-on-stomp-scheduler.md)
- [ApplicationRunner는 웹 서버가 요청을 받기 시작한 뒤에 실행된다](./spring/application-runner-after-web-server.md)

### [jpa](./jpa)

- [비관적 락은 경쟁하는 양쪽 모두에 걸어야 의미가 있다](./jpa/pessimistic-lock-on-both-sides.md)
- [외부 API 호출을 트랜잭션 밖으로 뺐더니 커밋이 쪼개졌다](./jpa/transaction-template-around-external-call.md)
- [선점 5분이 지나도 스케줄러가 돌기 전이면 결제가 성공했다](./jpa/hold-deadline-checked-only-by-scheduler.md)

### [db](./db)

### [java](./java)

- [API에서 LocalDateTime으로 시각을 받으면 시간대가 조용히 어긋난다](./java/localdatetime-in-api-timezone.md)
- [javac는 통과한 스트림 코드를 Eclipse 컴파일러는 컴파일 오류로 봤다](./java/ecj-wildcard-capture-inference.md)

### [kafka](./kafka)

- [역직렬화 안 되는 Kafka 메시지 하나가 파티션 전체를 멈춘다](./kafka/kafka-poison-message.md)
- [Kafka 리스너에서 InterruptedException을 삼키면 처리하지 않은 메시지가 커밋된다](./kafka/kafka-listener-interrupted-exception.md)
- [커밋 뒤에 바로 발행하면 그 사이에 좌석 이벤트가 사라질 수 있었다](./kafka/transactional-outbox-skip-locked.md)
- [Kafka 발행에 실패한 입장 요청이 대기 줄에 그대로 남았다](./kafka/kafka-send-failure-leaves-request-in-queue.md)

### [redis](./redis)

- [동시 결제 10건에 PG 승인이 10번 나갔다](./redis/redis-lock-before-external-call.md)
- [DB 트랜잭션이 롤백돼도 Redis는 되돌아가지 않는다](./redis/redis-side-effects-and-db-transaction.md)

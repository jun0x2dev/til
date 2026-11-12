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

### [jpa](./jpa)

- [비관적 락은 경쟁하는 양쪽 모두에 걸어야 의미가 있다](./jpa/pessimistic-lock-on-both-sides.md)
- [외부 API 호출을 트랜잭션 밖으로 뺐더니 커밋이 쪼개졌다](./jpa/transaction-template-around-external-call.md)

### [db](./db)

### [java](./java)

- [API에서 LocalDateTime으로 시각을 받으면 시간대가 조용히 어긋난다](./java/localdatetime-in-api-timezone.md)

### [kafka](./kafka)

- [역직렬화 안 되는 Kafka 메시지 하나가 파티션 전체를 멈춘다](./kafka/kafka-poison-message.md)
- [Kafka 리스너에서 InterruptedException을 삼키면 처리하지 않은 메시지가 커밋된다](./kafka/kafka-listener-interrupted-exception.md)

### [redis](./redis)

- [동시 결제 10건에 PG 승인이 10번 나갔다](./redis/redis-lock-before-external-call.md)
- [DB 트랜잭션이 롤백돼도 Redis는 되돌아가지 않는다](./redis/redis-side-effects-and-db-transaction.md)

# DB 트랜잭션이 롤백돼도 Redis는 되돌아가지 않는다

> [ticketing-simulator](https://github.com/jun0x2dev/ticketing-simulator)에서 겪은 일이다.

## 문제

좌석 선점은 Redis 락과 "토큰당 1석" 표시를 먼저 잡고, 그다음 DB의 좌석 상태를 `HELD`로 바꾼다. 확정과 해제는 반대로 Redis 표시를 먼저 지우고 DB를 바꾼다.

이 상태에서 DB 커밋이 실패하면 두 저장소가 어긋났다.

| 상황 | DB | Redis | 결과 |
|---|---|---|---|
| 선점 커밋 실패 | `AVAILABLE` | 락과 표시가 남음 (TTL 5분) | 5분 동안 아무도 그 좌석을 못 잡고, 그 사용자도 다른 좌석을 못 잡는다 |
| 확정/해제 커밋 실패 | `HELD` 그대로 | 표시가 이미 지워짐 | 같은 토큰으로 좌석을 하나 더 잡을 수 있다 |

## 원인

- **선점:** 실패 시 락을 푸는 코드가 메서드 안의 try/catch에만 있었다. 그런데 커밋은 메서드가 정상 반환한 뒤에 프록시가 한다. 커밋 단계에서 나는 실패(DB 장애, 제약 위반 등)는 이 catch에 잡히지 않는다.
- **확정/해제:** Redis 정리를 메서드 본문에서 바로 실행했다. Redis는 DB 트랜잭션에 참여하지 않으므로 롤백되지 않는다.

## 해결

`TransactionSynchronization`으로 Redis 작업을 트랜잭션 결과에 맞춰 실행한다.

**선점: 커밋되지 않았으면 되돌린다.**
본문 예외든 커밋 실패든 하나의 경로로 처리된다. 기존 try/catch 해제 코드는 이것 하나로 합쳤다.

```java
TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
    @Override
    public void afterCompletion(int status) {
        if (status != STATUS_COMMITTED) {
            seatLockManager.unlock(seatId, lockToken);
            seatLockManager.clearHolder(entryToken);
        }
    }
});
```

**확정/해제: 커밋된 뒤에만 정리한다.**

```java
private void afterCommit(Runnable action) {
    TransactionSynchronizationManager.registerSynchronization(new TransactionSynchronization() {
        @Override
        public void afterCommit() {
            action.run();
        }
    });
}
```

확정은 결제 서비스의 바깥 트랜잭션(결제 기록, outbox 포함) 안에서 호출된다. 그래서 바깥 트랜잭션 전체가 커밋된 뒤에 정리된다. 동기화는 현재 트랜잭션에 등록되기 때문이다.

재현 테스트는 바깥 트랜잭션에서 서비스를 호출한 뒤 일부러 롤백시켜 보면 된다. 수정 전에는 롤백 뒤에도 Redis에 락이 남았다.

## 느낀 점

- `@Transactional` 메서드 안에 있다고 다 같이 롤백되는 게 아니었다. Redis, 메시지 발행, 외부 API, 파일 쓰기처럼 트랜잭션에 참여하지 않는 작업은 따로 챙겨야 한다.
- 정리하면 이렇다.
  - 되돌려야 하는 것은 `afterCompletion`에서 커밋 여부를 보고 되돌린다.
  - 커밋을 전제로 하는 것은 `afterCommit`에서 실행한다.
- 메시지 발행도 같은 문제를 갖는다. 커밋과 발행을 원자적으로 묶어야 한다면 `afterCommit`만으로는 부족하고 Transactional Outbox가 필요하다. 커밋 직후 앱이 죽으면 `afterCommit`이 실행되지 않기 때문이다.

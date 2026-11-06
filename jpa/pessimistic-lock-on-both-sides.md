# 비관적 락은 경쟁하는 양쪽 모두에 걸어야 의미가 있다

> [ticketing-simulator](https://github.com/jun0x2dev/ticketing-simulator)에서 겪은 일이다.

## 문제

좌석 상태는 두 곳에서 바뀐다.

- **TTL 스케줄러:** 선점하고 5분이 지난 `HELD` 좌석을 `AVAILABLE`로 되돌린다.
- **결제 확정:** `HELD` 좌석을 `CONFIRMED`로 바꾼다.

스케줄러 쪽 조회에는 `@Lock(PESSIMISTIC_WRITE)`을 걸어 두었다. 그런데도 문제가 생겼다.

- 같은 좌석에 결제 10건을 동시에 보내면 결제 `SUCCESS`가 여러 건 저장됐다.
- 스케줄러가 좌석을 `AVAILABLE`로 복구한 직후에 결제가 그 위에 `CONFIRMED`를 덮어쓸 수 있었다. 그러면 "TTL 만료로 해제" 이벤트와 "확정" 이벤트가 둘 다 나간다.

## 원인

결제 쪽은 잠금 없이 `findById`로 읽었다.

```
스케줄러: SELECT ... FOR UPDATE (잠금) ── AVAILABLE로 UPDATE ── COMMIT
결제:     SELECT (잠금 없음, HELD로 보임) ── 검증 통과 ── UPDATE 대기 ······ 잠금 풀림 → status=CONFIRMED로 덮어씀
```

- `FOR UPDATE`가 건 배타 잠금은 잠금을 잡으려는 작업을 모두 기다리게 한다. `FOR UPDATE`, `FOR SHARE`, `UPDATE`, `DELETE`가 여기에 해당한다. 기다리지 않는 것은 잠금 없는 일반 `SELECT`뿐이다. MySQL InnoDB는 이 경우 일관된 읽기로 스냅샷을 본다.
- 그래서 결제 쪽의 `UPDATE`도 스케줄러의 잠금을 기다리긴 했다. 문제는 판단에 쓴 읽기가 잠금 없이 일어났다는 점이다.
- JPA의 dirty checking은 `UPDATE ... WHERE id = ?`를 보낸다. 상태가 바뀌었는지는 조건에 들어가지 않는다. 잠금이 풀린 뒤 이 `UPDATE`가 실행되면, 오래된 판단으로 최신 상태를 그대로 덮어쓴다.
- 게이트웨이 호출이 트랜잭션 밖에 있어서 읽기와 쓰기 사이 간격이 길었다.

## 해결

경쟁하는 쪽 모두 같은 행을 잠그고 읽는다.

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("select s from Seat s where s.id = :seatId")
Optional<Seat> findByIdForUpdate(@Param("seatId") Long seatId);
```

확정과 해제가 이것으로 읽는다. 이제 먼저 잠근 쪽이 커밋할 때까지 다른 쪽이 기다린다. 늦은 쪽은 최신 상태를 보고 판단하므로 `SEAT_NOT_HELD`로 실패한다. InnoDB의 잠금 읽기는 REPEATABLE READ에서도 스냅샷이 아니라 최신 커밋된 행을 읽는다.

잠금 뒤에는 선점자도 다시 확인한다. TTL이 지나 다른 사람이 다시 잡은 좌석을 원래 사용자의 늦은 결제가 확정하면 안 되기 때문이다.

```java
public void verifyHolder(String holderToken) {
    if (this.status != SeatStatus.HELD) {
        throw new CustomException(ErrorCode.SEAT_NOT_HELD);
    }
    if (holderToken == null || !holderToken.equals(this.heldBy)) {
        throw new CustomException(ErrorCode.SEAT_HOLDER_MISMATCH);
    }
    // TTL 스케줄러는 몇 초 간격으로만 돌기 때문에, 제한시간이 지난 선점은 스케줄러가 풀기 전이라도 여기서 거절한다.
    if (this.heldAt.plus(HOLD_TTL).isBefore(LocalDateTime.now())) {
        throw new CustomException(ErrorCode.SEAT_HOLD_EXPIRED);
    }
}
```

동시 결제 테스트로 검증했다. 잠금을 빼면 실패하고, 넣으면 통과한다.

## 느낀 점

- 한쪽만 잠가서는 소용이 없었다. 쓰기가 기다리는 것만으로는 부족하고, 판단에 쓰는 읽기부터 잠가야 한다.
- 같은 행을 바꾸는 코드 경로를 전부 찾아보는 것이 먼저다. 스케줄러, API, Consumer처럼 진입점이 다르면 놓치기 쉽다.
- 낙관적 락(`@Version`)을 썼어도 덮어쓰기는 막을 수 있었다. 다만 이 구간은 충돌이 잦다(TTL 직전의 결제, 같은 좌석 동시 결제). 낙관적 락이면 늦은 쪽마다 커밋 시점에 `OptimisticLockException`이 나고, 이걸 도메인 오류로 바꾸는 처리가 따로 필요하다. 비관적 락은 늦은 쪽이 기다렸다가 최신 상태를 읽고, 엔티티 검증에서 `SEAT_NOT_HELD`로 실패한다. 실패가 기존 검증 흐름 그대로 나와서 이쪽을 골랐다.

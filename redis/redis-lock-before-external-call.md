# 동시 결제 10건에 PG 승인이 10번 나갔다

> [ticketing-simulator](https://github.com/jun0x2dev/ticketing-simulator)에서 겪은 일이다.

## 문제

같은 좌석에 결제 요청 10건을 동시에 보냈다.

- 좌석 확정은 1건만 됐다. DB 행 잠금(`FOR UPDATE`)으로 막아 두었기 때문이다.
- 그런데 결제 게이트웨이 `charge()`는 10번 호출됐다.
- 나머지 9건은 승인을 받은 뒤 확정 단계에서 실패했고, 승인은 취소되지 않았다.

실제 PG였다면 한 사람에게 9번 결제되고 환불되지 않았을 것이다.

## 원인

```
요청 A ─ 상태 확인(HELD) ─ PG 승인 ─────── 확정(행 잠금) 성공
요청 B ─ 상태 확인(HELD) ─ PG 승인 ─────── 확정(행 잠금) 실패, 이미 확정됨
요청 C ─ 상태 확인(HELD) ─ PG 승인 ─────── 확정(행 잠금) 실패
```

- 행 잠금은 확정 단계에만 있었다. 게이트웨이를 부르기 전의 상태 확인은 잠금 없이 읽으므로, 동시에 들어온 요청이 모두 `HELD`를 보고 각자 승인을 요청했다.
- 게이트웨이 호출은 일부러 트랜잭션 밖에 둔다(지연 동안 DB 커넥션을 붙잡지 않으려고). 그래서 확인과 확정 사이의 간격이 길어 겹치기 쉬웠다.
- 확정에 실패했을 때 승인을 되돌리는 경로가 없었다.

## 해결

**1. 외부 호출 전에 좌석 단위 Redis 락을 잡는다.**

```java
String lockKey = "payment:lock:" + seatId;
String lockToken = redisLock.tryLock(lockKey, lockTtl);   // payment.lock-ttl-ms, 기본 30초
if (lockToken == null) {
    throw new CustomException(ErrorCode.PAYMENT_IN_PROGRESS);   // 409, 바로 거절
}
try {
    // 상태 확인은 락을 잡은 "뒤"에. 앞선 결제가 막 확정하고 락을 놓은 경우를 걸러낸다.
    SeatResponse seat = seatService.getHeldSeat(seatId, entryToken);
    ChargeResult charge = paymentGateway.charge();
    return charge.approved()
            ? confirm(seat, entryToken, charge)
            : decline(seat, entryToken);
} finally {
    redisLock.unlock(lockKey, lockToken);
}
```

**2. 락 해제는 "내 락일 때만"을 Lua 스크립트 하나로 처리한다.**
`GET`으로 확인하고 `DEL`로 지우는 두 명령으로 나누면, 그 사이에 TTL이 만료되고 다른 요청이 새로 잡은 락을 지울 수 있다.

```lua
if redis.call('GET', KEYS[1]) == ARGV[1] then
  return redis.call('DEL', KEYS[1])
end
return 0
```

**3. 승인 후 확정에 실패하면 승인을 취소한다.**
결제 마감이 지났거나 상태가 바뀐 경우다. 취소마저 실패하면 원래 예외를 가리지 않도록 예외를 던지지 않고, 승인 번호와 함께 error 로그를 남겨 사람이 처리하게 한다.

테스트에서는 게이트웨이에 200ms 지연을 넣어 요청이 실제로 겹치게 했다. 그리고 `charge()`가 1번만 호출되는지 검증했다. 락을 끄면 이 테스트가 실패한다.

## 느낀 점

- DB 잠금이 막는 건 DB 상태가 두 번 바뀌는 것까지다. 그 앞에서 나가는 결제, 메일, 문자 같은 외부 호출은 막지 못한다. 잠금이 막아야 할 동작을 실제로 감싸고 있는지 봐야 한다.
- 지연 없는 mock으로 테스트하면 요청이 겹치지 않아서 이 버그가 안 보인다. 동시성 테스트에는 실제와 비슷한 지연을 넣어야 한다.
- 남는 한계가 있다. PG 응답이 락 TTL보다 오래 걸리면 락이 풀린다. 그래서 PG 타임아웃은 반드시 락 TTL보다 짧게 둔다. 그래도 확정은 행 잠금으로 1건만 되고, 나머지는 승인 취소 경로를 탄다.

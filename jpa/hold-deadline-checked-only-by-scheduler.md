# 선점 5분이 지나도 스케줄러가 돌기 전이면 결제가 성공했다

> [ticketing-simulator](https://github.com/jun0x2dev/ticketing-simulator)에서 겪은 일이다.

## 문제

좌석을 선점하면 5분 안에 결제해야 한다. 5분이 지나면 TTL 스케줄러가 좌석을 `AVAILABLE`로 되돌린다.

그런데 선점 후 5분이 지났어도 스케줄러가 아직 돌지 않았으면 결제가 성공했다. 테스트로 재현했다. `held_at`을 5분 1초 전으로 바꾸면 결제 게이트웨이가 호출되고 결제가 승인된다.

같은 틈에 다른 문제도 생길 수 있었다. 이 프로젝트는 입장 토큰 하나가 좌석 1석만 잡도록 Redis에 `seat:holder:{token}` 표시를 둔다. 이 표시는 Redis TTL로 정확히 5분에 사라진다. 그러면 같은 토큰으로 다른 좌석을 잡은 뒤 원래 좌석까지 결제해서 2석을 가질 수 있다.

## 원인

5분 마감을 판단하는 곳이 TTL 스케줄러 하나뿐이었다.

```java
@Scheduled(fixedDelay = 5000)
public void releaseExpiredSeats() {
    seatService.releaseExpiredSeats(LocalDateTime.now().minus(Seat.HOLD_TTL));
}
```

- 스케줄러는 이전 실행이 끝난 뒤 5초 간격으로 돈다. 그래서 5분이 지난 뒤 최대 몇 초 동안은 좌석이 여전히 `HELD`다.
- 결제 확정은 "`HELD`이고 선점자가 맞다"만 확인했다. 만료 여부는 스케줄러가 이미 돌았는지에 따라 달라졌다.
- Redis의 1석 표시는 TTL로 정확히 5분에 사라지는데, DB의 선점 상태는 스케줄러 주기만큼 늦게 풀린다. 두 마감 시각이 서로 달랐다.

## 해결

마감을 엔티티가 직접 판단하게 했다. 결제 전 확인, 확정, 해제가 모두 `Seat.verifyHolder()`를 거치므로 한 곳에만 넣으면 된다.

```java
public static final Duration HOLD_TTL = Duration.ofMinutes(5);

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

- 마감이 지난 선점은 `SEAT_HOLD_EXPIRED`(409)로 거절한다.
- 스케줄러는 만료된 좌석을 DB에서 정리하는 역할만 맡는다. 마감을 지키는 일은 스케줄러 주기와 상관없어졌다.
- 결제 확정 시점의 검사도 같은 메서드라, 게이트웨이 응답을 기다리는 사이에 5분이 지나는 경우도 거절된다. 이때는 이미 승인된 결제를 취소한다.

통합 테스트 두 개로 확인했다. 수정 전 코드에서는 실패하고 수정 후에는 통과했다.

```java
@Test
@DisplayName("제한시간이 지난 선점은 스케줄러 복구 전이라도 SEAT_HOLD_EXPIRED로 거절된다")
void pay_afterHoldTtl_throwsBeforeSchedulerRelease() {
    jdbcTemplate.update("update seats set held_at = ? where id = ?",
            LocalDateTime.now().minus(Seat.HOLD_TTL).minusSeconds(1), seatId);

    assertThatThrownBy(() -> paymentService.pay(seatId, entryToken))
            .extracting(e -> ((CustomException) e).getErrorCode())
            .isEqualTo(ErrorCode.SEAT_HOLD_EXPIRED);
    verify(paymentGateway, never()).charge();
}

@Test
@DisplayName("결제 중 선점이 만료되어 확정에 실패하면 승인을 취소하고 CANCELLED로 기록한다")
void pay_holdExpiresDuringCharge_cancelsApproval() {
    when(paymentGateway.charge()).thenAnswer(invocation -> {
        // 게이트웨이 응답을 기다리는 동안 제한시간이 지난 상황을 만든다
        jdbcTemplate.update("update seats set held_at = ? where id = ?",
                LocalDateTime.now().minus(Seat.HOLD_TTL).minusSeconds(1), seatId);
        return ChargeResult.approved("approval-late");
    });

    assertThatThrownBy(() -> paymentService.pay(seatId, entryToken))
            .extracting(e -> ((CustomException) e).getErrorCode())
            .isEqualTo(ErrorCode.SEAT_HOLD_EXPIRED);
    verify(paymentGateway).cancel("approval-late");
}
```

## 느낀 점

- 마감이 스케줄러 주기만큼 늦어졌고, Redis 쪽 마감과도 어긋났다.
- 마감 판단을 상태를 가진 엔티티로 옮기고, 스케줄러에는 정리만 맡기기로 했다.
- 5분을 기다리지 않고 `held_at`을 과거로 바꾸는 방식으로 테스트에서 바로 재현했다.

# 외부 API 호출을 트랜잭션 밖으로 뺐더니 커밋이 쪼개졌다

> [ticketing-simulator](https://github.com/jun0x2dev/ticketing-simulator)에서 겪은 일이다.

## 문제

결제는 이렇게 흘러간다.

```
좌석 확인 → PG 승인 (200ms~1.5초 지연) → 좌석 확정 → 결제 기록 저장 → 이벤트 발행
```

PG가 응답을 기다리는 동안 DB 커넥션을 붙잡지 않으려고, `pay()`에서 트랜잭션을 뺐다(`Propagation.NOT_SUPPORTED`). 그랬더니 결제 기록 저장이 실패한 경우 좌석은 `CONFIRMED`인데 결제 기록이 없는 상태가 생겼다. 확정 이벤트도 발행되지 않아 통계와 어긋났다.

테스트로 재현했다. `PaymentRepository.save`가 예외를 던지게 하면 좌석이 `CONFIRMED`로 남는다.

## 원인

`pay()`에 트랜잭션이 없으니, 그 안에서 부른 메서드가 각자 트랜잭션을 열고 커밋했다.

```java
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public PaymentResponse pay(...) {
    paymentGateway.charge();
    seatService.confirmSeat(...);      // 자체 @Transactional → 여기서 이미 커밋
    paymentRepository.save(...);       // 자체 트랜잭션 → 여기서 실패
}
```

"외부 호출은 트랜잭션 밖으로"를 지키다가 "함께 바뀌어야 하는 것은 한 트랜잭션으로"를 깨뜨렸다.

## 해결

외부 호출 이후의 DB 작업만 `TransactionTemplate`으로 묶었다. 선언적 `@Transactional`은 메서드 단위라서 "메서드의 뒷부분만"을 표현하기 어렵다.

```java
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public PaymentResponse pay(Long seatId, String entryToken) {
    ...
    ChargeResult charge = paymentGateway.charge();          // 트랜잭션 밖 (커넥션 안 잡음)
    return charge.approved() ? confirm(seat, entryToken, charge) : decline(seat, entryToken);
}

private PaymentResponse confirm(SeatResponse seat, String entryToken, ChargeResult charge) {
    Long seatId = seat.id();
    Payment payment;
    try {
        payment = transactionTemplate.execute(status -> {
            seatService.confirmSeat(seatId, entryToken);                           // 바깥 트랜잭션에 참여
            Payment saved = paymentRepository.save(Payment.create(seatId, PaymentStatus.SUCCESS, charge.approvalId()));
            outboxService.append(KafkaTopicConfig.SEAT_CONFIRMED, String.valueOf(seat.eventId()),
                    SeatConfirmedEvent.of(seat.eventId(), seatId, saved.getId()));  // 같이 커밋
            return saved;
        });
    } catch (RuntimeException e) {
        cancelApproval(seatId, charge.approvalId());    // 셋 다 롤백됐으니 승인도 되돌린다
        throw e;
    }
    log.info("결제 승인 - seatId: {}, paymentId: {}", seatId, payment.getId());
    return PaymentResponse.from(payment);
}
```

- `confirmSeat`의 `@Transactional`은 기본 전파(`REQUIRED`)라서, 새 트랜잭션을 열지 않고 바깥 트랜잭션에 참여한다.
- 좌석 확정, 결제 기록, 발행할 이벤트(outbox)가 함께 커밋되거나 함께 롤백된다.
- 롤백되면 PG 승인만 남으므로, 승인 취소로 되돌린다.

## 느낀 점

- 트랜잭션 범위를 줄이다가 함께 바뀌어야 하는 것까지 갈라 버렸다. 범위를 줄일 때는 같이 커밋돼야 하는 것들이 여전히 한 묶음인지 봐야 한다.
- 메서드 일부만 트랜잭션으로 묶어야 할 때는 `TransactionTemplate`이 가장 분명하다. 같은 클래스 안에 `@Transactional` private 메서드를 만드는 방법은 프록시를 거치지 않아 적용되지 않는다.
- 외부 호출과 DB는 어차피 원자적으로 묶을 수 없다. DB 쪽이 실패했을 때 외부 쪽을 되돌리는 경로(여기서는 승인 취소)를 같이 만들어 둬야 한다.

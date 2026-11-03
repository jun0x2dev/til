# API에서 LocalDateTime으로 시각을 받으면 시간대가 조용히 어긋난다

## 문제

이벤트 생성 API가 오픈 시각을 `LocalDateTime`으로 받았다.

```java
public record EventCreateRequest(..., @NotNull @Future LocalDateTime openAt) {}
```

k6 부하 테스트 스크립트가 "1시간 뒤 오픈"인 이벤트를 만들자, `@Future` 검증에 걸려 실패했다.

- k6는 UTC 컨테이너에서 돈다. `new Date(...).toISOString()`의 끝 `Z`를 떼고 보냈다.
- 서버는 KST다. UTC로 "1시간 뒤"는 KST 기준으로 8시간 전이다.

당시에는 k6가 이틀 뒤로 잡게 우회했다. 9시간이 어긋나도 이틀 뒤면 여전히 미래라서 검증을 통과했다.

그런데 나중에 "오픈 시각 전에는 입장을 거절한다"는 검증을 넣으면서 문제가 다시 드러났다.

- 이틀 뒤 우회를 그대로 두면 회차가 이틀 가까이 열리지 않아 부하 테스트를 할 수 없다.
- k6가 "3초 뒤 오픈"을 오프셋 없이 보내면 서버는 9시간 이른 시각, 즉 이미 지난 시각으로 읽는다. 그래서 `@Future`에 걸린다.

## 원인

`LocalDateTime`에는 어느 시간대의 시각인지 정보가 없다. `"2026-11-02T10:00:00"`을 받으면 서버는 자기 시간대의 10시로 해석한다. 보낸 쪽이 UTC 10시를 뜻했어도 알 방법이 없다.

에러는 나지 않는다. 그냥 다른 순간으로 저장된다.

## 해결

API 입력은 `OffsetDateTime`으로 받아 오프셋을 필수로 했다. 저장할 때 서버 시간대로 바꾼다.

```java
public record EventCreateRequest(..., @NotNull @Future OffsetDateTime openAt) {}

// 오픈 시각 비교와 회차 정리를 서버 시각(LocalDateTime.now())으로 하므로, 같은 순간의 서버 시간대 시각으로 바꿔 저장한다.
LocalDateTime openAt = request.openAt()
        .atZoneSameInstant(ZoneId.systemDefault())
        .toLocalDateTime();
```

- `...Z`, `...+09:00`처럼 오프셋이 있는 시각만 받는다. 오프셋 없는 시각은 역직렬화에 실패해 400이 된다.
- k6는 `new Date(openAtMs).toISOString()`을 그대로 보낸다(`Z` 포함).
- 실서버에서 확인했다. `02:42:15Z`(KST 11:42:15)로 보낸 회차에 입장을 요청하면 오픈 전에는 409, 오픈 후에는 202가 나왔다. 응답의 `openAt`은 서버 시간대인 `11:42:15`였다.

DB 저장도 한 번 더 확인했다. JDBC URL이 `serverTimezone=UTC`라서 MySQL에는 `02:42`(UTC)로 저장되고, 다시 읽으면 `11:42`(KST)로 돌아온다. 같은 순간이 유지된다.

## 느낀 점

- `LocalDateTime`은 서버 안에서 한 시간대로만 쓸 때는 편하다. 문제는 API나 메시지처럼 밖에서 들어오는 값이다. 다른 시간대의 클라이언트가 보내면 시간대 정보가 그대로 사라진다.
- 밖에서 받는 값은 `OffsetDateTime`이나 `Instant`로 받고, 내부 표현으로는 한 곳에서 한 번만 바꾸기로 했다.
- 에러 없이 틀린 값이 들어간다는 게 제일 곤란했다. 이틀 뒤로 잡는 우회로 넘어갔더니, 나중에 그 값을 쓰는 검증이 생기면서 다른 모습으로 다시 터졌다.
- 서버를 옮겨도 해석이 바뀌지 않게 JVM 시간대(`-Duser.timezone`)와 DB 연결 시간대도 명시해 두는 게 좋겠다.

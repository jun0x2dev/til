# ApplicationRunner는 웹 서버가 요청을 받기 시작한 뒤에 실행된다

> [ticketing-simulator](https://github.com/jun0x2dev/ticketing-simulator)에서 겪은 일이다.

## 문제

기본 공연 데이터 10개를 `ApplicationRunner`로 넣고 있었다.

앱을 띄우자마자 `GET /api/v1/concerts`를 호출하면 빈 목록(`[]`)이 나왔다. 몇 초 뒤 다시 호출하면 10개가 정상으로 나왔다.

## 원인

Spring Boot의 시작 순서는 이렇다.

```
SpringApplication.run()
 └ refresh()
    ├ 싱글톤 생성 ─ SmartInitializingSingleton.afterSingletonsInstantiated()
    └ finishRefresh() ─ 내장 웹 서버 시작  ← 여기서부터 요청을 받음
 └ callRunners() ─ ApplicationRunner / CommandLineRunner 실행
```

웹 서버는 컨텍스트 refresh의 마지막 단계에서 시작한다. `ApplicationRunner`는 refresh가 끝난 뒤에 실행된다. 그 사이에 들어온 요청은 데이터가 들어가기 전의 빈 테이블을 조회한다.

## 해결

시드 작업을 `SmartInitializingSingleton`으로 바꿨다.

```java
@Component
@RequiredArgsConstructor
public class ConcertCatalogInitializer implements SmartInitializingSingleton {

    private final ConcertService concertService;

    @Override
    public void afterSingletonsInstantiated() {
        concertService.initializeCatalog();
    }
}
```

`afterSingletonsInstantiated()`의 호출 시점은 다음 조건을 모두 만족한다.

- 모든 싱글톤이 만들어진 뒤라서 JPA와 트랜잭션 프록시를 쓸 수 있다.
- 웹 서버가 시작되기 전이다.

그래서 서버가 요청을 받기 시작할 때는 시드가 끝나 있다.

## 느낀 점

- `ApplicationRunner`가 앱이 준비된 뒤에 도는 줄 알았는데, 실제로는 이미 요청을 받고 있는 중에 돈다. 요청 처리에 필요한 초기화(시드, 캐시 워밍)를 넣기에는 맞지 않았다.
- 로컬에서는 사람이 앱을 띄우고 브라우저를 여는 사이에 시드가 끝나서 문제가 보이지 않는다. 포트가 열리자마자 요청이 들어오는 환경에서 드러난다. 포트만 확인하는 로드밸런서, liveness만 보는 환경, 프로브 없이 직접 호출하는 클라이언트가 그렇다.
- Spring Boot의 readiness 상태는 `ApplicationRunner`까지 끝난 뒤에야 `ACCEPTING_TRAFFIC`가 된다. `SpringApplication.run()`이 `callRunners()` 다음에 ready 이벤트를 보내기 때문이다. 그래서 트래픽을 readiness 프로브(actuator의 `/actuator/health/readiness`)로 제어하는 환경이라면 Runner로 시드해도 요청이 먼저 들어오지 않는다. 다만 readiness는 트래픽을 보내는 쪽이 지켜 주는 약속일 뿐이다. 포트로 직접 들어오는 요청은 막지 못한다. 이 프로젝트는 actuator가 없어 프로브도 없었다.
- 시드 데이터가 고정이라면 DB 마이그레이션(Flyway 등)으로 넣는 것이 가장 깔끔하다.

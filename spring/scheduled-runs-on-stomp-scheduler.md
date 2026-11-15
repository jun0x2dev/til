# WebSocket(STOMP)을 켜면 @Scheduled가 STOMP 스레드에서 돈다

> [ticketing-simulator](https://github.com/jun0x2dev/ticketing-simulator)에서 겪은 일이다.

## 문제

outbox 발행기를 `@Scheduled`로 붙이고 SQL 로그를 보다가 이상한 점을 발견했다. 발행 쿼리(`... for update skip locked`)가 `MessageBroker-2` 스레드에서 실행되고 있었다.

- `spring.task.scheduling.pool.size=4`로 설정했는데 `scheduling-*` 스레드는 하나도 생기지 않았다.
- TTL 복구, 회차 정리 같은 다른 `@Scheduled` 작업도 같은 스레드 풀에서 돌고 있었다.

## 원인

세 가지가 맞물렸다.

1. `@EnableWebSocketMessageBroker`는 STOMP 하트비트용 `TaskScheduler` 빈(`messageBrokerTaskScheduler`)을 등록한다.
2. Spring Boot의 `@Scheduled`용 스케줄러 자동 설정은 `@ConditionalOnMissingBean`이다. `TaskScheduler` 빈이 하나라도 있으면 만들어지지 않는다. 그래서 `spring.task.scheduling.*` 설정도 적용되지 않았다.
3. `@Scheduled` 처리기(`ScheduledAnnotationBeanPostProcessor`)는 먼저 `TaskScheduler` 타입의 빈을 찾는다. 하나뿐이면 그것을 쓰고, 여러 개면 이름이 `taskScheduler`인 빈을 쓴다. 이 경우 STOMP 스케줄러가 유일한 빈이었다.

결국 모든 `@Scheduled` 작업이 STOMP 하트비트 스레드를 나눠 썼다.

outbox 발행은 Kafka 브로커 응답을 최대 몇 초 기다린다. 브로커 장애가 나면 이 대기가 WebSocket 하트비트를 밀어내고, 클라이언트 연결이 끊길 수 있는 구조였다.

## 해결

`taskScheduler`라는 이름의 스케줄러 빈을 직접 둔다. Spring Boot가 설정값을 반영해 만들어 둔 빌더로 만들면 `spring.task.scheduling.*` 설정도 그대로 적용된다.

```java
@Configuration
public class SchedulingConfig {

    @Bean(name = "taskScheduler")
    public ThreadPoolTaskScheduler taskScheduler(ThreadPoolTaskSchedulerBuilder builder) {
        return builder.build();
    }
}
```

이제 `TaskScheduler` 빈이 둘이므로 `@Scheduled`는 이름이 `taskScheduler`인 빈을 쓴다. STOMP는 이름으로 자기 스케줄러(`messageBrokerTaskScheduler`)를 계속 쓴다. 실서버 로그에서 발행 쿼리가 `scheduling-1`~`scheduling-4` 스레드로 옮겨간 것을 확인했다.

## 느낀 점

- `@ConditionalOnMissingBean`이 붙은 자동 설정은 다른 기능이 같은 타입의 빈을 하나만 만들어도 꺼진다. 에러도 경고도 없다.
- 설정값(`pool.size`)이 먹지 않는다면, 그 설정을 읽는 빈이 실제로 만들어졌는지부터 의심해 보자.
- 스레드 이름은 좋은 단서다. 로그에 스레드 이름이 찍히게 해 두면 "이 작업이 어디서 도는가"를 바로 알 수 있다.

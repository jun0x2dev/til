# 잘못된 요청과 없는 경로가 500으로 응답됐다

> [ticketing-simulator](https://github.com/jun0x2dev/ticketing-simulator)에서 겪은 일이다.

## 문제

- 부하 테스트 스크립트의 이벤트 생성 요청이 `500 INTERNAL_SERVER_ERROR`로 실패했다. 로그를 보니 `@Future` 검증 위반(`MethodArgumentNotValidException`)이었다.
- 나중에 API 하나를 특정 프로필에서만 열리도록 바꿨다. 그러자 프로필이 꺼진 환경에서 그 경로를 호출하면 404나 405가 아니라 500이 나왔다. 아무 경로(`/api/v1/unknown`)를 불러도 500이었다.

클라이언트 실수인데 서버 장애처럼 보인다. 모니터링에서 5xx 알람이 울리고, 클라이언트는 "재시도하면 될까?"를 판단할 수 없다.

## 원인

`@RestControllerAdvice`가 도메인 예외(`CustomException`)와 `Exception`만 처리하고 있었다.

```java
@ExceptionHandler(Exception.class)
public ResponseEntity<ApiResponse<Void>> handleException(Exception e) {
    log.error("Unhandled exception", e);
    ErrorCode errorCode = ErrorCode.INTERNAL_SERVER_ERROR;   // Spring MVC가 원래 4xx로 바꿔 주던 예외까지 여기서 500이 된다
    return ResponseEntity.status(errorCode.getStatus())
            .body(ApiResponse.failure(new ErrorResponse(errorCode.name(), errorCode.getMessage())));
}
```

Spring MVC는 요청 바인딩·검증 예외나 매핑 실패를 원래 4xx로 바꿔 준다. 그런데 `Exception` 전체를 잡는 핸들러가 있으면 그보다 먼저 가로채서 500으로 만든다.

- **검증과 바인딩:** `MethodArgumentNotValidException`, `HttpMessageNotReadableException`, `MethodArgumentTypeMismatchException`
- **없는 경로:** Spring 6.1부터 매핑이 없는 경로는 정적 리소스 처리기로 넘어가 `NoResourceFoundException`을 던진다.
- **경로는 있는데 메서드가 없음:** `HttpRequestMethodNotSupportedException`. 예를 들어 `GET /entries/{id}`만 있는데 `POST /entries/direct`를 부른 경우다.

## 첫 번째 해결: 예외를 하나씩 추가

처음에는 문제가 된 예외를 하나씩 4xx에 매핑했다. 400 검증 예외 처리기를 먼저 만들었고, 404/405 처리기는 나중에 추가했다.

```java
@ExceptionHandler(NoResourceFoundException.class)
public ResponseEntity<ApiResponse<Void>> handleNoResource(NoResourceFoundException e) {
    return clientError(ErrorCode.ENTITY_NOT_FOUND, e);            // 404
}
```

이 방식에는 구멍이 두 개 남았다.

- 처리기를 안 만든 예외는 여전히 500이었다. 예를 들어 `POST /api/v1/events`를 `Content-Type: text/plain`으로 보내면 `HttpMediaTypeNotSupportedException`이 나는데, 415가 아니라 500으로 응답했다.
- 표준 처리가 넣어 주던 헤더가 빠졌다. 405 응답에는 허용 메서드를 알리는 `Allow` 헤더가 붙어야 하는데, 직접 만든 응답에는 본문만 있었다.

## 두 번째 해결: ResponseEntityExceptionHandler 상속

Spring MVC 표준 예외 처리를 상위 클래스에 맡기고, 본문만 공통 포맷으로 바꾼다.

```java
@RestControllerAdvice
public class GlobalExceptionHandler extends ResponseEntityExceptionHandler {

    // 표준 예외의 상태 코드별 응답 코드. 여기 없는 4xx는 INVALID_REQUEST로 응답한다.
    private static final Map<HttpStatus, ErrorCode> STANDARD_ERROR_CODES = Map.of(
            HttpStatus.NOT_FOUND, ErrorCode.RESOURCE_NOT_FOUND,
            HttpStatus.METHOD_NOT_ALLOWED, ErrorCode.METHOD_NOT_ALLOWED,
            HttpStatus.UNSUPPORTED_MEDIA_TYPE, ErrorCode.UNSUPPORTED_MEDIA_TYPE);

    @Override
    protected ResponseEntity<Object> handleExceptionInternal(Exception ex, Object body, HttpHeaders headers,
                                                             HttpStatusCode statusCode, WebRequest request) {
        ResponseEntity<Object> response = super.handleExceptionInternal(ex, body, headers, statusCode, request);
        if (response == null) {
            return null;   // 응답이 이미 커밋되어 쓸 수 없는 경우
        }
        ErrorCode errorCode;
        if (statusCode.is5xxServerError()) {
            errorCode = ErrorCode.INTERNAL_SERVER_ERROR;
            log.error("Spring MVC exception", ex);
        } else {
            errorCode = STANDARD_ERROR_CODES.getOrDefault(HttpStatus.resolve(statusCode.value()), ErrorCode.INVALID_REQUEST);
            log.warn("{}: {}", errorCode, ex.getMessage());
        }
        return ResponseEntity.status(response.getStatusCode())
                .headers(response.getHeaders())          // Allow, Accept 등 유지
                .body(ApiResponse.failure(new ErrorResponse(errorCode.name(), errorCode.getMessage())));
    }

    // CustomException 처리기, Exception 처리기는 그대로 둔다
}
```

- 상위 클래스가 표준 예외 전부(검증, 바인딩, 404, 405, 415, 406 등)에 상태 코드와 헤더를 정해 준다. 처리기를 하나씩 둘 필요가 없어서, 400/404/405 개별 처리기는 지웠다. 남겨 두면 같은 예외에 처리기가 두 개가 되어 매핑 충돌이 난다.
- `super.handleExceptionInternal`을 먼저 부른다. 상위 클래스가 하던 일(응답 커밋 확인, 500일 때 요청 속성 기록)을 그대로 유지하고, 본문만 바꾸기 위해서다.
- 없는 경로에 쓰던 `ENTITY_NOT_FOUND`는 엔티티가 아니라 경로를 못 찾은 경우라 `RESOURCE_NOT_FOUND`로 이름을 바꿨다.
- 4xx는 warn, 5xx만 error 레벨로 남긴다. 클라이언트 실수가 error 로그와 알람을 채우지 않게 하기 위해서다.

415 응답과 405의 `Allow: GET` 헤더를 MockMvc 테스트로 고정했다. 수정 전 코드에서는 둘 다 실패했다.

## 느낀 점

- 안전망으로 둔 `@ExceptionHandler(Exception.class)`가 프레임워크의 기본 처리까지 덮어쓰고 있었다. 이 핸들러를 두면 프레임워크가 해 주던 4xx 매핑을 직접 다시 해야 한다.
- 예외를 하나씩 추가하는 방식은 겪은 것만 막는다. 증상(500)이 나올 때마다 처리기를 붙이다가 두 번째 구멍(415)에서 멈추고 원인에 맞는 구조로 바꿨다. 처음부터 프레임워크가 준비해 둔 기반 클래스를 찾아봤다면 한 번에 끝났을 일이다.
- "없는 경로 → 500", "잘못된 Content-Type → 500"은 정상 경로만 테스트하면 영원히 모른다. 실패 경로 테스트가 필요한 이유다.

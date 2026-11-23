# WebSocket 허용 Origin을 *로 열어 두면 다른 사이트가 알림을 엿들을 수 있었다

> [ticketing-simulator](https://github.com/jun0x2dev/ticketing-simulator)에서 겪은 일이다.

## 문제

입장 허용과 좌석 변경은 STOMP(`/ws`)로 브라우저에 실시간으로 보낸다. 엔드포인트 설정은 이랬다.

```java
@Override
public void registerStompEndpoints(StompEndpointRegistry registry) {
    // 로컬 시뮬레이터라 프론트 개발 서버(다른 포트)에서의 접속을 막지 않는다. 배포 시에는 허용 Origin을 제한한다.
    registry.addEndpoint("/ws").setAllowedOriginPatterns("*");
}
```

주석대로라면 "배포할 때 제한한다"였지만, 그 일을 맡은 코드는 없었다. 이 상태로 배포하면 다른 사이트의 페이지가 사용자의 브라우저로 `/ws`에 붙어, 입장 허용과 좌석 알림을 받아볼 수 있다.

## 원인

REST API는 브라우저가 CORS로 막아 준다. 다른 Origin의 페이지가 응답을 읽으려면 서버가 허락해야 한다.

WebSocket은 다르다. 브라우저는 다른 Origin의 페이지에서도 WebSocket 연결을 열어 주고, 핸드셰이크 요청에 `Origin` 헤더를 실어 보낼 뿐이다. 그 값을 보고 받을지 말지는 서버가 정한다(RFC 6455 10.2절).

Spring에서는 `OriginHandshakeInterceptor`가 이 판단을 한다. STOMP 엔드포인트를 등록하면 `WebMvcStompWebSocketEndpointRegistration.getInterceptors()`가 이 인터셉터를 항상 붙인다. Spring 7.0 소스의 판단 로직은 이렇다.

```java
public boolean beforeHandshake(ServerHttpRequest request, ServerHttpResponse response, ...) {
    if (!WebUtils.isSameOrigin(request) &&
            this.corsConfiguration.checkOrigin(request.getHeaders().getOrigin()) == null) {
        response.setStatusCode(HttpStatus.FORBIDDEN);
        return false;
    }
    return true;
}
```

같은 Origin이면 통과시키고, 다른 Origin이면 허용 목록에 있어야 통과시킨다. 허용 목록을 비워 두면 같은 Origin만 받는다. `setAllowedOriginPatterns("*")`가 이 검사를 꺼 버리고 있었다.

## 그런데 정말 *가 필요했을까?

주석은 프론트 개발 서버가 다른 포트라서 `*`가 필요하다고 했다. 그런데 Vite 설정을 보니 `/ws`를 백엔드로 프록시하고 있었다.

```js
proxy: {
  '/api': apiTarget,
  '/ws': { target: apiTarget, ws: true },
},
```

`changeOrigin`을 켜지 않았으므로 프록시는 브라우저가 보낸 `Host` 헤더를 그대로 넘긴다. 브라우저의 `Origin`도 개발 서버 주소이고 `Host`도 개발 서버 주소라서, 백엔드는 같은 Origin으로 판정한다. 로컬에서도 `*`가 필요 없었다.

## 해결

허용 Origin의 기본값을 빈 목록으로 바꾸고, 다른 도메인에서 화면을 서비스할 때만 설정으로 추가하게 했다.

```java
public WebSocketConfig(@Value("${websocket.allowed-origin-patterns:}") String[] allowedOriginPatterns) {
    this.allowedOriginPatterns = allowedOriginPatterns;
}

@Override
public void registerStompEndpoints(StompEndpointRegistry registry) {
    // 브라우저는 다른 사이트의 페이지에서도 WebSocket을 열 수 있으므로(CORS가 적용되지 않음), Origin 검사로 막는다.
    registry.addEndpoint("/ws").setAllowedOriginPatterns(allowedOriginPatterns);
}
```

실제 서버를 띄워 `Origin` 헤더만 바꿔 접속하는 테스트를 넣었다.

```java
@Test
@DisplayName("같은 Origin에서의 WebSocket 접속은 허용한다")
void sameOrigin_isAccepted() throws Exception {
    StompSession session = connect("http://localhost:" + port);
    assertThat(session.isConnected()).isTrue();
}

@Test
@DisplayName("다른 Origin에서의 WebSocket 접속은 거절한다")
void foreignOrigin_isRejected() {
    assertThatThrownBy(() -> connect("http://evil.example"))
            .isInstanceOf(ExecutionException.class);
}
```

## 느낀 점

`*`를 넣은 이유로 적어 둔 "개발 서버가 다른 포트"는 프록시 설정을 보니 맞지 않았다. 배포 때 하기로 미뤄 둔 제한은 기본값으로 옮겼고, 다른 Origin이 필요할 때만 설정으로 연다.

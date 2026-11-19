# javac는 통과한 스트림 코드를 Eclipse 컴파일러는 컴파일 오류로 봤다

> [ticketing-simulator](https://github.com/jun0x2dev/ticketing-simulator)에서 겪은 일이다.

## 문제

outbox 발행기에서 발행 결과 future를 목록으로 모으는 코드가 있다.

```java
List<CompletableFuture<?>> results = pending.stream().map(this::send).toList();
```

`send`는 `CompletableFuture<?>`를 반환한다. Gradle(javac) 빌드와 테스트는 통과했다. 그런데 VS Code Java 확장에서는 이 줄이 컴파일 오류로 표시됐다. VS Code Java 확장은 Eclipse 컴파일러(ECJ)를 쓴다.

## 원인

ECJ는 `map(this::send)`의 결과 타입을 추론할 때 와일드카드 `?`를 캡처 타입으로 추론했고, 그 결과를 `List<CompletableFuture<?>>` 변수에 담을 수 없다고 봤다. javac는 같은 코드를 문제없이 컴파일했다. 두 컴파일러의 추론 결과가 달랐던 것이다.

## 해결

`map`에 타입 인자를 명시했다.

```java
// 타입을 명시하지 않으면 Eclipse 컴파일러(VS Code Java)가 와일드카드를 캡처 타입으로 추론해 컴파일 오류로 본다.
List<CompletableFuture<?>> results = pending.stream().<CompletableFuture<?>>map(this::send).toList();
```

javac 빌드와 동작은 바뀌지 않는다. 다른 사람이 같은 코드에 손대며 타입 인자를 지우지 않도록 이유를 주석으로 남겼다.

## 느낀 점

빌드는 Gradle이 javac로 하고 편집기는 ECJ로 검사해서, 같은 코드에 대한 판단이 둘로 갈렸다. 와일드카드가 들어간 제네릭 타입을 스트림으로 모을 때는 타입 인자를 명시하기로 했다.

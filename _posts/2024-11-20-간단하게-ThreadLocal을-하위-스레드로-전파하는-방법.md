---
title: "간단하게 ThreadLocal을 하위 스레드로 전파하는 방법"
date: 2024-11-20 01:10:00 +0900
categories: ["Java"]
tags: [Java, Thread]
---

자바에서 동일한 스레드에서 값을 공유시켜주는 ThreadLocal 이라는 유용한 라이브러리가 존재한다. 이 말은 즉슨, 스레드 내에서 사용할 수 있는 전역변수라고 볼 수 있을 것이다. 

C의 전역변수처럼 잘 사용하면 편리하지만, 스파게티 코드를 만들기 쉬워지기 때문에 보통 스레드 도입부에서 한번 값을 저장하고 바꾸는 연산은 하지 않는게 좋을 것이다.

다음은 ThreadLocal을 사용하는 예제다. `AuthTokenHolder.getToken()`으로 해당 스레드 내에서 동일한 AuthToken을 얻을 수 있다.

```java
public final class AuthTokenHolder {
    private static final ThreadLocal<String> tokenHolder = new ThreadLocal<>();

    private AuthTokenHolder() {}
    public static void setToken(String token) {
        tokenHolder.set(token);
    }
    public static void free() {
        tokenHolder.remove();
    }
    public static String getToken() {
        return tokenHolder.get();
    }
}
```

이 유용한 스레드 전용 전역변수를 새로 만들어낸 하위 스레드에 전파하고 싶을 때가 있다. 예를 들면, 다른 서버로 비동기 호출을 하려고 하는데, 현재 스레드가 ThreadLocal로 들고 있는 인증정보를 비동기 호출될 하위 스레드에 전파시켜야 제대로 인증절차가 진행될 것이다.

첫번째는 현재 ThreadLocal의 값을 비동기 호출 코드에서 저장시키는 방법이다. 말만 들으면 이해가 안되니 예제 코드를 보자.

```java
    public CompletableFuture<MemberInfoResponseDto> getMemberInfoAsync() {
        String tok = AuthTokenHolder.getToken();
        return CompletableFuture.supplyAsync(() -> {
            try {
                AuthTokenHolder.setToken(tok);
                MemberInfoResponseDto resp = this.getMemberInfo();
            } finally{
                AuthTokenHolder.free();
            }
            return resp;
        });
    }
```
현재 스레드의 ThreadLocal 값을 CompletableFuture의 람다 함수를 줄때 넣어서 자식 스레드에서 ThreadLocal에 저장하도록 한다. 매번 이런 형식의 문장을 쓰기 귀찮다면 해당 형태를 Functional Interface factory로 만들 수 있다.

```java
public final class FunctionalWithAuthToken {
    private FunctionalWithAuthToken() {}
    public static <T> Supplier<T> supply(String authToken, Supplier<T> supplier) {
        return () -> {
            try{
                AuthTokenHolder.setToken(authToken);
                return supplier.get();
            } finally {
                AuthTokenHolder.free();
            }
        };
    }
}
```

이제 다음과 같이 축약할 수 있다.
```java
    public CompletableFuture<MemberInfoResponseDto> getMemberInfoAsync() {
        String tok = AuthTokenHolder.getToken();
        return CompletableFuture.supplyAsync(
            FucntionalWithAuthToken.supply(AuthTokenHolder.getToken(), this::getMemberInfo)
        );
    }
```

두번째 방법은 아주 간단한데, `InheritableThreadLocal`을 사용하는 방법이다.
```java
private static final ThreadLocal<String> tokenHolder = new InheritableThreadLocal<>();
```
이러면 해당 스레드에서 생성한 자식 노드에 ThreadLocal이 전파되어 위처럼 임의로 전파하지 않아도 된다. 

그러나 이 방법은 Thread Pool을 사용할 때 문제가 된다. 따라서 첫번째의 임의 전달 방법을 알아놓아야 한다. 이 때문에 고생을 꽤나 했는데, 다음에 한번 알아보도록 하자.
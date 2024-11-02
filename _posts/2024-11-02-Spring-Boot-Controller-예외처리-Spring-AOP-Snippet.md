---
title: "Spring Boot Controller 예외처리 Spring AOP Snippet"
date: 2024-11-02 21:44:00 +0900
categories: ["Spring Boot"]
tags: [Spring Boot, Java, Exception, AspectJ, Code Snippet]
---

Spring으로 서버 사이드 렌더링을 할때 예외상황에서 리디렉트를 해야할 상황이 생긴다. 

Spring AOP와 Annotation을 조합하여 특정 예외가 던져졌을 시 리디렉트를 할 수 있는 코드 스니펫이다.

먼저 체크할 예외와 리디렉트할 경로를 설정하는 어노테이션이다.
```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RedirectOnException {
    // 체크할 예외
    Class<? extends Exception>[] exceptions() default {Exception.class};

    // 리디렉트 경로
    String value();
}
```

기본적으로, 동일한 어노테이션을 중복해서 쌓을 수 없다. 만약 예외별로 리디렉트 경로를 다르게 바꾸고 싶다면 어노테이션 컨테이너를 정의하면 된다.

```java
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
@Repeatable(RedirectOnExceptionContainer.class) // Repeatable Annotation
public @interface RedirectOnException {
    Class<? extends Exception>[] exceptions() default {Exception.class};
    String value();
}

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface RedirectOnExceptionContainer {
    RedirectOnException[] value();
}
```

이제 Annotation을 감시하는 Aspect를 정의한다.

```java
@Aspect
@Component
public class RedirectAspect {
    @Pointcut("@annotation(com.flowerfulfort.annotation.RedirectOnException)")
    public void redirectCut(){}

    // 다중 RedirectOnException 적용했을 경우, 
    // 실제로 RedirectOnExceptionContainer 가 붙어있는 것으로 취급함.
    @Pointcut("@annotation(com.flowerfulfort.annotation.RedirectOnExceptionContainer)")
    public void redirectContainerCut(){}

    // RedirectOnException 어노테이션을 읽어 예외가 발생했을때
    // 해당 경로로 메세지와 함께 throw 하여
    // ExceptionControllerAdvice 에서 캐치하게 해줌.
    @Around("redirectCut() || redirectContainerCut()")
    public Object redirectThrowable(ProceedingJoinPoint joinPoint) throws Throwable {
        Object value;
        try {
            value = joinPoint.proceed();
        }catch (Exception e) {
            MethodSignature signature = (MethodSignature) joinPoint.getSignature();
            Method method = signature.getMethod();
            // @RedirectOnException 을 여러개 붙일수 있음!
            RedirectOnException[] r = 
                method.getAnnotationsByType(RedirectOnException.class);
            if (r.length == 0) {
                throw new IllegalArgumentException();
            }
            for (RedirectOnException redirect: r) {
                Class<? extends Exception>[] target = redirect.exceptions();
                // annotation 에서 설정한 class 의 하위 클래스이면
                // RedirectRuntimeException 으로 재포장해서 throw.
                for (Class<? extends Exception> clazz: target) {
                    // 예외를 clazz로 assign 할 수 있는지 체크
                    if (clazz.isAssignableFrom(e.getClass())) {
                        throw new RedirectException(e, redirect.value());
                    }
                }
            }
            // 아니면 그냥 throw.
            throw e;
        }
        return value;
    }
}
```

`joinPoint.proceed()`를 try 문으로 감싸 예외를 체크한다. 만약 Annotation에서 넘겨받은 예와타입으로 취급할 수 있는 경우, `RedirectException`으로 감싸 다시 던진다.

다음은 다시 던지는 예외와 해당 예외를 처리하는 ControllerAdvice 이다.

```java
@Getter
public class RedirectException extends RuntimeException {
    private final String redirect;
    public RedirectRuntimeException(Exception e, String redirectTo) {
        super(e);
        this.redirect = redirectTo;
    }
}

@ControllerAdvice
public class ExceptionControllerAdvice {
    @ExceptionHandler({RedirectException.class})
    public ModelAndView redirectExceptionHandler(RedirectException e) {
        ModelAndView mv = new ModelAndView("error/alertMessageRedirect");
        mv.addObject("exception", ExceptionUtils.getRootCauseMessage());
        mv.addObject("redirect", e.getRedirect());
        return mv;
    }
}
```

예외에서 redirect 주소와, Apache commons lang3 패키지의 `ExceptionUtils`를 이용해 Cause exception의 메시지를 꺼내 Model attribute로 사용한다.

Thymeleaf 기준으로, 다음과 같은 간단한 Alert 메시지와 함께 리디렉트를 할 수 있다.

```html
<script th:inline="javascript">
    alert([[${exception}]]);
    window.location.href=[[${redirect}]];
</script>
```

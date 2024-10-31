---
title: "Prototype Bean, 그리고 ObjectProvider"
date: 2024-10-31 13:08:00 +0900
categories: ["Spring Boot"]
tags: [Spring Boot, Java]
---

Spring Boot로 프로젝트를 하는 도중, Bean scope가 prototype인 Bean을 singleton bean에서 참조해야 할 일이 생겼다.

```java
@Configuration
public UrlBuilderConfig() {    
    @Value("${serv.task.host}")
    private String taskHost;

    @Value("${serv.task.port}")
    private int taskPort;

    @Bean("taskRequestBuilder")
    @Scope("prototype")
    public UriComponentsBuilder taskRequestBuilder() {
        return UriComponentsBuilder.newInstance().scheme("http")
                .host(taskHost)
                .port(taskPort)
                .path("/api");
    }
    ...
}
```

Uri를 빌드하기 위한 UriComponentsBuilder이다. 매번 PathVariable이 섞여있는 uri 빌드를 쉽게 해주기 위한 도구이다.

만약 singleton 빈이었다면 **빌더의 변경내용**이 계속 유지될 것이고, 다음 요청 uri를 빌드할때 의도치않은 동작이 될 것이다. 따라서 prototype 빈으로 설정한다.

```java
@Service
public class MemberService {
    // singleton 빈에서 prototype 빈을 가져올 수 없다.
    // @Autowired
    // @Qualifier("taskRequestBuilder")
    // UriComponentsBuilder builder;

    public void sendCreateRequest(long projectId, String memberId) {
        UriComponents components = builder.getIfAvailable()
            .path("/projects/{projectId}/members")
            .queryParam("memberId", memberId).encode()
            .buildAndExpand(projectId);
        
        ...
    }
    ...
}
```

그러나 singleton 빈에서 prototype 빈을 가져올 수 없다. singleton 빈이 의존성을 주입받을 때는 빈을 생성할 시점인데, 메소드를 호출할 때마다 새로운 빈을 가져오고 싶다.

이때 사용할 수 있는 것이 ObjectProvider 이다.

```java
@Service
public class MemberService {
    @Qualifier("taskRequestBuilder")
    @Autowired
    private ObjectProvider<UriComponentsBuilder> builderProvider;

    public void sendCreateRequest(long projectId, String memberId) {
        UriComponents components = builderProvider.getIfAvailable()
                .path("/projects/{projectId}/members")
                .queryParam("memberId", memberId).encode()
                .buildAndExpand(projectId);
        ...
    }
    ...
}
```

`ObjectProvider<UriComponentsuilder>`를 주입받아 prototype 빈을 매번 새롭게 초기화시켜 가져올 수 있다.
---
title: "Spring Boot에서 파일 로그 설정"
date: 2024-10-26 14:57:00 +0900
categories: ["Spring Boot"]
tags: [Spring Boot, Java, Logback]
---

Spring Boot에서 application.properties 또는 .yml을 수정하여 logging에 관한 설정을 할 수 있다. 파일로 저장하는 `logging.file` 옵션이 존재하는데, 알아야 할 몇가지가 존재한다.

### 1. `logging.file.name`, 그리고 `logging.file.path`

Spring Boot는 로그를 파일로 저장하기 위한 옵션을 제공하는데, 그것들이 `logging.file.name`과 `logging.file.path` 이다.

네이밍만 보면 `logging.file.path`로 경로를 지정하고, `logging.file.name`으로 저장할 로그 파일의 이름을 적는 것처럼 보이지만 실상은 그렇지 않다.

현재 `application.properties`가 이렇게 정의되어 있다고 가정하자.
```properties
logging.file.name=server.log
logging.file.path=/logs
```
얼핏보면 /logs/server.log 에 로그가 저장될 것을 기대할 수 있다.

그러나 `logging.file.name`은 `logging.file.path`를 덮어씌워버리고 Spring boot application을 실행시킨 쉘의 경로에서 server.log를 저장한다.

예컨데 / 경로에서 앱을 실행시켰다면,
```shell
root@server:/$ java -jar /opt/app/server.jar
```
`/server.log`에 저장된다.

만약 `logging.file.path=/logs`만 존재한다면 `/logs/spring.log` 에 저장된다.

본래 목적대로 /logs/server.log 에 저장하고 싶다면 `logging.file.name`에 full path를 적어주면 된다.

```properties
logging.file.name=/logs/server.log
```

---

### 2. `logging.logback.rollingpolicy`이 설정된 경우.

로그를 한 곳에 저장하면 파일이 너무 두꺼워 지므로 관리가 매우 힘들다. 따라서 로그를 나눠서 저장하는 솔루션을 제공하는데 Spring Boot는 기본으로 logback을 사용하므로 `logging.logback.rollingpolicy`에서 설정할 수 있다.

```yaml
logging:
  file.name: /logs/server.log

  logback:
    rollingpolicy:
      max-history: 100 # 로그 분할 최대 개수
      file-name-pattern: server-%i.log  # %i로 인덱스 지정 가능
      max-file-size: 100KB  # 로그가 특정 크기 이상이 되면 분할함.
      clean-history-on-start: false   # 앱 기동시 이전 로그 삭제 (기본 false)
```

`logging.logback.rollingpolicy.file-name-pattern`에서 분할될 로그 파일의 이름을 정해줄 수 있다. 만약 앱을 `/` 에서 실행시켰고 위와 같이 `logging.file.name`과 같이 있다면 어떻게 될까?

![log-with-rollingpolicy](/assets/images/2024-10-26/log1.png)

(사진은 `logging.file.path`가 설정되어 있지만 name과도 동일하다.)

결론적으로 `/logs/server.log`로그가 쌓이다가 100KB를 넘어서면 `/server-0.log`에 분할된다.

같은 곳에 저장하고 싶다면 마찬가지로 `logging.logback.rollingpolicy.file-name-pattern`에 full path를 적어준다.

```properties
logging.logback.rollingpolicy.file-name-pattern=/logs/server-%i.log
```

---

### 3. 날짜별로 로그 저장하기.

`file-name-pattern`에는 인덱스 뿐만 아니라 날짜별로 저장할 수도 있다.

```properties
logging.file.name=/logs/server.log
...
logging.logback.rollingpolicy.file-name-pattern=/logs/server-%d{yyyy-MM-dd}.%i.log
```

이렇게 설정해두면 `/logs/server.log`에 로그가 쌓이다가 날짜가 바뀐 로그가 생기면 이전의 로그를 모두 `/logs/server-%d{yyyy-MM-dd}.%i.log`에 저장하고 현재 날짜의 로그를 `/logs/server.log`에 쌓는다.

또 메세지 포맷에서 시간포맷을 정해준것 처럼 시간도 설정할 수 있다.

```properties
...file-name-pattern=/logs/server-%d{yyyy-MM-dd_HH:mm}.%i.log
```

이렇게 지정한다면 로그가 분이 바뀌었을때 위에서와 마찬가지로 이전 로그를 따로 저장하고 다시 로그를 쌓는다.
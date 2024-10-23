---
title: "Docker를 이용하여 live, dev 서버 이중화 배포 #1"
date: 2024-10-23 09:10:00 +0900
categories: DevOps
tags: [Spring Boot, Java, Docker, DevOps, CI/CD]
---

우리 프로젝트 팀은 `master`, `develop` 두 브랜치를 공용으로 두고 기능별로 브랜치를 분기하여 Develop에 Merge 한 다음 live 서버가 배포되는 Master 브랜치에 Merge 하는 버전관리 전략을 도입했다.

따라서 실제 서비스가 올라가는 live 서버, 개발코드가 올라가는 dev 서버로 나뉘어 운영해야 한다.

이 포스트에서 Github Actions를 이용해 Spring Boot의 프로젝트를 live, dev를 나누어 Docker 컨테이너로 자동으로 배포해 볼 것이다.

먼저 배포가 되는 구조를 생각해보자. 애플리케이션을 배포하고 도커 이미지로 빌드하기 위해서 `.jar` 파일과 `Dockerfile`이 필요하다. 또, 이미지 빌드 명령을 workflow에 기입하기에는 너무 긴 관계로 빌드명령을 할 스크립트 `startup.sh`가 필요하다.

```
front
├── live    # master 브랜치
│   ├── target
│   │   └── front.jar
│   ├── Dockerfile
│   └── startup.sh
│
└── dev     # develop 브랜치
    ├── target
    │   └── front.jar
    ├── Dockerfile
    └── startup.sh
```

이미지를 빌드할 Dockerfile을 정의하자. Temurin JDK 21을 사용할 것이다.

```Dockerfile
FROM eclipse-temurin:21

ENV SPRING_PROFILE="default"  # 스크립트에서 바꿀 값.
ENV SERVER_PORT=8080          # 스크립트에서 바꿀 값.

RUN mkdir /opt/app
COPY target/front.jar /opt/app
CMD ["java", 
     "-Dspring.profiles.active=${SPRING_PROFILE}", 
     "-Dserver.port=${SERVER_PORT}",
     "-jar", "/opt/app/front.jar"]
```

이미지 빌드와 컨테이너 생성 명령을 내릴 스크립트 `startup.sh`를 정의하자.

```shell
#!/bin/bash

ABSOLUTE_PATH="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"
profile=$1
image_name="front-server"

container_name="front-live" # 컨테이너 이름 prefix
spring_env="live"           # 적용시킬 spring profiles
server_port=(8080 8081)     # 서버 이중화
network_bridge="live"       # 연결할 도커 네트워크
container_postfix="live"    # 컨테이너 이름 뒤에 붙일 postfix

if [ "$profile" == "--dev" ]; then    # --dev 옵션이 있다면,
  # dev 서버를 대상으로...
	container_name="front-dev"
	spring_env="dev"
	server_port=(8090)  # dev 서버는 하나만..
	network_bridge="dev"
  container_postfix="dev"
fi

cd $ABSOLUTE_PATH

# 이미 작동중인 컨테이너가 있는지 체크.
docker_ps=$(docker ps --all --filter "name=${container_name}" | awk '{ print $1 }')

# 도커 네트워크 설정..
# 컨테이너 이름을 host로서 다른 컨테이너에 접근하고 싶다면 브릿지 네트워크를 생성해야 한다.
docker_network_live_ps=$(docker network ls | grep 'live')
if [ -z "$docker_network_live_ps" ]; then
  docker network create live
fi

docker_network_dev_ps=$(docker network ls | grep 'dev')
if [ -z "$docker_network_dev_ps" ]; then
  docker network create dev
fi

# 라인을 분리해 배열에 저장.
i=0
for line in $docker_ps; do
  ps_arr[i]=$line
  i=$((i+1))
done

# 이전에 실행되어 있는 컨테이너 제거
for ((i=1; i<${#ps_arr[@]}; i++)); do
    echo "Removing container ${ps_arr[i]}..."
    docker stop ${ps_arr[i]}
    docker rm ${ps_arr[i]}
done

# 새로운 이미지 빌드
docker build -t $image_name-$container_postfix .

# 위에서 설정한 포트 수 만큼 컨테이너 생성
for ((i=0; i<${#server_port[@]}; i++)); do
    docker run -d --name $container_name-$i \ 
        --network $network_bridge \ 
        --env SPRING_PROFILE=$spring_env \ 
        --env SERVER_PORT=${server_port[i]} \ 
        -p ${server_port[i]}:${server_port[i]} \ 
        $image_name-$container_postfix
done

# 이전 컨테이너가 사용했던 구버전 이미지 제거
docker image prune --force
```

계속 빌드는 올라갈 것이기 때문에 이전에 존재했던 컨테이너와 이미지를 제거하는 코드가 있어야 한다. 또, docker network에 관한 명령어도 존재하는데, 네트워크 내의 IP를 알지 못해도 도커 컨테이너 이름을 호스트로서 접근할 수 있게 해준다.

예를 들어 live 브릿지 네트워크 내에서 front와 eureka 컨테이너가 존재한다고 가정했을때, front 컨테이너에서 eureka 컨테이너에게 다음과 같은 요청을 할 수 있다.

```shell
$ curl http://eureka
```

도커 컨테이너 내부에서 호스트로 직접 접근할 수 있게 해주는 `host.docker.internal`와 비슷한 맥락이다.

`Dockerfile`과 `startup.sh`정의가 끝났으니 브랜치에 푸쉬해주면 사전 작업은 끝난다.

다음에는 Github Actions workflow를 수정하여 master와 develop의 행동을 구분해보자.

[다음](/devops/Docker를-이용하여-live-dev-서버-이중화-배포-2)
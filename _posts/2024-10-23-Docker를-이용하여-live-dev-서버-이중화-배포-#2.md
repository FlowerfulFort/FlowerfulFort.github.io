---
title: "Docker를 이용하여 live, dev 서버 이중화 배포 #2"
date: 2024-10-23 17:21:00 +0900
categories: DevOps
tags: [Spring Boot, Java, Docker, DevOps, CI/CD]
---

[여기](/devops/Docker를-이용하여-live-dev-서버-이중화-배포-1)에서 계속된다.

[이전](/devops/Github-Actions를-이용하여-Spring-Boot-배포-자동화-환경-구축하기/)에서 이미 `master` 브랜치에 대응하는 자동 배포 workflow를 만들었다. 이를 조금 더 확장해 볼 것이다.

먼저, develop 브랜치에도 대응이 되도록 push와 pull_request 모두에 `develop` 브랜치를 넣는다.

```yaml
on:
  push:
    branches: [ "master", "develop" ]
  pull_request:
    branches: [ "master", "develop" ]
```

브랜치별로 다른 행동을 주기 위해서는 해당 step에 다음 항목을 추가해야 한다.

```yaml
if: github.ref == 'refs/heads/branch-name'
```

이를 반영하여 scp-action과 ssh-action을 추가해주자. 기존에 있었던 action 들은 master로 조건을 주었다.

{% raw %}
```yaml
  steps:
...

    - name: upload file develop
      uses: appleboy/scp-action@master
      # 조건: 현재 브랜치가 develop
      if: github.ref == 'refs/heads/develop' 
      with:
        host: ${{ secrets.SSH_IP }}
        username: ${{ secrets.SSH_ID }}
        key: ${{ secrets.SSH_KEY }}
        port: ${{ secrets.SSH_PORT }}

        # jar, Dockerfile, startup.sh 업로드
        source: "target/*.jar,Dockerfile,startup.sh"  
        target: "~/app/front/dev/"
        rm: false

    - name: execute shell script develop
      uses: appleboy/ssh-action@master
      if: github.ref == 'refs/heads/develop'
      with:
        host: ${{ secrets.SSH_IP }}
        username: ${{ secrets.SSH_ID }}
        key: ${{ secrets.SSH_KEY }}
        port: ${{ secrets.SSH_PORT }}
        script_stop: true
        script: "~/app/front/dev/startup.sh --dev"
```
{% endraw %}

master와 develop에 대한 action을 모두 정의하였으면 커밋하고 실제로 작동이 되는지 Actions 탭에서 확인할 수 있다.

![maven-action](/assets/images/2024-10-22/maven-action.png)

master 브랜치의 Action이기 대문에 develop에 할당한 action들은 모두 비활성화 되었음을 알 수 있다.

이제 클라우드 서버에서 도커가 잘 빌드되었는지 확인하면 된다.

```shell
$ docker ps
```

![docker-ps](/assets/images/2024-10-22/docker-ps.jpg)

master, develop 별로 컨테이너와 이미지가 만들어졌고, 로드밸런싱을 위해 live 서버에 포트를 두개 할당한 것도 각각 컨테이너로 나뉘어 작동하게 되었다.
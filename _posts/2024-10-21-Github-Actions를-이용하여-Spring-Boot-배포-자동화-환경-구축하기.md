---
title: "Github Actions를 이용하여 Spring Boot 배포 자동화 환경 구축하기"
date: 2024-10-21 21:17:00 +0900
categories: DevOps
tags: [Spring Boot, Java, Github Actions, DevOps, Cloud, CI/CD]
---

애플리케이션의 코드가 업데이트 될때마다 매번 패키징하여 서버로 배포해야한다. 그러나 프로젝트에 관여하는 개발자가 많아지고 코드가 업데이트되는─*PR이 진행되는*─ 주기가 짧아진다면 반복되는 배포에 들이는 시간이 개발속도를 발목잡을 것이다.

설령 이를 스크립트로 만들어 놓았다고 해도 해당 스크립트를 매번 실행시키는 것도 골치아픈 일. Jenkins나 Azure DevOps 등은 이런 문제를 해결하는 솔루션을 제공하고 이러한 *지속적인 코드의 통합과 배포*를 통칭 **CI/CD** 라고 한다.

이번엔 **Github Actions**를 이용해 Spring Boot 프로젝트의 브랜치가 업데이트 될때마다 자동으로 빌드 및 배포를 진행하도록 환경을 만들어 볼 것이다.

배포 자동화를 원하는 레포지터리에서 Actions탭을 눌러주자.

![github-actions](/assets/images/2024-10-21/github-actions.png)

workflow를 고르라고 한다. 빌드 도구로 Maven을 사용하고 있으니 **Java With Maven**을 눌러주자.

눌러주면 .github/workflows/maven.yml 파일을 생성하는 커밋이 등장한다.

```yaml
name: Java CI with Maven

on:
# 해당 브랜치가 push 될 때 실행됨
  push:
    branches: ["master"]
# 해당 브랜치의 pull request merge가 완료될 때 실행됨
  pull_request:
    branches: ["master"]

jobs:
  build:
    runs-on: ubuntu-latest

  steps:
    - uses: actions/checkout@v4
    - name: Set up JDK 17
        # 자바 설정
      uses: actions/setup-java@v4
      with:
        java-version: "17"
        distribution: "temurin"
        cache: maven
    # Maven 빌드 명령
    - name: Build with Maven
      run: mvn -B package --file pom.xml
```

여기서 감시할 브랜치와 JDK 버전 등을 설정할 수 있다. 적당히 develop 브랜치도 감시목록이 포함한다고 하면 아래와 같을 것이다.

```yaml
on:
  push:
    branches: ["master", "develop"]
  pull_request:
    branches: ["master", "develop"]
```

우리가 원하는 것은 빌드에 한정된 것이 아닌, 서버로 배포까지가 목적이다. 배포를 위한 step을 만들기 위해서 Actions 마켓플레이스에 등록된 applyboy/scp-action, appleboy/ssh-action을 사용한다.

{% raw %}
```yaml
  steps:

...

  - name: upload file with scp
    uses: appleboy/scp-action@master
    with:
      host: ${{ secrets.SSH_IP }}
      username: ${{ secrets.SSH_ID }}
      key: ${{ secrets.SSH_KEY }}
      port: ${{ secrets.SSH_PORT }}
      source: "target/*.jar"  # 빌드된 target/*.jar 선택.
      target: "~/"            # ssh로 연결된 대상의 home 디렉토리에 저장.
      rm: false

  - name: execute command
    uses: appleboy/ssh-action@master
    with:
      host: ${{ secrets.SSH_IP }}
      username: ${{ secrets.SSH_ID }}
      key: ${{ secrets.SSH_KEY }}
      port: ${{ secrets.SSH_PORT }}
      script_stop: true
      script: "./startup.sh"  # ssh에서 실행할 명령.
```
{% endraw %}

SSH에 필요한 값들은 [여기](/devops/Github-Actions를-위한-사전작업-Actions-secret/)에서 미리 할당한 키값들을 넣는다.

서버에는 배포된 애플리케이션을 실행하는 스크립트를 startup.sh로 저장해 놓았다.
```shell
#!/bin/bash
target_pid=$(ps -ef | grep "/target/app.jar" | grep -v "grep" | awk '{ print $2 }')

if [ -z "$target_pid" ]; then
	echo "No process"
else
	for pid in $target_pid
	do
		echo "Killing process $pid..."
		kill -9 $pid
	done
	echo "Killing process done"
fi

java -jar ~/target/app.jar > log 2>&1 &
```
브랜치에 커밋하고(감시목록에 있는 브랜치) Actions에서 빌드 로그를 확인하면 된다.

![result](/assets/images/2024-10-21/actions-result.png)

모든 절차가 제대로 진행되었음을 알 수 있다. 서버에도 제대로 애플리케이션이 배포되었는지 ps -ef 등으로 확인하자.

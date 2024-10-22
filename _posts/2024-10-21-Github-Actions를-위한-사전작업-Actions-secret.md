---
title: "Github Actions를 위한 사전작업: Actions secret"
date: 2024-10-21 10:42:23 +0900
categories: DevOps
tags: [Github Actions, DevOps, CI/CD]
---

Github Actions는 레포지터리에 변화가 생길때 마다 자동으로 빌드, 배포를 하는 도구를 제공한다. 빌드된 애플리케이션을 배포하기 위해서는 대상이 되는 서버로 접속이 되어야 하는데, 가장 많이 쓰이는 것 중 하나는 **SSH** 연결이다.

SSH는 서버의 계정 비밀번호를 알고있거나, 서버 내 RSA 공개키에 대응되는 개인키를 갖고 있어야 인증을 할 수 있다. Github Actions 또한 예외가 될 수 없으므로 직접 key를 미리 저장해야한다.

Actions workflow를 생성하기를 원하는 레포지터리에서 Settings 탭으로 가자.

![secrets](/assets/images/2024-10-21/secrets.png)

Security 항목 하위에 **Secrets and variables** 항목이 존재한다. Actions를 누르자.

![secrets-info](/assets/images/2024-10-21/secrets-info.png)

아무것도 저장한 것이 없으니 처음엔 비어있다.

**New repository secret** 을 눌러 새롭게 secret을 생성하자.

![add-secret](/assets/images/2024-10-21/add-secret.png)

Name에 변수로서 접근할 이름, Secret에 value를 넣는다. 예를 들면 Name에 SSH_PORT, Secret에 22를 넣는다면 Actions에서 secrets.SSH_PORT 로 접근하여 22라는 값을 가져올 수 있다. SSH 로그인을 위한 개인키도 저장하자.

![secrets-list](/assets/images/2024-10-21/secrets-list.png)

배포에 필요하지만 감춰야 할 것들을 모두 넣었다.

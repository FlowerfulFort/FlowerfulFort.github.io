---
title: "jekyll을 이용한 Github Pages 생성 기록"
date: 2024-10-19 21:50:59 +0900
categories: jekyll
---

jekyll을 이용하여 블로그를 생성하고 싶다면 먼저 ruby 환경이 필요하다. 분명 jekyll 사이트에서는 준비에서 실행까지 몇초만에 끝난다고 하지만 ruby 환경이 없다면 전혀 그렇지 않다...

    본 문서는 Windows 기준으로 진행한다.

[RubyInstaller for Windows](https://rubyinstaller.org/downloads/)에서 Devkit을 포함한 Ruby installer를 다운로드 받는다.

![rubyinstaller](/assets/images/2024-10-19/ruby-installer.png)

만약 MSYS2 환경이 이미 설치되어 있다면 하단의 MSYS2 툴체인 설치를 하지 않아도 된다. ruby 디렉토리 하단에 새로 MSYS2 환경이 설치되어 번거로워진다.

![ridk-version](/assets/images/2024-10-19/ridk-version.png)

제대로 설치되었다면 ridk version 명령어로 Ruby 환경과 MSYS2 환경이 제대로 잡혀있을 것이다. 툴체인을 설치하지 않아도 MSYS2 환경만 있다면 path가 잡힌다.

만약 이전에 MSYS2 환경이 이미 설치되어 있어서 따로 툴체인을 설치하지 않았다면,

```shell
$ ridk install
```

명령을 입력하여 ruby installer를 실행하여 MINGW development toolchain을 설치해주면 된다.

![ridk-install](/assets/images/2024-10-19/ridk-install.png)

MSYS2가 없다면 그냥 엔터, MSYS2가 있다면 3번만 누르면 된다.

여기까지 잘 진행되었다면 **gem**으로 jekyll과 bundler를 설치해주면 된다.

```shell
$ gem install jekyll bundler
```

만약 설치도중 오류가 났다면 MSYS2 환경이나 mingw 툴체인이 없기 때문이므로 제대로 설치해주면 문제없이 진행된다.

이제 적절한 경로에서 jekyll을 이용해 블로그를 만들 수 있다.

```shell
$ jekyll new [blog-path]        # 블로그 생성
$ cd [blog-path]                # 디렉토리 이동
$ bundle exec jekyll serve      # 블로그 서빙
```

별다른 설정을 하지 않았다면 4000번 포트에 서빙될 것이다. localhost:4000으로 접속하면...

![initial-jekyll](/assets/images/2024-10-19/initial-jekyll.png)

간단? 하게 블로그가 생성되었다.

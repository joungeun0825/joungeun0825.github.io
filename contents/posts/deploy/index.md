---
title: "Google Cloud Platform에서 Jenkins 도커 컨테이너 실행하기"
description: "gatsby-starter-hoodie 에 대해 배워봅시다."
date: 2024-09-28
update: 2021-04-01
tags:
  - CI/CD
  - docker
series: "WhereWear 프로젝트 CI/CD 구축기"
---
이번 글에서는 Docker와 Jenkins를 사용해 배포한 과정에 대해서 이야기해보려고 합니다. 

## Jenkins 없이 배포했다면 어땠을까?
일단 주요 배포과정을 단순하게 아래 3단계로 나누어 생각해 보겠습니다.

먼저 GCP 인스턴스에서 **프로젝트 github repository를 clone** 하여야 합니다.
```bash
git clone https://github.com/where-wear/where-wear-backend.git
```
다음으로는 **프로젝트를 빌드**해야 합니다. 프로젝트 빌드는 Gradle Wrapper로 할 수 있습니다. Gradle Wrapper를 실행하는 명령어인 gradlew에 실행 권한을 부여하고, 빌드 합니다.
```bash
chmod +x gradlew
./gradlew bootJar --debug
```
그러면 Java 애플리케이션, 라이브러리 및 관련 리소스가 압축된 **JAR파일이 생성**됩니다. 이 JAR파일을 실행하여 **애플리케이션을 실행**시킵니다.
```bash
java -jar wherewear-0.0.1-SNAPSHOT.jar
```
성공적으로 배포가 완료 되었습니다. 하지만, 이후에 프로젝트 코드가 수정되고 메인 브랜치에 병합된다면, 위 과정을 또 반복해야 할겁니다. 이런 **수고스러운 과정을 없애기 위해서 Jenkins를 사용하여 CI(지속적 통합)/CD(지속적 배포) 과정을 자동화** 해보겠습니다.

## Jenkins를 사용한 CI/CD 자동화
이제 CI/CD 과정을 자동화 해보겠습니다. 먼저 GCP 인스턴스에 **Jenkins를 설치**해야 합니다. Jenkins는 스프링부트 애플리케이션과 격리된 환경에서 실행하기 위해서 GCP 인스턴스에 Jenkins를 직접 설치하지 않고, **Jenkins 도커 컨테이너를 만드는 방법**을 사용하겠습니다. 그러면 결국 **GCP 인스턴스에는 Jenkins 컨테이너와 애플리케이션 컨테이너 두 대가 띄워지게 됩니다.**

생각해봐야 할 부분이 있습니다. Jenkins가 프로젝트를 빌드하고 애플리케이션 컨테이너를 띄우게 되는데, 그러면 Jenkins가 Docker를 사용하려면 Jenkins 안에 Docker를 설치해야 할까요? 하지만 이 방법(Docker-in-Docker)을 사용한다면 컨테이너 내에서 루트 권한으로 실행되기 때문에 보안 리스크가 증가 합니다. 그래서 그 대신, Jenkins가 호스트의 Docker를 제어할 수 있는 방법을 사용할 것입니다. 따라서, GCP 인스턴스 소켓을 Jenkins 컨테이너 안에 마운트 하여 Docker-out-Docker 방법을 사용하도록 하겠습니다.

### Docker 설치
도커 컨테이너를 만들기 위해서는 GCP 인스턴스에 Docker를 설치해야합니다. 아래 방법으로 Docker를 설치할 수 있습니다.

**패키지 업데이트**<br>
아래 명령어로 Docker의 GPG 키를 추가하고 Docker 패키지를 설치하는 데 필요한 환경을 설정합니다.
```bash
$ sudo apt-get update
$ sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release
```

**Docker의 공식 GPG 키 추가**<br>
```bash
$ sudo mkdir -p /etc/apt/keyrings
$ curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
```

**Docker 저장소 설정**<br>
```bash
$ echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

**Docker 엔진 설치**<br>
```bash
$ sudo apt-get update
$ sudo apt-get install docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

**Docker 버전 확인**<br>
Docker가 정상적으로 설치되었는지 버전을 확인합니다.
```bash
$ docker --version​
```

### Jenkins 컨테이너 실행하기

**Jenkins 이미지 다운로드**<br>
아래 명령어로 Jenkins 이미지를 다운 받습니다.
```bash
$ docker pull jenkins/jenkins:lts
```

**Docker 소켓 마운트 하여 Jenkins 컨테이너 실행**<br>
Jenkins가 GCP 인스턴스의 Docker를 사용할 수 있도록 Docker 소켓을 마운트하여 Jenkins 컨테이너를 실행합니다.
```bash
$ docker run -d \
  --name my-jenkins \
  -p 8080:8080 \
  -v jenkins_home:/var/jenkins_home \  # Jenkins 데이터 저장을 위한 볼륨
  -v /var/run/docker.sock:/var/run/docker.sock \  # Docker 소켓 마운트
  jenkins/jenkins:lts  # Jenkins 이미지 사용
```

## 끝으로
이렇게 Google Cloud Platform 인스턴스에 Jenkins 컨테이너 실행을 성공시켰습니다. 다음 포스팅에서는 Jenkins를 세팅하고, 파이프라인을 구성해보도록 하겠습니다.
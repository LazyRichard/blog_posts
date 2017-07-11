---
title:  "Docker를 이용한 Redmine 설치"
categories: 
  - Linux
tags:
  - Docker
  - Redmine
---

제 도메인 상에 있는 모든 서비스들을 도커 위에서 구동하려고 합니다. 이 첫 번째 단계로 레드마인을 도커 위에서 서비스 하기 위한 과정들을 정리해보았습니다.

호스트 리눅스 서버는 Centos 7 1611버전을 기준으로 합니다.

## Docker 소개

도커는 Hyper-V, VMWare, VirtualBox와 비슷한 가상화 프로그램입니다. 하지만 이 프로그램들은 OS 전체를 가상화 하는 방식으로 호스트 OS위에 게스트 OS가 올라가고 그 위에 어플리케이션이 동작하는 방식입니다.

하지만 Docker의 경우에는 조금 다릅니다. Docker는 OS를 가상화하는 방식이 아닌 어플리케이션 레이어와 디펜던시를 가상화한 방식으로 각각의 프로세스들은 격리된 환경에서 실행합니다.

### 도커 설치

도커가 Community Edition과 Enterprise Edition으로 나뉘면서 그랬는지는 몰라도 설치 방법도 조금 바뀌었습니다.

먼저 필요 패키지들을 설치해 줍니다.
```
# yum install -y yum-utils device-mapper-persistent-data lvm2
```

그 다음에는 docker-ce stable 레파지토리를 추가합니다.
```
# yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
```

이제 docker를 설치합니다.
```
# yum install -y docker-ce
```

## Redmine 소개

레드마인은 웹 기반의 프로젝트 관리 도구 입니다. 루비 온 레일즈 프레임워크를 이용해 작성되었으며, 크로스 플렛폼 및 크로스 데이터베이스를 지원합니다.

레드마인에서 지원하는 기능들은 다음과 같습니다.

* 다중 프로젝트 지원
* 유연한 역할 기반 접근 관리
* 유연한 이슈 관리 시스템
* 간트 차트 및 달력
* 뉴스, 문서 및 파일 관리
* 피드 및 이메일 알림
* 프로젝트 별 위키 및 포럼
* 이슈, 시간 엔트리, 프로젝트와 사용자에 대한 사용자 지정 필드
* SCM 통합(SVN, CVS, Git, Mercurial, Bazaar 및 Darcs)
* 이메일을 통한 이슈 작성
* 다중 LDAP 인증 지원

## 레드마인 설치

도커를 이용하여 레드마인을 설치하는 것은 명령어 단 두 줄로 충분합니다.
```
# docker run -d --name db_redmine -e MYSQL_ROOT_PASSWORD=example -e MYSQL_DATABASE=redmine mysql

# docker run -d --name site_redmine --link db_redmine:mysql redmine
```
하지만 플러그인을 설치하거나 그럴 경우 어플리케이션을 다시 시작해야 하는데, 그 때마다 긴긴 명령어를 다 치기에는 부담스럽습니다.

그래서 이런 복잡한 것들을 미리 입력해 놓고 한 번에 처리할 수 있도록 Docker-Compose를 사용하도록 하겠습니다.

## Docker-Compose 설치

### Centos 7 w Python-pip

Centos 7 에서 Python-pip를 사용하기 위해서는 먼저 epel 레파지토리를 추가해야 합니다.

```
# yum install -y epel-release
```

그 다음 Python-pip를 설치하고 업데이트를 해줍니다.

```
# yum install -y python-pip && pip install pip --upgrade
```

이제 docker-compose를 설치합니다.

```
# pip install docker-compose
```

설치가 완료된 후에 정상적으로 설치되었는지 확인하기 위해서는 다음과 같은 명령어를 입력합니다.

```
docker-compose --version
```
아래와 비슷하게 뜨면 정상적으로 설치 된 것입니다.

```
docker-compose version 1.13.0, built 1719ceb
docker-py version: 2.3.0
CPython version: 2.7.5
OpenSSL version: OpenSSL 1.0.1e-fips 11 Feb 2013
```

## docker-compose.yml 파일 만들기

docker-compose를 설치했으면 이제 귀찮은 작업들을 대신해줄 명령서를 발행해야 합니다. docker-compose.yml에 docker-compose에서 지원하는 명령들을 넣어 놓으면 docker-compose up 명령 한줄로 많은 명령들을 처리할 수 있습니다.

아래 파일에서 ```/path/to/<DIR>``` 같은 경우에는 실제 데이터가 저장될 폴더를 의미하니 각자 데이터를 저장하고 싶은 폴더 주소를 입력해 주세요.

제가 원하는 것은 원본 파일은 최대한 수정하지 않는 것이라 플러그인 폴더와 테마 폴더를 마운트 했습니다. 하지만 github에 issue로 등록된 것을 읽어 보니 plugins 디렉터리는 마운트하는 것을 추천하지 않는다더라구요. plugins 디렉터리를 마운트하지 않고 플러그인을 설치하는 방법은 모르겠네요;;

그리고 네트워크에 default와 redmine_network 이렇게 2개를 만든 이유는 db와 망 분리를 위해서인데 제대로 한 건지는 잘 모르겠습니다.
```
version "3.2"

services:
  site_redmine:
    image: redmine:latest
    container_name: site_redmine
    networks:
      - default
      - redmine_network
    links:
      - redmine_db:mysql
    volumes:
      - /path/to/plugins:/usr/src/redmine/plugins
      - /path/to/themes:/usr/src/redmine/public/themes
      - /path/to/files:/usr/src/redmine/files
    ports:
      - "80:3000"
    environment:
      REDMINE_DB_MYSQL: "redmine_db"
      REDMINE_DB_PASSWORD: "example"
      REDMINE_PLUGINS_MIGRATE: "true"

  redmine_db:
    image: mariadb:latest
    container_name: db_redmine
    networks:
      - redmine_network
    volumes:
      - /path/to/db:/var/lib/mysql
    environment:
      MYSQL_ROOT_PASSWORD: "example"
      MYSQL_DATABASE: "redmine"
    command:
      [ "--character-set-server=utf8",
        "--collation-server=utf8_unicode_ci" ]
        
networks:
  redmine_network:
```

docker-compose.yml 파일을 만든 후에 이 파일을 실행하기 위해서는 다음과 같은 명령어를 입력합니다.

```
# docker-compose up
```

docker-compose로 올린 서비스들을 모두 내리려면 다음과 같은 명령어를 입력합니다.

```
# docker-compose down 
```

## 테마 설치

테마 설치는 간단합니다. ```/path/to/themes``` 폴더에 다운로드 받은 테마 파일을 풀어 놓습니다.

그 후, ```docker-compose down && docker-compose up``` 하시면 됩니다.

## 플러그인 설치

플러그인 설치는 경우에 따라서는 살짝 까다로울 수도 있습니다. 대부분의 경우에는 그냥 테마와 마찬가지로 ```/path/to/plugins``` 폴더에 다운로드 받은 플러그인 압축을 풀고, 어플리케이션을 재기동하면 완료됩니다.

하지만 이렇게 했음에도 불구하고 internal error를 띄우는 경우가 있습니다.

이 경우에는 먼저 설치한 플러그인이 현재 기동중이 Redmine을 지원하는지 확인하셔야 합니다.

대표적인 예로 Clipboard 플러그인의 경우에는 Redmine 3.x를 지원하지 않아 다른 사람이 fork 한 저장소에서 설치했습니다.

그 다음에는 DB가 제대로 마이그레이션 되지 않은 경우입니다.

이 경우에는 다음과 같은 명령어를 입력하고, 브라우져를 리프레시 하면 정상적으로 작동합니다.

```
# docker exec site_redmine rake
```
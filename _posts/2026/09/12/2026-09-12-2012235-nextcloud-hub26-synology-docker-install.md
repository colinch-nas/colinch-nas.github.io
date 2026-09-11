---
layout: post
title: "Nextcloud Hub 26 Spring 설치 - Synology DSM 7.4 Docker로 내 클라우드 만들기"
description: "Nextcloud Hub 26 Spring을 Synology DSM 7.4 Container Manager에 설치하고 MariaDB·Redis·HTTPS까지 연결하는 실전 설정 가이드다."
date: 2026-09-12
tags: [Nextcloud, Synology, Docker, DSM, 자체호스팅, NAS보안]
comments: true
share: true
---

![Nextcloud Hub 26 Spring Synology NAS 사설 클라우드 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

구글 드라이브처럼 파일을 동기화하면서 데이터는 집에 두고 싶다면 Synology NAS에 Nextcloud를 올리는 구성이 현실적이다. 2026년 6월 공개된 Nextcloud Hub 26 Spring과 8월 유지보수 업데이트 흐름을 기준으로, DSM 7.4의 Container Manager에서 웹·DB·Redis를 함께 구성했다. 단순히 컨테이너 하나만 실행하는 방식보다 업데이트와 장애 대응이 수월하다.

## 이번 구성

| 항목 | 설정값 |
|---|---|
| NAS | Synology DS923+ (메모리 8GB) |
| OS | DSM 7.4, Container Manager |
| 서비스 | Nextcloud Hub 26 Spring, MariaDB 11, Redis 7 |
| 내부 주소 | NAS `192.168.0.20`, 웹 포트 `8088` |
| 외부 주소 | `cloud.example.com`을 기존 역방향 프록시로 연결 |

Nextcloud 공식 Docker 예제도 데이터베이스와 애플리케이션을 Compose로 묶는 구성을 안내한다. 다만 예제의 기본 비밀번호를 그대로 쓰면 안 된다. 아래 값은 예시이므로 실제 설치 때 모두 바꾼다.

## 폴더와 Compose 파일 만들기

Container Manager의 프로젝트 폴더를 `/volume1/docker/nextcloud`로 만들고, 그 안에 `compose.yaml`을 저장한다. 사진·문서가 쌓이는 `data` 폴더는 DB와 분리해 스냅샷 대상도 구분했다.

```yaml
services:
  db:
    image: mariadb:11
    container_name: nextcloud-db
    restart: unless-stopped
    command: --transaction-isolation=READ-COMMITTED --binlog-format=ROW
    environment:
      MYSQL_ROOT_PASSWORD: "교체할-긴-루트비밀번호"
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud
      MYSQL_PASSWORD: "교체할-긴-DB비밀번호"
    volumes:
      - ./db:/var/lib/mysql

  redis:
    image: redis:7-alpine
    container_name: nextcloud-redis
    restart: unless-stopped
    command: redis-server --requirepass 교체할-Redis비밀번호

  app:
    image: nextcloud:apache
    container_name: nextcloud-app
    restart: unless-stopped
    ports:
      - "8088:80"
    depends_on:
      - db
      - redis
    environment:
      MYSQL_HOST: db
      MYSQL_DATABASE: nextcloud
      MYSQL_USER: nextcloud
      MYSQL_PASSWORD: "교체할-긴-DB비밀번호"
      REDIS_HOST: redis
      REDIS_HOST_PASSWORD: 교체할-Redis비밀번호
    volumes:
      - ./html:/var/www/html
      - ./data:/var/www/html/data
```

이 파일에서 NAS의 8088 포트만 외부에 열어 둔 것은 의도적이다. 공유기에서 8088을 포트포워딩하지 않고, DSM 제어판의 로그인 포털 > 고급 > 역방향 프록시에서 `cloud.example.com`을 `http://127.0.0.1:8088`로 전달한다. 인증서는 기존 글에서 만든 Let’s Encrypt 인증서를 선택한다.

프로젝트 화면에서 Compose 파일을 선택하고 배포한다. 브라우저에서 `http://NAS주소:8088`을 열어 관리자 계정을 만들고, 데이터베이스는 다음처럼 입력한다.

| 설치 화면 | 입력값 |
|---|---|
| 데이터베이스 사용자 | `nextcloud` |
| 데이터베이스 비밀번호 | Compose의 `MYSQL_PASSWORD` 값 |
| 데이터베이스 이름 | `nextcloud` |
| 데이터베이스 호스트 | `db` |

`localhost`를 DB 호스트로 입력하면 실패한다. Nextcloud 컨테이너에서 `db`는 Compose 내부 DNS가 MariaDB 컨테이너로 연결해 주는 서비스 이름이다.

## 설치 후 꼭 바꿀 설정

관리자 화면의 설정 > 기본 설정에서 백그라운드 작업을 AJAX에서 Cron으로 바꾼다. 컨테이너 안에서 5분마다 실행되도록 DSM 작업 스케줄러에 아래 명령을 등록했다.

```bash
docker exec --user www-data nextcloud-app php -f /var/www/html/cron.php
```

설정 > 관리 > 개요에서 `cloud.example.com`을 신뢰할 수 있는 도메인으로 확인하고, 보안 경고가 남으면 역방향 프록시의 HTTPS 헤더 설정을 점검한다. 외부 접속은 반드시 HTTPS로만 사용한다. 관리자 계정에는 별도 2단계 인증을 켜고, 사진 원본과 DB 폴더는 Hyper Backup 또는 다른 장치로 다시 백업한다. RAID는 백업이 아니다.

## 실제로 걸렸던 문제

처음에는 `nextcloud:latest`를 사용했는데 업데이트 시점에 PHP 버전과 앱 호환성을 한꺼번에 확인해야 했다. 이미지 태그를 무조건 최신으로 당기기보다, 업데이트 전 Hyper Backup과 DB 덤프를 만들고 유지보수 공지 확인 후 배포하는 편이 안전하다. 외부에서 업로드가 멈추면 역방향 프록시의 요청 본문 크기와 타임아웃도 확인해야 한다. 2GB 동영상 하나로 문제가 드러나는 경우가 많다.

Nextcloud Hub 26 Spring의 변경점과 유지보수 공지는 공식 릴리스 페이지에서 확인할 수 있다. 설치 시점의 지원 버전과 보안 패치를 확인하고 진행하는 것이 좋다.

핵심만 정리하면, `app·db·redis`를 같은 Compose 프로젝트로 묶고 데이터 폴더를 별도 백업한다. 8088은 내부에서만 쓰고 HTTPS 역방향 프록시로 공개한다. 업데이트 전에는 반드시 복구 가능한 백업을 만든다.

참고: [Nextcloud Hub 26 Spring 공식 발표](https://nextcloud.com/blog/nextcloud-hub26-spring/), [Nextcloud 유지보수 업데이트](https://nextcloud.com/blog/category/release/), [Nextcloud Docker 공식 저장소](https://github.com/nextcloud/docker)

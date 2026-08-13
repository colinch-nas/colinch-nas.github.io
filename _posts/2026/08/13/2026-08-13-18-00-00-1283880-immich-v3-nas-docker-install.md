---
layout: post
title: "Immich v3.0 NAS 설치 — Synology Docker로 구글 포토 대체 서버 만들기"
description: "2026년 7월 공개된 Immich v3.0을 Synology DSM 7.4 Container Manager에 설치하는 방법이다. 사진 저장 경로, Docker Compose, 모바일 백업과 업그레이드 주의점을 실제 설정 기준으로 정리했다."
date: 2026-08-13
tags: [Immich, Synology, Docker, 자체호스팅, NAS설정]
comments: true
share: true
---

![Synology NAS에 Immich 사진 서버 구축](https://images.unsplash.com/photo-1618005198919-d3d4b5a92ead?w=1200&q=80)

Immich v3.0을 Synology NAS의 Container Manager에 올리면 사진과 동영상을 내 서버에 보관하면서 휴대폰 자동 백업까지 구성할 수 있다. 2026년 7월 공개된 v3.0은 모바일 비파괴 편집, 백그라운드 백업 개선, 무결성 검사, Workflows 미리보기 등이 추가됐다. 다만 메이저 업데이트라 API 변경이 포함됐으므로 기존 v2 사용자는 백업 후 올려야 한다.

## 이번 구성

DS923+에 DSM 7.4, Container Manager를 설치한 상태에서 사진은 `/volume1/photo/immich`, 설정 파일은 `/volume1/docker/immich`에 둔다. 사진 폴더와 데이터베이스를 같은 곳에 몰아넣지 않는 게 복구할 때 덜 헷갈린다.

| 항목 | 설정값 |
|---|---|
| NAS | Synology DS923+ |
| OS | DSM 7.4.x |
| Immich | v3 계열 |
| 접속 | NAS 내부 `2283` 포트, 외부는 기존 HTTPS 역방향 프록시 |
| 저장 위치 | `/volume1/photo/immich` |

이 그림에서 볼 것은 사진 원본은 NAS 볼륨에 남고, Immich 서버·Redis·PostgreSQL이 Docker 네트워크 안에서 함께 동작한다는 점이다.

## Docker Compose 준비

Immich 공식 문서의 Docker Compose 파일을 내려받아 프로젝트 폴더에 저장한다. Container Manager에서 프로젝트를 만들 때도 같은 Compose 내용을 붙여 넣으면 된다. 아래처럼 저장 경로만 NAS 환경에 맞춘다.

```bash
mkdir -p /volume1/docker/immich
cd /volume1/docker/immich
curl -L https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml -o docker-compose.yml
curl -L https://github.com/immich-app/immich/releases/latest/download/example.env -o .env
```

`.env`에서 핵심 값은 세 가지다. 비밀번호에 특수문자를 많이 넣으면 Compose 해석에서 삽질할 수 있어 영문·숫자 조합으로 시작하는 편이 안전하다.

```env
UPLOAD_LOCATION=/volume1/photo/immich
DB_DATA_LOCATION=/volume1/docker/immich/postgres
IMMICH_VERSION=v3
DB_PASSWORD=긴영문숫자비밀번호
DB_USERNAME=postgres
DB_DATABASE_NAME=immich
```

Synology 공유 폴더 권한에서 `docker` 공유 폴더와 `photo` 공유 폴더에 컨테이너가 접근할 수 있어야 한다. 권한 오류가 나면 사진 폴더 전체를 무작정 `777`로 바꾸지 말고 Container Manager 프로젝트의 볼륨 매핑과 공유 폴더 권한을 함께 확인한다.

## 컨테이너 실행과 첫 접속

구성 파일 검사를 거친 뒤 백그라운드로 올린다. 이 명령은 서버, 머신러닝, Redis, 데이터베이스 컨테이너를 한 번에 실행한다.

```bash
docker compose config
docker compose pull
docker compose up -d
docker compose ps
```

브라우저에서 `http://NAS_IP:2283`으로 접속해 관리자 계정을 만든다. 외부 공개가 필요하면 공유기에서 2283 포트를 직접 열지 않고, DSM 역방향 프록시에서 `photos.example.com`을 NAS의 2283으로 전달한다. HTTPS 인증서는 앞서 발급한 Let's Encrypt 인증서를 연결한다.

## 휴대폰 백업 설정

Immich 모바일 앱에 같은 서버 주소로 로그인한 뒤 **설정 → 백업**에서 카메라 폴더를 선택한다. Wi-Fi에서만 업로드하도록 제한하고, 기존 사진이 많으면 충전 중에 나눠 올린다. 수만 장을 한 번에 올리면 NAS의 썸네일·얼굴 인식 작업이 몰려 DSM 응답이 느려졌다.

v3.0에서는 Android 백그라운드 백업이 주기 작업 기반으로 개선됐지만, 배터리 최적화가 켜져 있으면 업로드가 멈출 수 있다. 앱 알림 허용과 배터리 최적화 제외를 같이 적용해야 한다.

## 업데이트 전 체크리스트

| 확인 항목 | 이유 |
|---|---|
| PostgreSQL 백업 | 사진 원본과 앨범·얼굴 정보는 별개라 DB가 필요하다 |
| `.env` 복사 | 경로와 비밀번호 유실 방지 |
| 외부 API 사용 여부 | v3에서 일부 API가 변경됐다 |
| 디스크 여유 공간 | 마이그레이션과 썸네일 생성 공간 필요 |

공식 v3 업데이트 방식은 `.env`의 `IMMICH_VERSION`을 `v3`로 바꾼 뒤 아래 명령을 실행하는 것이다.

```bash
cp .env .env.backup-2026-08-13
docker compose exec database pg_dump -U postgres immich > immich-db-2026-08-13.sql
docker compose pull
docker compose up -d
docker compose logs -f immich-server
```

사진 원본은 NAS 스냅샷과 별도 백업으로 보호해야 한다. Immich의 무결성 검사는 파일과 DB 기록의 차이를 찾아주는 기능이지 백업 자체는 아니다. NAS 고장에 대비해 다른 디스크나 외장 저장소에 3-2-1 백업을 두는 구성이 맞다.

Immich v3.0은 NAS에 올릴 만한 기능이 충분히 늘었지만, 사진 원본·데이터베이스·외부 접속을 한 번에 공개하는 서비스이기도 하다. `v3` 자동 추적보다는 변경 시점을 통제할 수 있게 버전을 고정하고, 업데이트 전 DB 백업과 복구 테스트를 습관으로 만드는 편이 안전하다.

- [Immich v3.0 공식 릴리스 노트](https://immich.app/blog/v3.0.0-release)
- [Immich 공식 Docker 설치 문서](https://docs.immich.app/install/docker-compose)

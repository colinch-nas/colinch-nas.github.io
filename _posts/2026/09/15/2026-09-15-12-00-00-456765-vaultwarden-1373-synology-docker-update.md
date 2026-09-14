---
layout: post
title: "Vaultwarden 1.37.3 업데이트 — Synology Docker에서 안전하게 버전 고정·롤백하기"
description: "Vaultwarden 최신 안정 버전 1.37.3을 Synology DSM 7.2 Docker에 적용하는 방법과 Bitwarden 클라이언트 호환성, 백업·롤백·외부접속 보안 점검을 정리한다."
date: 2026-09-15
tags: [Vaultwarden, Synology, Docker, 자체호스팅, NAS보안]
comments: true
share: true
---

![Synology NAS에서 Vaultwarden 비밀번호 서버를 업데이트하는 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

Vaultwarden 최신 안정 버전은 1.37.3이다. 2026년 9월 13일 릴리스에는 최신 Web Vault에서 비밀번호 변경이 실패하던 문제와 MariaDB 12.2.2 마이그레이션 문제가 수정됐다. 비밀번호 서버는 “컨테이너 이미지를 최신으로 받기”만 하고 끝내면 안 된다. 업데이트 전에 데이터베이스 백업을 만들고, `latest` 대신 버전을 고정해야 복구할 수 있다.

## 이번에 확인할 환경

아래 구성은 Synology DSM 7.2 계열의 Container Manager(예전 Docker) 기준이다. 이미 Vaultwarden을 설치해 사용 중인 경우에도 그대로 적용할 수 있다.

| 항목 | 기준값 |
|---|---|
| NAS | Synology DS923+ 또는 x86-64 모델 |
| DSM | DSM 7.2.2 이상 |
| 컨테이너 | `vaultwarden/server:1.37.3` |
| 데이터 경로 | `/volume1/docker/vaultwarden/data` |
| 외부 접속 | Reverse Proxy(외부 요청을 내부 컨테이너로 전달) + HTTPS |

이번 릴리스는 2026.7.0 이상 Bitwarden 클라이언트 지원을 위해 필요한 1.37 계열이다. 다만 모바일 앱과 브라우저 확장 프로그램을 한꺼번에 업데이트하면 문제가 생겼을 때 원인을 찾기 어렵다. 서버를 먼저 올리고, 테스트 계정으로 로그인·비밀번호 변경·자동 채우기를 확인하는 순서가 안전하다.

## 1. 업데이트 전 백업 만들기

가장 먼저 Container Manager에서 Vaultwarden을 중지한다. 실행 중인 SQLite 파일을 그대로 복사하면 마지막 쓰기 작업이 빠질 수 있어서다. SSH로 접속했다면 데이터 폴더를 압축해 별도 위치에 보관한다.

```bash
cd /volume1/docker/vaultwarden
docker compose stop vaultwarden
tar -czf /volume1/backup/vaultwarden-pre-1.37.3-$(date +%Y%m%d-%H%M).tar.gz data
```

백업 파일이 실제로 만들어졌는지 크기와 목록을 확인한다.

```bash
ls -lh /volume1/backup/vaultwarden-pre-1.37.3-*.tar.gz
tar -tzf /volume1/backup/vaultwarden-pre-1.37.3-*.tar.gz | head
```

여기서 삽질하기 쉬운 부분은 백업 경로를 컨테이너 내부 `/data`로 착각하는 것이다. NAS 호스트 기준의 `/volume1/docker/vaultwarden/data`에 `db.sqlite3`, `attachments`, `sends`가 있어야 한다. 이 폴더 전체가 Vaultwarden의 핵심 데이터다.

## 2. Compose 파일에서 버전 고정

기존 파일의 `vaultwarden/server:latest`를 아래처럼 바꾼다. `latest`는 다음 재시작 때 예고 없이 이미지가 바뀌므로 비밀번호 서버에는 맞지 않는다.

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:1.37.3
    container_name: vaultwarden
    restart: unless-stopped
    environment:
      DOMAIN: "https://vault.example.com"
      SIGNUPS_ALLOWED: "false"
      ADMIN_TOKEN: "여기에-긴-랜덤-토큰"
      WEBSOCKET_ENABLED: "true"
    volumes:
      - /volume1/docker/vaultwarden/data:/data
    ports:
      - "127.0.0.1:8085:80"
      - "127.0.0.1:3012:3012"
```

`127.0.0.1`로 바인딩하면 NAS 외부 인터페이스에서 8085와 3012 포트가 직접 열리지 않는다. Synology Reverse Proxy가 같은 NAS의 8085 포트로 연결하도록 구성한다. 3012는 실시간 동기화용 WebSocket 포트라서 프록시가 WebSocket을 전달하지 않으면 알림과 동기화가 늦게 반영될 수 있다.

## 3. 이미지 교체와 기본 확인

Compose 파일이 있는 위치에서 1.37.3 이미지만 내려받고 컨테이너를 다시 만든다.

```bash
docker compose pull vaultwarden
docker compose up -d vaultwarden
docker compose ps
docker compose logs --tail=50 vaultwarden
```

로그에 반복적인 `database` 오류가 없고 웹 화면 하단에서 1.37.3을 확인하면 1차 업데이트는 끝난다. 곧바로 관리자 페이지를 열기보다 일반 계정에서 다음 네 가지를 확인한다.

| 확인 항목 | 성공 기준 |
|---|---|
| 로그인 | 기존 계정으로 로그인 가능 |
| 비밀번호 변경 | 테스트 항목의 새 비밀번호 저장 |
| 모바일 동기화 | 앱에서 새 항목이 보임 |
| 브라우저 자동 채우기 | HTTPS 도메인에서 정상 동작 |

## 롤백해야 하는 경우

로그인 실패나 데이터베이스 오류가 보이면 컨테이너를 계속 재시작하지 않는다. 먼저 중지하고 이미지 태그를 이전 버전으로 되돌린 뒤, 업데이트 전 백업을 복원한다. 예를 들어 이전 버전이 1.37.1이었다면 다음처럼 실행한다.

```bash
docker compose down
sed -i 's/vaultwarden\/server:1.37.3/vaultwarden\/server:1.37.1/' docker-compose.yml
mv data data.failed
mkdir data
tar -xzf /volume1/backup/vaultwarden-pre-1.37.3-YYYYMMDD-HHMM.tar.gz -C .
docker compose up -d
```

백업 파일이 실제로 존재하고 압축 해제가 성공했는지 확인한 뒤에만 복구한다. `data` 폴더를 바로 삭제하지 않고 `data.failed`로 이름을 바꿔 보존하는 이유도 여기에 있다. 파일명이 틀리거나 압축 해제가 실패하면 복구할 원본까지 사라질 수 있다.

## 외부 공개 전에 다시 보는 항목

관리자 페이지 `/admin`은 Reverse Proxy에서 별도 접근 제한을 걸거나 외부에 공개하지 않는 편이 좋다. `ADMIN_TOKEN`은 Compose 파일에 평문으로 남으므로 NAS 관리자만 읽을 수 있는 폴더 권한을 적용하고, 채팅이나 깃 저장소에 붙여넣지 않는다. 회원가입을 닫아도 기존 계정의 비밀번호가 안전해지는 것은 아니므로 모든 계정에 2단계 인증을 설정한다.

이번 업데이트에서 가장 실용적인 변화는 버전 고정과 복구 절차다. 새 이미지가 정상 실행됐다는 사실보다, 문제가 생겼을 때 10분 안에 이전 데이터와 이미지로 돌아갈 수 있는지가 더 중요하다.

## 짧은 요점 정리

- Vaultwarden 1.37.3으로 올리기 전에 `data` 전체를 백업한다.
- `latest` 대신 `vaultwarden/server:1.37.3`처럼 버전을 고정한다.
- 8085와 WebSocket 3012는 Reverse Proxy 뒤에 두고 HTTPS만 외부에 공개한다.
- 로그인·비밀번호 변경·모바일 동기화·자동 채우기를 테스트 계정으로 확인한다.
- 롤백 명령을 실행하기 전 백업 압축 파일과 복원 경로를 반드시 검증한다.

참고 링크: [Vaultwarden 1.37.3 공식 릴리스](https://github.com/dani-garcia/vaultwarden/releases), [Vaultwarden 공식 저장소](https://github.com/dani-garcia/vaultwarden)

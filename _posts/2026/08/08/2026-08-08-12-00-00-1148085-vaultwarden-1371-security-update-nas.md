---
layout: post
title: "Vaultwarden 1.37.1 업데이트 - NAS에서 보안 패치와 안전한 롤백 적용"
description: "Vaultwarden 1.37.1의 초대 링크·Alpine 이미지 수정과 Bitwarden 2026.7 호환성을 확인하고, Synology NAS Docker에서 백업 후 안전하게 업데이트하는 방법을 정리한다."
date: 2026-08-08
tags: [Vaultwarden, Docker, NAS보안, 자체호스팅, Synology]
comments: true
share: true
---

![Vaultwarden NAS 비밀번호 서버 업데이트](https://images.unsplash.com/photo-1563013544-824ae1b704d3?w=1200&q=80)

Vaultwarden을 NAS에서 쓰고 있다면 2026년 7월 29일 공개된 1.37.1로 올리는 편이 좋다. 최신 Bitwarden 클라이언트와의 호환 문제를 해결한 1.37.0에 이어, 1.37.1은 초대 링크 오류와 Alpine 기반 이미지 빌드 문제를 고쳤다. 비밀번호 서버는 “일단 최신 이미지”를 당기는 것보다 백업과 되돌리기 순서를 먼저 정하는 게 안전하다.

## 이번 업데이트에서 확인할 점

Vaultwarden 1.37.0은 Bitwarden 클라이언트 2026.7.0 이상 지원을 위해 필요한 버전이며, 아이콘 엔드포인트 SSRF(서버가 공격자 요청을 대신 보내게 하는 취약점) 등 여러 보안 수정도 포함했다. 1.37.1은 초대 링크 처리와 Alpine 이미지의 OpenSSL 빌드 문제를 추가로 수정했다.

| 항목 | 확인 내용 |
|---|---|
| 최신 이미지 | `vaultwarden/server:1.37.1` |
| 적용 대상 | Docker·Container Manager로 실행 중인 NAS |
| 업데이트 전 필수 | `/data` 전체 백업, 현재 이미지 태그 기록 |
| 외부 접속 | HTTPS 도메인과 WebSocket 프록시 필요 |

이 글의 구성은 Synology DSM 7.2 Container Manager, `/volume1/docker/vaultwarden` 경로, 이미 역방향 프록시(외부 요청을 내부 컨테이너로 전달하는 중계 서버)가 구성된 환경을 기준으로 했다.

## 업데이트 전 백업

먼저 데이터베이스와 첨부파일이 있는 `/data`를 별도 백업 폴더에 복사한다. 실행 중 SQLite 파일을 그대로 복사하는 것보다 컨테이너 안에서 SQLite 백업 명령을 쓰는 쪽이 안전하다.

```bash
cd /volume1/docker/vaultwarden
BACKUP_DIR="backup/$(date +%Y%m%d-%H%M)"
mkdir -p "$BACKUP_DIR"
docker exec vaultwarden sqlite3 /data/db.sqlite3 \
  ".backup '/data/backup-vault.sqlite3'"
cp -a vw-data/backup-vault.sqlite3 "$BACKUP_DIR/"
cp -a vw-data/attachments "$BACKUP_DIR/"
cp compose.yml "$BACKUP_DIR/"
```

`sqlite3` 명령이 없다는 오류가 나오면 데이터 폴더 전체를 복사한다. 이때 컨테이너를 잠시 멈추고 복사해야 데이터베이스가 변경되지 않는다.

## Compose 이미지 태그 고정

`latest` 대신 버전을 고정하면 다음 업데이트에서 예기치 않게 바뀌지 않는다. 기존 환경변수와 볼륨 경로는 유지하고 `image`만 바꾼다.

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:1.37.1
    container_name: vaultwarden
    restart: unless-stopped
    environment:
      DOMAIN: "https://vault.example.com"
      SIGNUPS_ALLOWED: "false"
      WEBSOCKET_ENABLED: "true"
      ADMIN_TOKEN: "환경파일에_보관한_관리자_토큰"
    volumes:
      - ./vw-data:/data
    ports:
      - "11001:80"
```

관리자 토큰을 Compose 파일에 직접 넣으면 Git 저장소나 NAS 백업에 평문으로 남는다. 별도 `.env` 파일로 옮기고 파일 권한을 제한한다.

```bash
printf 'ADMIN_TOKEN=%s\n' "$(openssl rand -base64 48)" > .env
chmod 600 .env
docker compose config >/dev/null
docker compose pull vaultwarden
docker compose up -d vaultwarden
docker compose logs --tail=100 vaultwarden
```

컨테이너가 재생성된 뒤 웹 볼트 로그인, 모바일 동기화, 새 항목 저장을 각각 확인한다. WebSocket이 끊기면 실시간 동기화가 늦어질 수 있으니 역방향 프록시가 `/notifications/hub` 요청을 그대로 전달하는지도 본다.

## 문제가 생기면 되돌리기

로그에 데이터베이스 마이그레이션 오류가 있거나 클라이언트가 빈 보관함을 보여주면 계속 재시작하지 않는다. Compose의 이미지 태그를 이전 버전으로 바꾸고 컨테이너를 다시 만든다.

```bash
docker compose down
sed -i 's/vaultwarden\/server:1.37.1/vaultwarden\/server:1.36.0/' compose.yml
docker compose up -d
docker compose logs --tail=100 vaultwarden
```

데이터 자체가 손상된 경우에는 컨테이너를 멈춘 뒤 백업한 `db.sqlite3`와 `attachments`를 `vw-data`에 복원한다. 업데이트 중 데이터 폴더를 삭제하는 것은 복구 방법이 아니므로 피해야 한다.

## 적용 후 보안 점검

- `SIGNUPS_ALLOWED=false`인지 확인한다.
- 관리자 페이지(`/admin`)를 인터넷 전체에 공개하지 않는다.
- 80·443 외에 컨테이너 포트 11001을 공유기에서 포워딩하지 않는다.
- NAS 스냅샷만 믿지 말고 다른 저장소에도 `vw-data`를 백업한다.
- Bitwarden 확장 프로그램과 모바일 앱도 공식 스토어 최신 버전으로 맞춘다.

Vaultwarden 1.37.1은 기능 추가보다 호환성과 패치 성격이 강한 업데이트다. NAS에서는 백업 → 태그 고정 → 컨테이너 재생성 → 로그인·동기화 확인 순서만 지켜도 위험을 크게 줄일 수 있다.

참고 자료:

- [Vaultwarden 1.37.1 릴리스 노트](https://github.com/dani-garcia/vaultwarden/releases/tag/1.37.1)
- [Vaultwarden 1.37.0 보안·호환성 변경](https://github.com/dani-garcia/vaultwarden/releases/tag/1.37.0)
- [Vaultwarden Docker Compose 공식 위키](https://github.com/dani-garcia/vaultwarden/wiki/Using-Docker-Compose)

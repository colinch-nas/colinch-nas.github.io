---
layout: post
title: "Vaultwarden으로 비밀번호 관리 서버 직접 운영하기 — Bitwarden 자체 호스팅"
description: "Bitwarden 호환 오픈소스인 Vaultwarden을 Docker로 홈서버에 설치하는 방법. HTTPS 설정, 관리자 토큰, 브라우저 확장 연동까지 실전 가이드."
date: 2026-06-15
tags: [Vaultwarden, 자체호스팅, 보안, Docker]
comments: true
share: true
---

# Vaultwarden으로 비밀번호 관리 서버 직접 운영하기 — Bitwarden 자체 호스팅

![Vaultwarden 비밀번호 관리 보안 서버](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=800&q=80)

1Password는 월 구독이 필요하고, LastPass는 작년에 또 해킹 당했다. Bitwarden은 무료인데 셀프호스팅 서버 요구사항이 무거워서 NAS에서 돌리기 부담스럽다. Vaultwarden은 Bitwarden 프로토콜과 완전 호환되면서 메모리 10MB 남짓 쓰는 경량 서버다.

이전 자체 호스팅 서비스 글에서 [Nextcloud로 클라우드 구축]({% post_url 2026-06-14-10-00-00-123450-nextcloud-install-self-hosted-cloud %})을 다뤘다. 이번엔 비밀번호 관리 서버를 올린다.

## 환경 정보

| 항목 | 버전/모델 |
|------|----------|
| NAS 모델 | Synology DS923+ |
| Vaultwarden 버전 | 1.32.x |
| Docker 이미지 | vaultwarden/server:latest |

주의: Vaultwarden은 **HTTPS 없이는 작동하지 않는다**. 브라우저의 보안 정책 때문에 암호화되지 않은 HTTP에서 비밀번호 관련 API가 차단된다. 외부 접속을 위한 HTTPS 설정이 먼저 돼 있어야 한다.

## docker-compose로 설치

```yaml
# /volume1/docker/vaultwarden/docker-compose.yml
version: "3.9"
services:
  vaultwarden:
    image: vaultwarden/server:latest
    container_name: vaultwarden
    restart: unless-stopped
    ports:
      - "8070:80"
    environment:
      - DOMAIN=https://vault.example.com    # 실제 도메인으로 변경
      - ADMIN_TOKEN=your-secure-admin-token  # 관리자 페이지 접속 토큰
      - SIGNUPS_ALLOWED=false                # 가입 비허용 (나만 쓸 경우)
      - WEBSOCKET_ENABLED=true
    volumes:
      - /volume1/docker/vaultwarden/data:/data
```

**ADMIN_TOKEN 생성**:

```bash
# openssl로 안전한 토큰 생성
openssl rand -base64 48
# 출력된 문자열을 ADMIN_TOKEN에 사용
```

```bash
cd /volume1/docker/vaultwarden
sudo docker compose up -d
```

## HTTPS 리버스 프록시 설정

Vaultwarden 자체에는 SSL이 없다. 앞단에 리버스 프록시를 두고 HTTPS를 처리해야 한다. Synology 리버스 프록시를 쓰는 경우:

```
소스: HTTPS / vault.example.com / 443
대상: HTTP / localhost / 8070
```

WebSocket도 활성화해야 실시간 동기화가 된다. Synology 리버스 프록시의 **커스텀 헤더** 탭에서:

```
Upgrade: $http_upgrade
Connection: $connection_upgrade
```

Nginx로 직접 설정하는 경우:

```nginx
server {
    listen 443 ssl;
    server_name vault.example.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location / {
        proxy_pass http://localhost:8070;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
    }
}
```

## 첫 번째 계정 생성

`https://vault.example.com`으로 접속해서 **계정 만들기**를 클릭. `SIGNUPS_ALLOWED=false`로 설정했다면 처음 한 번은 가입이 되지만 이후에는 막힌다. 본인 계정 만든 뒤에도 막힌 상태 유지.

관리자 페이지: `https://vault.example.com/admin`
ADMIN_TOKEN을 입력하면 사용자 관리, 초대, 설정 변경이 가능하다.

## 브라우저 확장 연동

Chrome, Firefox, Edge용 Bitwarden 확장을 설치한다. 확장 로그인 화면에서 이메일 입력란 위의 **서버** 아이콘을 클릭해 서버 URL을 변경한다.

```
서버 URL: https://vault.example.com
```

로그인하면 기존 Bitwarden 클라우드 계정처럼 자동완성, 비밀번호 생성, 보안 노트 등 모든 기능을 쓸 수 있다.

모바일 앱(iOS/Android)도 같은 방식으로 서버를 커스텀 URL로 변경하면 된다.

![Vaultwarden 비밀번호 자체 호스팅 화면](https://images.unsplash.com/photo-1544197150-880d4212a0c4?w=800&q=80)

## 백업

비밀번호 데이터는 `/volume1/docker/vaultwarden/data/` 폴더 안에 있다. 여기 있는 `db.sqlite3` 파일이 전부다.

```bash
# 백업 스크립트 예시
cp /volume1/docker/vaultwarden/data/db.sqlite3 \
   /volume1/backup/vaultwarden/db-$(date +%Y%m%d).sqlite3
```

Hyper Backup으로 `/volume1/docker/vaultwarden` 폴더 전체를 정기 백업에 포함시키는 게 가장 간단하다.

## 삽질했던 부분

**WebSocket 미설정**: 처음엔 WebSocket 헤더 설정을 빠뜨려서 앱에서 비밀번호 변경하면 다른 기기에 바로 반영이 안 됐다. 동기화가 느려서 처음엔 서버 문제인 줄 알았는데, 리버스 프록시에서 WebSocket 업그레이드 헤더를 추가하니 해결됐다.

**비밀번호 내보내기를 먼저 할 것**: 기존에 다른 비밀번호 관리자(1Password, LastPass 등)에서 마이그레이션할 때 내보내기 형식을 확인해야 한다. Bitwarden 호환 JSON 또는 CSV 형식으로 내보낸 뒤 Vaultwarden 웹에서 가져오기 하면 된다.

**도메인 변경 시 클라이언트 재로그인 필요**: 서버 URL이 바뀌면 모든 기기에서 다시 로그인해야 한다.

## 한 줄 정리

Vaultwarden은 메모리 10MB로 Bitwarden 기능 대부분을 집 서버에서 돌릴 수 있다. HTTPS와 WebSocket 설정만 제대로 하면 된다.

---
다음엔 [Immich 설치 — 구글 포토 대체 자체 호스팅 사진 관리 서버]({% post_url 2026-06-16-10-00-00-148140-immich-google-photos-alternative %})를 다룬다.

**참고 링크**
- [Vaultwarden GitHub](https://github.com/dani-garcia/vaultwarden)
- [Vaultwarden Wiki](https://github.com/dani-garcia/vaultwarden/wiki)
- [Bitwarden 공식 클라이언트 앱](https://bitwarden.com/download/)

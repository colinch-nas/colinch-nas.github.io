---
layout: post
title: "Vaultwarden 1.37.1 NAS 설치 - Docker로 비밀번호 서버 직접 운영하기"
description: "Vaultwarden 1.37.1을 Synology NAS Docker에 설치하고 DSM 역방향 프록시 HTTPS, 관리자 토큰, SQLite 백업까지 설정하는 실전 가이드다."
date: 2026-08-15
tags: [Vaultwarden, Docker, Synology, 자체호스팅, NAS보안]
comments: true
share: true
---

![Vaultwarden NAS 자체 호스팅 비밀번호 서버](/assets/images/2026-08-15-vaultwarden-nas.png)
이 그림에서 봐야 할 부분은 NAS와 공유기 사이의 내부망 구성이다. 비밀번호 데이터베이스는 NAS에 두되, 외부에는 HTTPS로만 공개해야 한다.

Vaultwarden 1.37.1을 Synology NAS Docker에 올려 Bitwarden 호환 비밀번호 서버를 만들었다. 2026년 8월 15일 공식 릴리스 기준 최신 버전은 1.37.1이며, 초대 링크 문제와 Alpine 이미지 빌드 문제가 수정된 버전이다. 최근 Bitwarden 클라이언트가 2026.7 계열로 올라오면서 구버전 서버에서 로그인·동기화가 꼬이는 경우도 있어 새로 설치한다면 이 버전부터 시작하는 편이 낫다.

## 이번에 사용한 환경

| 항목 | 설정값 |
|---|---|
| NAS | Synology DS923+ |
| OS | DSM 7.2.2 |
| 컨테이너 | vaultwarden/server:1.37.1-alpine |
| 데이터 경로 | `/volume1/docker/vaultwarden/data` |
| 외부 주소 | `https://vault.example.com` |

Vaultwarden은 공식 Bitwarden 서버가 아니다. 다만 Bitwarden 클라이언트와 호환되는 Rust 기반 서버라 브라우저 확장, 모바일 앱, 데스크톱 앱을 그대로 연결할 수 있다. 민감한 데이터를 다루므로 NAS 관리 계정 비밀번호와 Vaultwarden 마스터 비밀번호는 서로 다르게 만들었다.

## Docker Compose로 설치

먼저 File Station에서 `/volume1/docker/vaultwarden/data` 폴더를 만들고, 아래 Compose 파일을 같은 프로젝트 폴더에 저장한다. `ADMIN_TOKEN`에는 평문을 넣지 않고 Argon2id 해시를 사용한다.

```yaml
services:
  vaultwarden:
    image: vaultwarden/server:1.37.1-alpine
    container_name: vaultwarden
    restart: unless-stopped
    environment:
      DOMAIN: "https://vault.example.com"
      SIGNUPS_ALLOWED: "false"
      INVITATIONS_ALLOWED: "true"
      WEBSOCKET_ENABLED: "true"
      ADMIN_TOKEN: "여기에_Argon2id_해시"
    volumes:
      - /volume1/docker/vaultwarden/data:/data
    ports:
      - "127.0.0.1:8080:80"
```

`127.0.0.1`을 붙인 이유가 중요하다. NAS의 모든 네트워크 인터페이스에서 8080 포트를 열지 않고, DSM 역방향 프록시만 컨테이너에 접근하게 만든다. 프로젝트 폴더에서 다음 명령으로 기동한다.

```bash
sudo docker compose up -d
sudo docker compose logs -f vaultwarden
```

관리자 토큰은 컨테이너가 실행된 뒤 아래 명령으로 생성한다. 출력된 해시 전체를 Compose의 `ADMIN_TOKEN` 값에 넣고 컨테이너를 재생성한다.

```bash
sudo docker run --rm -it vaultwarden/server:1.37.1-alpine /vaultwarden hash
sudo docker compose up -d --force-recreate
```

## DSM 역방향 프록시와 HTTPS

DSM에서 **제어판 → 로그인 포털 → 고급 → 역방향 프록시**를 열고 다음처럼 등록한다.

| 항목 | 값 |
|---|---|
| 소스 호스트 이름 | `vault.example.com` |
| 소스 프로토콜/포트 | HTTPS / 443 |
| 대상 호스트/포트 | `127.0.0.1` / 8080 |

인증서는 **제어판 → 보안 → 인증서**에서 `vault.example.com`용 Let's Encrypt 인증서를 발급한다. 공유기에서 외부로 여는 포트는 443 하나만 남겼다. 8080을 포트포워딩하면 암호화되지 않은 접속 경로가 생기고, 관리자 화면까지 불필요하게 노출된다.

Bitwarden 앱의 서버 URL에는 `https://vault.example.com`을 넣는다. 웹소켓이 연결되지 않으면 DSM 역방향 프록시의 사용자 지정 헤더에 `Upgrade: $http_upgrade`, `Connection: upgrade`를 추가하고 `WEBSOCKET_ENABLED`가 `true`인지 확인한다.

## 첫 로그인과 운영 보안

`https://vault.example.com/admin`에서 관리자 토큰으로 들어간 뒤 회원가입을 닫았다. 초대가 필요할 때만 `INVITATIONS_ALLOWED=true`를 유지하고, 가족 계정을 만든 뒤에는 다시 `false`로 바꾸는 방식이 안전하다. 관리자 페이지는 평소 사용하지 않으므로 역방향 프록시에서 별도 접근 제한을 걸어도 된다.

| 확인할 것 | 기준 |
|---|---|
| 외부 공개 | 443만 허용, 8080 차단 |
| 회원가입 | 기본 `false` |
| 관리자 토큰 | Argon2id 해시, 평문 금지 |
| 마스터 비밀번호 | NAS 관리자 비밀번호와 다르게 설정 |
| 2단계 인증 | 각 Bitwarden 계정에서 활성화 |

데이터가 SQLite 하나에 모인다고 가볍게 보면 안 된다. `/data/db.sqlite3`뿐 아니라 첨부파일, 아이콘 캐시, 설정 파일도 같이 백업해야 복구가 된다. Synology Hyper Backup 작업 대상으로 `/volume1/docker/vaultwarden/data` 전체를 지정하고, 최소 하루 한 번 버전 백업을 남겼다. 백업 파일을 같은 NAS에만 두면 디스크 고장에는 취약하므로 USB 디스크나 다른 NAS에도 한 부씩 복제한다.

```bash
sudo docker compose stop
sudo tar -czf /volume1/backup/vaultwarden-$(date +%F).tar.gz \
  -C /volume1/docker/vaultwarden data
sudo docker compose start
```

운영 중에는 업데이트 전 백업을 먼저 만들고, `docker compose pull` 뒤 변경 로그를 확인한다. Vaultwarden 1.37.1은 2026.7 계열 클라이언트 호환성 때문에 특히 구버전으로 방치할 이유가 적지만, 자동 업데이트 도구가 즉시 컨테이너를 갈아끼우게 두면 로그인 장애를 놓칠 수 있다.

정리하면 NAS에 Vaultwarden을 설치하는 핵심은 컨테이너 실행 자체가 아니다. `127.0.0.1` 바인딩으로 포트를 줄이고, DSM에서 HTTPS를 끝내고, 회원가입을 닫고, `/data` 전체를 다른 장치에도 백업하는 네 가지가 실제 운영을 좌우한다.

참고: [Vaultwarden 1.37.1 공식 릴리스](https://github.com/dani-garcia/vaultwarden/releases/tag/1.37.1), [Vaultwarden 공식 Wiki](https://github.com/dani-garcia/vaultwarden/wiki)

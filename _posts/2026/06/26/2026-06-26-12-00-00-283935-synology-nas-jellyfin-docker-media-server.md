---
layout: post
title: "Synology NAS Jellyfin 설치 — Docker로 집에서 미디어 서버 구축"
description: "Synology NAS DSM 7.2.2에서 Docker Container Manager로 Jellyfin 미디어 서버를 설치하는 방법. 트랜스코딩 설정, 미디어 폴더 구조, 외부 접속까지 실제 설치 경험으로 정리했다."
date: 2026-06-26
tags: [Synology, Jellyfin, Docker, NAS설정, 미디어서버]
comments: true
share: true
---

![홈 미디어 서버 스트리밍 TV](https://images.unsplash.com/photo-1593784991095-a205069470b6?w=800&q=80)

Synology DS923+에 Jellyfin을 Docker로 올리면 집 안 어디서든, 외부에서도 넷플릭스처럼 영상을 스트리밍할 수 있다. Plex처럼 계정 연동 없이 완전 로컬·자체 호스팅으로 돌아가고, 라이선스 비용도 없다.

## 환경 정보

| 항목 | 버전/모델 |
|------|----------|
| NAS 모델 | Synology DS923+ |
| OS | DSM 7.2.2-72806 |
| 컨테이너 관리 | Container Manager 20.10.23 |
| Jellyfin 버전 | 10.9.x (jellyfin/jellyfin:latest) |
| 스토리지 | /volume1 (Btrfs 풀) |

[이전 편에서 Docker Container Manager 설치와 기본 사용법을 다뤘다.]({% post_url 2026-06-26-10-00-00-271590-synology-docker-install-container-management %}) 이번 글은 그 연장선이다.

## 왜 Jellyfin인가

Plex도 있고 Emby도 있는데 굳이 Jellyfin을 고른 이유는 하나다: **완전 무료**. 트랜스코딩, 앱, 원격 접속 전부 돈 안 낸다. Plex는 원격 스트리밍과 라이브 TV 기능에 Plex Pass(월 구독)가 필요하고, Emby도 Premier 없으면 앱에서 제한이 걸린다.

오픈소스라 개발 속도가 빠르고, 10.9 버전 기준으로 UI도 많이 깔끔해졌다.

## 미디어 폴더 구조 먼저 잡기

컨테이너 띄우기 전에 NAS에 폴더 구조를 만들어두는 게 낫다. 나중에 바꾸면 라이브러리를 다시 스캔해야 한다.

File Station에서 `/volume1/media` 아래에 이렇게 만들었다:

```
/volume1/media/
├── movies/      # 영화
├── tv/          # TV 시리즈
└── music/       # 음악
```

Jellyfin이 폴더 구조를 기준으로 라이브러리를 인식하기 때문에 TV 시리즈는 아래처럼 정리되어 있어야 에피소드 매칭이 된다:

```
tv/
└── Breaking Bad/
    ├── Season 01/
    │   ├── S01E01.mkv
    │   └── S01E02.mkv
    └── Season 02/
        └── ...
```

영화는 그냥 `movies/영화제목 (연도).mkv` 식으로 넣으면 자동 매칭된다.

## docker-compose.yml 작성

DSM의 Container Manager는 `docker-compose up`을 직접 쓸 수 없다. 대신 Container Manager > 프로젝트 탭에서 compose 파일을 붙여넣는 방식으로 쓴다.

SSH로 접속해서 직접 올리는 방법도 있지만, GUI가 편하다면 프로젝트 탭을 쓰면 된다.

```yaml
version: "3"
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    environment:
      - PUID=1026
      - PGID=100
      - TZ=Asia/Seoul
    volumes:
      - /volume1/docker/jellyfin/config:/config
      - /volume1/docker/jellyfin/cache:/cache
      - /volume1/media:/media
    ports:
      - 8096:8096
    restart: unless-stopped
```

`/volume1/docker/jellyfin/config`와 `/volume1/docker/jellyfin/cache` 폴더는 미리 만들어줘야 한다. File Station에서 `/volume1/docker/jellyfin/` 아래에 `config`, `cache` 두 폴더 생성.

## PUID/PGID — 여기서 반나절 날렸다

처음에 PUID랑 PGID를 기본값 1000으로 넣었다가 미디어 파일을 못 읽었다. Synology는 첫 번째 사용자 UID가 1026부터 시작한다. 1000은 아무 것도 아닌 ID라 권한이 없었던 거다.

내 계정 UID 확인:

```bash
# SSH 접속 후
id username
# uid=1026(username) gid=100(users) ...
```

Synology의 기본 `users` 그룹 GID는 100이다. `PGID=100`으로 설정하면 `users` 그룹 권한으로 파일 접근이 된다.

미디어 폴더 권한도 확인:

```bash
ls -la /volume1/media/
# drwxrwxrwx  또는  drwxr-xr-x 1026 users ...
```

권한이 너무 좁으면 Jellyfin이 폴더를 읽지 못한다. 아래 명령으로 읽기 권한을 준다:

```bash
chmod -R 755 /volume1/media/
```

## 컨테이너 실행 및 초기 설정

Container Manager > 프로젝트 > 생성에서 위 compose 파일을 붙여넣고 실행하면 된다.

컨테이너가 뜨면 브라우저에서 `http://NAS_IP:8096`으로 접속. 초기 설정 마법사가 뜨는데:

1. 언어 선택 (한국어 지원됨)
2. 관리자 계정 생성
3. 라이브러리 추가 — 미디어 유형 선택 후 `/media/movies`, `/media/tv` 경로 지정
4. 메타데이터 언어 설정 (한국어로 바꾸면 한글 포스터·설명이 나옴)

라이브러리 스캔이 끝나면 파일이 자동으로 포스터와 함께 정리된다.

![NAS 서버 하드웨어](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)

## 하드웨어 가속 트랜스코딩 (DS923+)

DS923+는 AMD Ryzen R1600 CPU를 쓴다. Intel NUC처럼 QuickSync가 없어서 GPU 가속은 없지만, CPU 트랜스코딩 성능 자체는 괜찮다.

Jellyfin 대시보드 > 재생 > 트랜스코딩에서:

- **하드웨어 가속**: `Software` (CPU)
- **인코딩 스레드 수**: 2~4 (코어 수 절반이 안정적)
- **트랜스코딩 임시 경로**: `/cache/transcodes` (자동 설정됨)

4K 파일을 1080p로 트랜스코딩하면 CPU가 꽤 올라간다. DS923+에서 단일 4K 스트림 트랜스코딩 시 CPU 사용률이 60~80%까지 치솟는 걸 확인했다. 동시 2스트림은 버겁다.

4K를 클라이언트에서 직접 재생(Direct Play)할 수 있다면 트랜스코딩이 안 일어나서 CPU 부담이 없다. TV 앱이나 PC Jellyfin 앱에서 Direct Play를 지원하면 그쪽을 쓰는 게 낫다.

## 외부 접속 설정

두 가지 방법이 있다.

**방법 1: Synology DDNS + 역방향 프록시 (기본 추천)**

DSM의 로그인 포털 > 고급 탭 > 역방향 프록시에서 규칙 추가:

- 소스: `jellyfin.내도메인.synology.me` (HTTPS, 443)
- 대상: `localhost:8096` (HTTP)

Synology DDNS를 켜두면 `xxx.synology.me` 도메인이 생기고, Let's Encrypt 인증서도 DSM에서 자동으로 발급된다. 포트 포워딩은 80과 443만 열면 된다.

**방법 2: Nginx Proxy Manager (Docker)**

[이미 Nginx Proxy Manager를 올린 경우]({% post_url 2026-06-23-10-00-00-234555-nginx-proxy-manager-domain-https %})라면 거기서 Proxy Host를 추가하면 된다:

- Domain: `jellyfin.yourdomain.com`
- Forward Hostname/IP: NAS IP
- Forward Port: 8096
- SSL 탭에서 Let's Encrypt 발급

방법 2는 커스텀 도메인을 쓰거나 여러 서비스를 한 곳에서 관리하고 싶을 때 편하다.

## 모바일·TV 앱

- **Android / iOS**: Jellyfin 공식 앱 ([App Store](https://apps.apple.com/app/jellyfin-mobile/id1480192618), [Play Store](https://play.google.com/store/apps/details?id=org.jellyfin.mobile))
- **Android TV / Fire TV**: [Jellyfin for Android TV](https://play.google.com/store/apps/details?id=org.jellyfin.androidtv)
- **Samsung Smart TV**: Tizen용 앱이 있는데 설치가 조금 번거롭다. Kodi + Jellyfin 플러그인이 더 안정적인 경우도 있다.

iOS 앱은 2026년 기준으로 꽤 안정화됐다. 예전엔 Direct Play 지원이 불안정했는데 지금은 대부분 잘 된다.

## 한 줄 정리

DS923+ + Container Manager + docker-compose 조합으로 Jellyfin 설치 자체는 10분이면 된다. PUID/PGID 권한 문제만 처음에 잡으면 그 다음은 막히는 게 없다.

---

다음엔 Synology NAS 외부 접속을 제대로 다룬다 — DDNS 설정부터 Let's Encrypt 인증서 자동 갱신까지.

**참고 링크**
- [Jellyfin 공식 문서](https://jellyfin.org/docs/)
- [Jellyfin Docker Hub](https://hub.docker.com/r/jellyfin/jellyfin)
- [Synology Container Manager 가이드](https://kb.synology.com/ko-kr/DSM/help/ContainerManager/docker_desc)

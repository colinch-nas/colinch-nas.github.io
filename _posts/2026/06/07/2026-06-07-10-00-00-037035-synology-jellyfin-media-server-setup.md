---
layout: post
title: "Synology NAS에 Jellyfin 설치하기 — Plex 없이 미디어 서버 구축"
description: "Synology DS923+에 Docker로 Jellyfin을 설치하고 미디어 라이브러리를 구성하는 방법. 하드웨어 트랜스코딩 설정과 외부 접속 방법까지 실전 가이드."
date: 2026-06-07
tags: [Synology, Jellyfin, 미디어서버, Docker]
comments: true
share: true
---

# Synology NAS에 Jellyfin 설치하기 — Plex 없이 미디어 서버 구축

![NAS 미디어 서버 Jellyfin](https://images.unsplash.com/photo-1603732351-97311a4c36c6?w=800&q=80)

Plex는 무료 버전에 제한이 많고, Emby는 유료다. Jellyfin은 완전 무료 오픈소스인데 기능은 Plex와 크게 차이 없다. Synology NAS에 Docker로 5분이면 올라간다.

이 시리즈의 이전 글에서 [Synology Container Manager 설치와 기본 Docker 사용법]({% post_url 2026-06-06-10-00-00-024690-synology-docker-install-container-run %})을 다뤘다. Docker가 준비돼 있어야 이 글을 따라할 수 있다.

## 환경 정보

| 항목 | 버전/모델 |
|------|----------|
| NAS 모델 | Synology DS923+ |
| DSM 버전 | DSM 7.2.2 |
| Jellyfin 버전 | 10.9.7 |
| Docker 이미지 | jellyfin/jellyfin:latest |

## 미디어 폴더 구조 먼저 잡기

Jellyfin 올리기 전에 NAS에서 미디어 파일 폴더 구조를 정해두는 게 좋다. 나중에 바꾸면 라이브러리 재스캔해야 해서 번거롭다.

```
/volume1/
└── media/
    ├── movies/        # 영화
    ├── tv/            # TV 시리즈
    ├── music/         # 음악
    └── photos/        # 사진
```

파일 시리즈물은 시즌 폴더로 나눠두면 Jellyfin이 자동으로 에피소드 정보를 긁어온다.

```
/volume1/media/tv/
└── Breaking Bad/
    ├── Season 01/
    │   ├── S01E01.mkv
    │   └── S01E02.mkv
    └── Season 02/
```

## docker-compose로 Jellyfin 실행

```yaml
# /volume1/docker/jellyfin/docker-compose.yml
version: "3.9"
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    network_mode: host
    environment:
      - PUID=1026
      - PGID=100
      - TZ=Asia/Seoul
    volumes:
      - /volume1/docker/jellyfin/config:/config
      - /volume1/docker/jellyfin/cache:/cache
      - /volume1/media:/media
    restart: unless-stopped
```

```bash
# SSH에서 실행
cd /volume1/docker/jellyfin
sudo docker compose up -d

# 로그 확인
sudo docker compose logs -f jellyfin
```

`network_mode: host`를 쓰면 포트 매핑 없이 NAS IP로 바로 접속된다. Jellyfin 기본 포트는 8096(HTTP)과 8920(HTTPS)다.

접속 주소: `http://NAS-IP:8096`

## 초기 설정

처음 접속하면 초기 설정 마법사가 뜬다. 언어 선택 → 관리자 계정 생성 → 미디어 라이브러리 추가 순서다.

미디어 라이브러리 추가할 때 경로를 `/media/movies`, `/media/tv` 식으로 입력한다. 컨테이너 내부 경로 기준이라서 볼륨 마운트 설정과 맞춰야 한다.

![Jellyfin 미디어 라이브러리 설정](https://images.unsplash.com/photo-1451187580094-2cfb76b7c9b7?w=800&q=80)

메타데이터 제공자는 기본으로 TMDB(영화)와 TheTVDB(시리즈)가 설정돼 있다. 한국 콘텐츠는 인식률이 좀 낮은 편이라서 파일명을 영어로 맞춰두는 게 낫다.

## 하드웨어 트랜스코딩 설정

DS923+는 AMD Ryzen R1600 내장 그래픽을 활용한 하드웨어 트랜스코딩이 가능하다. 소프트웨어 트랜스코딩은 CPU를 많이 잡아먹어서 4K 파일 재생 시 버벅거리는데, 하드웨어 트랜스코딩하면 훨씬 부드럽다.

```yaml
# docker-compose.yml에 추가
services:
  jellyfin:
    # ... 기존 설정 ...
    devices:
      - /dev/dri:/dev/dri    # GPU 장치 마운트
```

Jellyfin 관리자 패널 → **대시보드 → 재생** → **트랜스코딩** 에서 하드웨어 가속을 `Video Acceleration API (VAAPI)`로 설정한다.

주의: Synology 모델마다 GPU 지원 여부가 다르다. DS923+, DS1522+처럼 AMD CPU 모델은 VAAPI가 된다. Intel 기반 모델은 Intel QSV를 써야 한다.

## 삽질했던 부분

**자막 문제**: 한국어 자막을 `.srt` 파일로 별도 보관하면 Jellyfin이 자동으로 인식한다. 파일명이 영상 파일명과 같아야 한다.

```
movie.mkv
movie.ko.srt    # 한국어 자막
movie.en.srt    # 영어 자막
```

**캐시 폴더 권한**: 처음에 캐시 폴더 권한 문제로 미디어 이미지(썸네일)가 저장이 안 됐다. SSH에서 아래 명령으로 해결했다.

```bash
sudo chown -R 1026:100 /volume1/docker/jellyfin/cache
```

**외부 접속**: 기본 설정으로는 로컬 네트워크에서만 접속된다. 외부에서 접속하려면 DDNS 설정이나 Tailscale VPN이 필요하다. 다음 편에서 DDNS와 Let's Encrypt 인증서 설정을 다룬다.

## 한 줄 정리

Jellyfin은 Docker 하나면 올라가고, 폴더 구조만 잘 잡으면 메타데이터 자동 수집까지 된다. Plex Pass 유료 결제 필요 없다.

---
다음엔 [Synology NAS 외부 접속 설정 — DDNS와 Let's Encrypt 인증서로 HTTPS 연결]({% post_url 2026-06-08-10-00-00-049380-synology-ddns-letsencrypt-external-access %})을 다룬다.

**참고 링크**
- [Jellyfin 공식 문서](https://jellyfin.org/docs/)
- [Jellyfin Docker Hub](https://hub.docker.com/r/jellyfin/jellyfin)
- [Synology 하드웨어 가속 지원 모델 목록](https://kb.synology.com/ko-kr/DSM/tutorial/What_kind_of_CPU_does_my_NAS_have)

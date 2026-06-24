---
layout: post
title: "Immich 설치 — 구글 포토 대체 자체 호스팅 사진 관리 서버"
description: "Immich를 Docker Compose로 홈서버에 설치하고 구글 포토 대체 사진 관리 서버를 구축하는 방법. 얼굴 인식, 장소 태깅, 모바일 앱 자동 백업 설정 포함."
date: 2026-06-16
tags: [Immich, 자체호스팅, Docker, 미디어서버]
comments: true
share: true
---

# Immich 설치 — 구글 포토 대체 자체 호스팅 사진 관리 서버

![Immich 사진 관리 서버 자체 호스팅](https://images.unsplash.com/photo-1603732351-97311a4c36c6?w=800&q=80)

구글 포토가 2021년부터 유료로 전환된 이후 대안을 찾는 사람이 많아졌다. Immich는 구글 포토와 UI가 비슷하고 얼굴 인식, 장소 기반 지도, 타임라인 뷰까지 지원한다. 완성도가 높은 편인데 아직 개발 중이라서 업데이트가 잦다.

이전 글에서 [Vaultwarden 비밀번호 관리 서버]({% post_url 2026-06-15-10-00-00-135795-vaultwarden-password-server-setup %})를 올렸다. 이번엔 사진 백업 서버다.

## 환경 정보

| 항목 | 버전/모델 |
|------|----------|
| NAS 모델 | Synology DS923+ |
| Immich 버전 | 1.111.x |
| Docker Compose | 2.x |

주의: Immich는 버전마다 변경이 많다. 공식 문서의 docker-compose 파일을 그대로 쓰는 걸 권장한다. 직접 수정하면 업데이트 시 충돌이 생기기 쉽다.

## 설치

공식 문서에서 제공하는 방법을 그대로 따른다.

```bash
# 설치 폴더 생성
mkdir -p /volume1/docker/immich
cd /volume1/docker/immich

# 공식 docker-compose.yml 다운로드
wget -O docker-compose.yml https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml

# 환경변수 파일 다운로드
wget -O .env https://github.com/immich-app/immich/releases/latest/download/example.env
```

`.env` 파일에서 아래 항목 수정:

```env
# 사진 저장 경로 (NAS 볼륨으로 변경)
UPLOAD_LOCATION=/volume1/photos/immich

# DB 저장 경로
DB_DATA_LOCATION=/volume1/docker/immich/pgdata

# 타임존
TZ=Asia/Seoul
```

```bash
sudo docker compose up -d
```

Immich는 여러 컨테이너로 구성된다:
- `immich-server`: 메인 API 서버 (포트 2283)
- `immich-microservices`: 썸네일 생성, 얼굴 인식 등
- `immich-machine-learning`: ML 기반 얼굴/객체 인식
- `postgres`: 메타데이터 데이터베이스
- `redis`: 캐시

접속: `http://NAS-IP:2283`

## 초기 설정

첫 접속 시 관리자 계정을 만든다. **관리 → 사용자 관리**에서 가족 계정을 추가할 수 있다.

**스토리지 템플릿 설정**: 업로드된 사진이 어떤 폴더 구조로 저장될지 설정한다.

```
관리 → 설정 → 스토리지 템플릿
예: {{y}}/{{MMMM}} → 2024/June 폴더로 저장
```

## 모바일 앱으로 자동 백업

iOS/Android 앱 스토어에서 **Immich** 앱 설치.

서버 URL 입력: `http://NAS-IP:2283` (외부에서 접속하려면 HTTPS 필요)

**백업 설정**:
```
앱 설정 → 백업 → 자동 백업 활성화
백업 앨범 선택: 카메라 롤 전체
Wi-Fi에서만 백업: 활성화 (모바일 데이터 절약)
```

처음 백업은 사진 수에 따라 오래 걸린다. 10,000장이면 몇 시간 걸린다.

![Immich 사진 타임라인 뷰](https://images.unsplash.com/photo-1451187580094-2cfb76b7c9b7?w=800&q=80)

## 얼굴 인식 및 ML 기능

Immich Machine Learning 컨테이너가 얼굴 인식, 객체 인식(CLIP 모델)을 처리한다. 자동으로 사진을 분석해서 사람별로 묶어주고, "자동차", "바다", "음식" 같은 키워드로 검색이 된다.

DS923+에서는 CPU로 처리하는데 사진이 많으면 처음 스캔에 시간이 꽤 걸린다. AMD iGPU를 ML에 쓸 수 없어서 GPU 가속은 안 됨. Nvidia GPU 달린 서버라면 CUDA 가속이 가능하다.

## 업데이트 방법

Immich는 업데이트가 잦고, 주요 변경이 많다. 업데이트 전에는 반드시 데이터베이스 백업을 해야 한다.

```bash
cd /volume1/docker/immich

# DB 백업
sudo docker exec -t immich_postgres pg_dumpall -c -U postgres > /volume1/backup/immich-db-$(date +%Y%m%d).sql

# 이미지 업데이트
sudo docker compose pull
sudo docker compose up -d
```

## 삽질했던 부분

**ML 컨테이너가 메모리를 많이 먹음**: 얼굴 인식 모델을 로딩하면 RAM을 1~2GB 쓴다. DS923+의 8GB 중 상당 부분을 가져가니, 다른 컨테이너도 많이 올려두면 스왑이 발생할 수 있다. ML 컨테이너를 끄면 얼굴 인식 기능이 동작하지 않지만 메모리는 아낄 수 있다.

**외부 라이브러리 스캔**: 이미 NAS에 있는 사진을 Immich에 가져오려면 **외부 라이브러리** 기능을 쓴다. 파일을 복사하지 않고 원본 위치에서 인덱싱만 한다. 설정에서 외부 라이브러리 경로를 `/volume1/media/photos` 식으로 지정하고 컨테이너 볼륨 마운트도 맞춰줘야 한다.

**사진 삭제 동기화**: 앱에서 사진 삭제해도 서버에서 영구 삭제는 30일 후에 된다. 구글 포토와 동일한 동작이다.

## 한 줄 정리

구글 포토 UI를 그대로 원하는데 클라우드에 사진 올리기 싫다면 Immich가 현재 최선이다. 다만 아직 개발 중이라서 업데이트 주의가 필요하다.

---
다음엔 [Uptime Kuma로 내 서비스들 모니터링하기 — 다운타임 알림 설정]({% post_url 2026-06-17-10-00-00-160485-uptime-kuma-service-monitoring %})을 다룬다.

**참고 링크**
- [Immich 공식 사이트](https://immich.app)
- [Immich 설치 공식 가이드](https://immich.app/docs/install/docker-compose)
- [Immich GitHub](https://github.com/immich-app/immich)

---
layout: post
title: "Paperless-ngx Docker 설치 — NAS에 문서 자동 분류 서버 구축하기"
description: "Paperless-ngx v2.20을 Docker Compose로 설치해 NAS·홈서버에서 PDF·스캔 문서를 OCR 검색 가능하게 만드는 방법. 한국어 OCR과 AI 자동 분류 설정까지 다룬다."
date: 2026-06-29
tags: [자체호스팅, Paperless, Docker, NAS설정, 홈서버]
comments: true
share: true
---

# Paperless-ngx Docker 설치 — NAS에 문서 자동 분류 서버 구축하기

![NAS 홈서버에서 문서 디지털 관리하는 모습](https://images.unsplash.com/photo-1568667256531-6bc3ac0cdb32?w=800&q=80)

공과금 고지서, 세금계산서, 계약서, 의료 기록. 이런 문서들을 폴더에 쌓아두다가 "그거 언제 받은 건데?" 하고 한참 뒤지는 경험, 한 번쯤 있을 거다. Paperless-ngx를 NAS에 올려두면 스캔하거나 PDF 저장하는 순간 OCR로 텍스트를 추출하고, 자동으로 태그·분류까지 달아준다. 검색창에 "2025 종합소득세" 치면 바로 나온다.

## 환경 정보

| 항목 | 버전/모델 |
|------|----------|
| Paperless-ngx | v2.20.11 |
| Docker Compose | v2.x |
| PostgreSQL | 18 |
| Redis | 8 |
| 테스트 환경 | Synology DS923+ / DSM 7.2.2, Ubuntu Server 24.04 홈서버 |

Docker Compose만 있으면 Synology, QNAP, Proxmox VM, 미니PC 어디서든 동일하게 설치된다.

## 왜 이게 필요한가

처음엔 그냥 폴더 구조 잘 만들면 되겠지 싶었다. `2026/세금/` 이런 식으로. 근데 6개월 지나면 어느 폴더에 있는지 모른다. 검색도 파일명만 되지 내용은 안 된다.

Paperless-ngx의 핵심은 **소비(consume)** 개념이다. 지정한 폴더에 파일을 던져두면 알아서 OCR → 태그 → 분류 → 이동까지 처리한다. 한 번 설정해두면 이후엔 그냥 스캔만 하면 된다.

## Docker Compose 설치

Paperless-ngx는 단일 컨테이너가 아니다. 최소 5개 서비스가 같이 올라간다.

```
paperless     ← 웹 UI + API + 문서 소비 워커
postgresql    ← 메타데이터 저장
redis         ← 작업 큐
gotenberg     ← 문서 변환 (Word, Excel → PDF)
tika          ← 텍스트 추출 보조
```

공식 GitHub에서 docker-compose 파일을 받는다.

```bash
# 설치 디렉토리 만들기
mkdir -p ~/paperless && cd ~/paperless

# 공식 compose 파일 다운로드
curl -O https://raw.githubusercontent.com/paperless-ngx/paperless-ngx/main/docker/compose/docker-compose.postgres-tika.yml
curl -O https://raw.githubusercontent.com/paperless-ngx/paperless-ngx/main/docker/compose/.env.example

# 환경 파일 복사
cp .env.example .env
```

`.env` 파일에서 아래 항목을 반드시 수정한다.

```bash
# .env
PAPERLESS_SECRET_KEY=여기에_랜덤_64자리_문자열
PAPERLESS_TIME_ZONE=Asia/Seoul
PAPERLESS_OCR_LANGUAGE=kor+eng
USERMAP_UID=1000
USERMAP_GID=1000
```

`PAPERLESS_SECRET_KEY`는 Django 시크릿 키다. 짧게 쓰면 경고 뜨니까 길게 넣는다.

```bash
# 시크릿 키 생성 (openssl 있으면)
openssl rand -base64 48
```

그 다음 컨테이너를 올린다.

```bash
docker compose -f docker-compose.postgres-tika.yml up -d
```

첫 실행에 이미지 다운로드 + DB 초기화까지 2~3분 걸린다. `docker compose logs -f paperless-ngx` 로 진행 상황 볼 수 있다.

## 관리자 계정 생성

컨테이너가 올라가면 관리자 계정을 만든다.

```bash
docker compose -f docker-compose.postgres-tika.yml exec paperless-ngx python3 manage.py createsuperuser
```

이후 `http://NAS_IP:8000` 접속하면 로그인 화면이 나온다.

![Paperless-ngx 대시보드 - 문서 검색 및 분류 화면](https://images.unsplash.com/photo-1554224155-8d04cb21cd6c?w=800&q=80)

## 한국어 OCR 설정

`.env`에 `PAPERLESS_OCR_LANGUAGE=kor+eng` 넣는 걸로 기본 설정은 끝난다. Paperless-ngx가 내부적으로 Tesseract를 쓰는데, 한국어 언어팩이 포함된 이미지를 기본으로 쓴다.

근데 여기서 함정이 있다. 스캔 품질이 낮으면 한국어 OCR이 엉망으로 나온다. 해결책은 OCR DPI 설정이다.

```bash
# .env 추가
PAPERLESS_OCR_USER_ARGS={"optimize": 1, "pdfa": 2}
```

스캔 원본이 150DPI 이하면 텍스트 인식이 거의 안 된다. 300DPI 이상으로 스캔하는 게 훨씬 낫다.

## 자동 분류 설정

Paperless-ngx는 기계학습으로 태그와 문서 유형을 자동 분류한다. 처음엔 아무것도 안 되는 것처럼 보이는데, 문서를 10~20개 정도 수동으로 분류해주면 패턴을 학습해서 그 다음부터는 알아서 태그를 단다.

관리자 페이지에서:
1. **태그** → 자주 쓰는 태그 미리 만들기 (세금, 보험, 의료, 공과금 등)
2. **문서 유형** → 청구서, 계약서, 영수증 등
3. **발신자** → 회사명, 기관명 등

초반에 20개만 분류해두면 그 다음부터 거의 다 맞춘다.

## Consume 폴더 연결

문서를 던지면 자동으로 소비되는 폴더다. Synology 기준으로 SMB 공유 폴더를 Consume 폴더로 마운트하면 스마트폰 스캐너 앱에서 바로 저장할 수 있다.

```yaml
# docker-compose.yml volumes 섹션
volumes:
  - /volume1/paperless/consume:/usr/src/paperless/consume
  - /volume1/paperless/data:/usr/src/paperless/data
  - /volume1/paperless/media:/usr/src/paperless/media
  - /volume1/paperless/export:/usr/src/paperless/export
```

Synology에서는 `/volume1/paperless/consume` 폴더를 SMB로 공유해두면 된다. iPhone의 [Scanner Pro](https://apps.apple.com/app/scanner-pro/id333710667) 같은 앱에서 스캔 후 SMB 저장하면 자동으로 처리된다.

## 삽질했던 부분

**1. 권한 문제로 파일이 안 소비됨**

Consume 폴더에 파일을 넣어도 아무 반응이 없는 경우가 있다. 거의 다 권한 문제다.

```bash
# paperless 컨테이너 내부 UID 확인
docker exec paperless-ngx id

# 호스트 폴더 권한 맞추기
chown -R 1000:1000 /volume1/paperless/
```

USERMAP_UID를 실제 파일 소유자 UID와 맞춰야 한다.

**2. PostgreSQL 버전 18 호환성**

공식 compose 파일이 postgres:18을 쓰는데, 구형 Synology에서 Docker 버전이 낮으면 이미지를 못 당겨오는 경우가 있다. 이럴 땐 postgres:16으로 낮춰도 동작한다.

**3. Gotenberg 메모리 사용량**

Word 파일 변환을 담당하는 Gotenberg가 생각보다 메모리를 많이 먹는다. Word 문서 안 쓰면 compose 파일에서 빼도 된다. PDF만 쓰면 Tika도 선택사항이다.

```yaml
# docker-compose.yml에서 아래 두 서비스 제거 가능 (PDF만 쓰는 경우)
# gotenberg: ...
# tika: ...
```

RAM이 4GB 이하 환경에서는 이렇게 경량화하는 게 낫다.

## 한 줄 정리

PDF, 스캔 이미지, Word 파일을 지정 폴더에 넣으면 OCR + 자동 분류까지 처리해주는 문서 서버가 NAS에 올라간다. 공과금 고지서부터 의료 기록까지 "찾기" 버튼 하나로 꺼낼 수 있게 된다.

---

다음엔 Paperless-ngx에 메일 자동 수집을 연결하는 방법을 다룬다. 특정 메일함으로 오는 PDF 첨부파일을 자동으로 소비 폴더에 넣는 설정이다.

관련 글:
- [{% post_url 2026-06-14-10-00-00-123450-nextcloud-install-self-hosted-cloud %} — Nextcloud로 내 서버에 클라우드 만들기]
- [{% post_url 2026-06-29-10-00-00-382695-portainer-2394-lts-upgrade-homelab-docker %} — Portainer로 Docker 컨테이너 관리하기]

**참고 링크**
- [Paperless-ngx 공식 GitHub](https://github.com/paperless-ngx/paperless-ngx)
- [Paperless-ngx 공식 문서](https://docs.paperless-ngx.com)
- [Docker Compose 공식 postgres-tika 예제](https://github.com/paperless-ngx/paperless-ngx/tree/main/docker/compose)

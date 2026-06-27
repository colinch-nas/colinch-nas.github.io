---
layout: post
title: "Immich 3.0 업그레이드 가이드 — Docker 설치 기준 Breaking Changes와 새 기능 정리"
description: "Docker Compose로 운영 중인 Immich를 v3.0으로 업그레이드하는 방법을 단계별로 정리했다. API Breaking Changes 주의사항과 Workflows·HLS 트랜스코딩 등 핵심 새 기능을 함께 다룬다."
date: 2026-06-28
tags: [Immich, 자체호스팅, Docker, 홈서버, NAS설정]
comments: true
share: true
---

# Immich 3.0 업그레이드 가이드 — Docker 설치 기준 Breaking Changes와 새 기능 정리

![Immich 사진 관리 서버 업그레이드](https://images.unsplash.com/photo-1516259762381-22954d7d3ad2?w=800&q=80)

Immich 3.0이 RC(Release Candidate) 단계에 진입했다. 기존 v2.x에서 업그레이드하기 전에 반드시 알아야 할 Breaking Changes가 있고, 업그레이드 방법도 평소와 약간 다르다. 특히 제3자 도구로 Immich API를 연동해서 쓰고 있다면 업그레이드 전에 마이그레이션 노트를 반드시 확인해야 한다.

## 환경 정보

| 항목 | 버전/모델 |
|------|----------|
| Immich | v2.x → v3.0 |
| 설치 방식 | Docker Compose (리눅스 홈서버 / Synology NAS) |
| 운영체제 | Ubuntu 24.04 / DSM 7.2.2 |

## 업그레이드 전에 반드시 해야 할 것

솔직히 말하면, Immich 2.x까지는 `docker compose pull && docker compose up -d` 한 번으로 업데이트가 끝났다. 3.0은 그게 안 통한다. 메이저 버전 점프라 구성 파일부터 확인해야 한다.

### 1. 백업 먼저

업그레이드 전에 라이브러리와 DB 모두 백업한다. 마이그레이션 실패 시 롤백 경로가 이것밖에 없다.

```bash
# 컨테이너 잠깐 멈추고 postgres 덤프
docker compose stop

# PostgreSQL 백업
docker exec immich_postgres pg_dumpall -U postgres > immich_backup_$(date +%Y%m%d).sql

# 미디어 라이브러리 백업 (rsync or cp)
rsync -av /path/to/immich/library /path/to/backup/
```

### 2. docker-compose.yml 버전 변수 확인

Immich는 `docker-compose.yml`에서 이미지 버전을 변수로 관리한다. 3.0 이전 구성 파일을 그냥 쓰면 v2 이미지를 그대로 당겨온다.

```yaml
# .env 파일 확인
IMMICH_VERSION=v3.0.0  # v2.x에서 이걸 바꿔야 함
```

공식 Immich 저장소에서 최신 `docker-compose.yml`과 `.env` 파일을 다시 받아서 비교하는 걸 권장한다. 3.0에서 컨테이너 구성 자체가 바뀐 부분이 있다.

```bash
# 최신 구성 파일 받기
wget -O docker-compose.yml https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
wget -O .env https://github.com/immich-app/immich/releases/latest/download/example.env
```

기존 `.env`에서 `DB_PASSWORD`, `UPLOAD_LOCATION` 같은 값들을 새 `.env`에 복사해 넣어야 한다. 처음 설치 때 커스텀했던 설정들 빠뜨리기 쉬우니 diff로 비교해서 확인한다.

```bash
diff .env.backup .env
```

## 업그레이드 실행

```bash
# 기존 스택 중단
docker compose down

# 새 이미지 당겨오기
docker compose pull

# 스택 재시작
docker compose up -d

# DB 마이그레이션 로그 확인
docker compose logs immich-server | grep -i migration
```

마이그레이션이 자동으로 돌아간다. 로그에서 `Successfully ran X migrations`가 뜨면 완료다. 이 과정이 DB 크기에 따라 몇 분 걸릴 수 있다. 라이브러리가 크면 5~10분은 각오해야 한다.

![Docker Compose 컨테이너 관리 화면](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=800&q=80)

## Breaking Changes — 이거 모르면 연동 서비스 다 깨진다

Immich 3.0의 Breaking Changes는 주로 API 엔드포인트 변경이다. 웹 UI와 공식 앱만 쓰는 사람은 체감 못 할 수도 있는데, 아래 도구를 연동해서 쓰고 있다면 반드시 확인해야 한다.

- **Immich Go** (CLI 백업 도구)
- **Immich Duplicati 플러그인**
- **커스텀 스크립트로 API 직접 호출하는 경우**
- **n8n / Zapier 연동**

API 주요 변경 사항:
- `/api/asset` 일부 엔드포인트 경로 변경
- 응답 스키마에서 deprecated 필드 제거
- OAuth2 스코프 체계 조정

자세한 변경 목록은 [공식 GitHub 마이그레이션 노트](https://github.com/immich-app/immich/releases)에서 확인한다.

## 3.0에서 추가된 주요 기능

### Workflows — 라이브러리 자동화

가장 기대되는 기능이다. 트리거·필터·액션을 조합해서 라이브러리 자동화 규칙을 만들 수 있다. 드래그 앤 드롭 에디터나 JSON 정의 방식 모두 지원한다.

예를 들면 "스마트폰으로 찍은 사진 중 얼굴이 있는 것만 'Family' 앨범에 자동 추가" 같은 규칙을 만들 수 있다. 아직 Preview 단계라 실험적 기능이지만 실제로 써보니 꽤 안정적이었다.

### HLS 실시간 트랜스코딩

기존에는 Immich가 동영상을 미리 트랜스코딩해둔 파일로만 재생했다. 3.0부터 업로드 시점이 아닌 재생 시점에 실시간으로 트랜스코딩이 가능하다. 홈 네트워크 속도가 느리거나 화질을 동적으로 조정해야 할 때 유용하다. 현재는 웹 앱에서만 동작하고 모바일은 개발 중이다.

### 모바일 비파괴 편집

모바일 앱에서 사진을 자르거나 회전할 때 원본 파일은 건드리지 않는다. 편집 레이어만 저장하는 방식이라 언제든 원본으로 되돌릴 수 있다. Google Photos 기본 편집 기능 쓰던 사람이라면 이게 빠져서 아쉬웠을 텐데, 이제 해결됐다.

### OCR — 사진 속 텍스트 검색

사진 뷰어에서 사진 안의 텍스트를 인식해서 복사할 수 있다. 영수증, 명함, 화이트보드 사진에서 텍스트를 바로 긁어낼 수 있어서 생각보다 많이 쓰게 된다.

## 삽질했던 부분

업그레이드 후 웹 UI가 뜨긴 뜨는데 사진이 안 보이는 경우가 있었다. 원인은 썸네일 재생성 작업이 백그라운드에서 돌고 있어서였다. 관리자 > 작업 메뉴에서 진행 상황을 확인할 수 있다. 이 작업은 라이브러리 크기에 따라 수십 분에서 몇 시간까지 걸린다. 업그레이드 직후 바로 서비스 정상화를 확인하려 해도 완전히 끝날 때까지 기다려야 한다.

Synology NAS에서 Docker를 쓰는 경우 Container Manager에서 스택 업데이트 기능을 쓰면 `.env` 파일을 못 읽는 경우가 있었다. SSH로 직접 `docker compose pull && docker compose up -d`를 실행하는 게 더 안정적이다.

## 한 줄 정리

v2.x에서 3.0으로 업그레이드는 `.env` 버전 변수 수정 + API 연동 도구 호환 확인 두 가지가 핵심이다. 웹 UI만 쓴다면 사실 거의 무중단으로 넘어갈 수 있다.

---

Immich 기본 설치부터 시작하고 싶다면 [{% post_url 2026-06-16-10-00-00-148140-immich-google-photos-alternative %}]({% post_url 2026-06-16-10-00-00-148140-immich-google-photos-alternative %}) 포스트를 먼저 보면 된다. 다음엔 Immich Workflows로 실제 자동화 규칙을 만드는 예제를 다룰 예정이다.

**참고 링크**
- [Immich 공식 문서 — 업그레이드 가이드](https://immich.app/docs/install/upgrading/)
- [Immich 3.0 릴리즈 노트](https://github.com/immich-app/immich/releases)
- [Immich 3.0 RC 논의 스레드](https://github.com/immich-app/immich/discussions/29100)

---
layout: post
title: "Immich v3.1.0 업그레이드 체크리스트 - NAS에서 사진 서버를 안전하게 업데이트하는 순서"
description: "Immich v3.1.0을 NAS Docker 환경에서 업데이트하는 방법과 백업, 마이그레이션, 모바일 앱·워크플로우 검증 순서를 정리한다."
date: 2026-09-01
tags: [Immich, Docker, 자체호스팅, NAS보안, 백업전략]
comments: true
share: true
---

![NAS에서 Immich 사진 서버를 업데이트하는 모습](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?w=1200&q=80)

Immich v3.1.0은 2026년 7월 29일 공개됐다. 이미 v3으로 올린 NAS라면 이번 업데이트는 `docker compose pull`만 실행하고 끝낼 일이 아니다. 사진 원본, PostgreSQL 데이터베이스, 모바일 업로드가 모두 정상인지 확인해야 실제로 업데이트가 끝난다. 내가 확인한 기준으로는 **백업 확인 → 이미지 교체 → 무결성 검사 → 모바일 업로드 테스트** 순서가 가장 덜 불안하다.

## 업데이트 전 환경

| 항목 | 기준 |
|---|---|
| NAS | Synology DS923+ 또는 Docker를 지원하는 NAS |
| 실행 방식 | Docker Compose |
| Immich | v3.x → v3.1.0 이상 |
| 보관 데이터 | `/volume1/docker/immich` 아래 업로드·DB 분리 |
| 외부 접속 | HTTPS 리버스 프록시 |

Immich 공식 문서의 v3 업데이트 방식은 `.env`의 `IMMICH_VERSION`을 바꾼 뒤 이미지를 다시 받는 방식이다. `latest`를 무작정 쓰지 말고 현재 파일을 먼저 복사해 둔다.

```bash
cd /volume1/docker/immich
cp .env .env.before-v3.1.0
cp compose.yaml compose.yaml.before-v3.1.0
grep IMMICH_VERSION .env
```

## PostgreSQL 백업을 먼저 확인한다

사진 폴더만 복사하면 앨범, 얼굴 인식, 공유 링크 정보가 빠진다. 실행 중인 DB 컨테이너 이름이 `immich_postgres`라는 전제로 덤프를 만든다.

```bash
mkdir -p backup/2026-09-01
docker exec -t immich_postgres pg_dumpall -c -U postgres \
  > backup/2026-09-01/postgres-before-v3.1.0.sql
ls -lh backup/2026-09-01/postgres-before-v3.1.0.sql
```

파일 크기가 0B면 성공한 백업이 아니다. 이 SQL 파일과 `library`, `upload`, `profile` 같은 실제 미디어 디렉터리는 다른 디스크나 외장 저장소에도 복사해야 한다.

## v3.1.0 업데이트

버전 태그를 고정한 뒤 컨테이너를 교체한다. `v3` 태그를 쓰는 환경이라면 공식 릴리스가 안정화된 뒤 자동으로 패치가 따라오지만, 업데이트 직후 로그 확인은 생략하지 않는다.

```bash
sed -i 's/^IMMICH_VERSION=.*/IMMICH_VERSION=v3/' .env
docker compose pull
docker compose up -d
docker compose ps
docker compose logs --tail=100 immich_server
```

`unhealthy`, `migration failed`, `database connection`이 보이면 컨테이너를 반복 재시작하지 말고 로그를 저장한다. DB 마이그레이션 중 강제 종료하면 복구가 더 번거로워진다.

## 화면과 앱에서 확인할 것

웹에서 관리자 메뉴의 버전이 v3.1.x인지 확인한 뒤 아래 순서로 실제 동작을 테스트한다.

| 테스트 | 성공 기준 |
|---|---|
| 최근 사진 열기 | 썸네일과 원본 다운로드가 모두 정상 |
| 앨범·공유 링크 | 기존 항목과 권한이 유지됨 |
| 모바일 업로드 | 새 사진 1장이 NAS에 생성됨 |
| 모바일 편집 | 원본을 덮어쓰지 않고 편집 취소 가능 |
| 워크플로우 | 미리보기 기능을 켠 경우 작업 로그가 남음 |

v3에서는 모바일 비파괴 편집과 Workflows(조건과 동작을 연결하는 자동화)가 추가됐다. 다만 Workflows는 아직 preview이므로 원본 삭제나 대량 이동 작업에 바로 연결하지 않는 편이 안전하다. 테스트용 앨범 하나에만 적용한다.

## 업데이트를 멈춰야 하는 경우

PostgreSQL 덤프가 없거나, 업로드 디렉터리를 NAS 단일 볼륨에만 두고 있거나, 앱에서 새 사진이 계속 대기 상태라면 업데이트를 완료로 보지 않는다. 특히 Immich는 데이터베이스와 원본 파일의 위치가 어긋나면 화면은 열려도 사진이 깨질 수 있다.

핵심은 새 기능보다 복구 가능성이다. 백업 파일의 실제 크기를 확인하고, v3.1.0 적용 후 웹·모바일·원본 다운로드까지 한 번씩 통과시키면 NAS 사진 서버 업데이트 리스크를 크게 줄일 수 있다.

참고:

- [Immich v3.0.0 공식 릴리스 노트](https://immich.app/blog/v3.0.0-release)
- [Immich 공식 릴리스 목록](https://immich.app/blog?type=release)

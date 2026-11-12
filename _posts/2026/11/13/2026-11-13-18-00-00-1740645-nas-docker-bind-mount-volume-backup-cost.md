---
layout: post
title: "NAS Docker 앱 데이터 위치 - Synology DSM 7.4·QNAP QTS 5.2 bind mount와 named volume 복원 비용"
description: "Synology DSM 7.4와 QNAP QTS 5.2에서 Docker 앱 데이터를 bind mount와 named volume 중 어디에 둘지, 백업·복원 호환성과 운영 총비용으로 판단한다."
date: 2026-11-13
tags: [Synology, QNAP, Docker, Immich, NAS, 백업]
comments: true
share: true
---

![NAS에서 Docker 앱 데이터와 백업 디스크를 분리하는 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)
*이 그림에서 볼 부분은 컨테이너 자체보다 데이터 경로와 복원 가능한 백업 위치다.*

1~2명이 Synology DSM 7.4나 QNAP QTS 5.2에서 컨테이너를 1~3개만 운영한다면, DB와 업로드 폴더를 NAS 공유 폴더에 직접 연결하는 **bind mount**가 관리·복원 비용이 낮다. 앱을 자주 교체하거나 배포를 표준화한다면 named volume이 편하지만 백업 도구와 복원 명령을 함께 남겨야 한다. 아래 내용은 실제 장비 시험이 아니라 2026년 11월 13일 공식 문서와 가상 조건을 바탕으로 했다.

## 경로 선택 기준

| 환경 | 우선 선택 | 이유 | 구매하지 않아도 되는 것 |
|---|---|---|---|
| 1~2명, Immich·Vaultwarden 1~2개 | bind mount | 파일 위치를 NAS 백업에서 바로 확인 | 앱 전용 SSD 풀 |
| 3~5명, DB 앱 3~6개 | bind mount + DB 백업 | 폴더와 SQL 덤프를 분리해 복원 | 컨테이너마다 SSD를 추가할 필요는 없음 |

Docker 공식 문서 기준 bind mount는 호스트 경로를 직접 연결하고, volume은 Docker가 관리하는 저장 영역이다. 이미지와 설정 파일만 백업해서는 데이터 백업이 되지 않는다. ([Docker 저장소 문서](https://docs.docker.com/engine/storage/), [Docker volume 백업·복원](https://docs.docker.com/engine/storage/volumes/))

## Immich에서 자주 생기는 복원 누락

Immich는 DB에 사진 경로와 메타데이터를 저장하고 원본은 별도 폴더에 둔다. 공식 문서도 DB 백업만으로는 부족하므로 `UPLOAD_LOCATION`과 DB를 함께 보관하라고 안내한다. 앱을 멈추고 복사하거나, 중지할 수 없으면 DB를 먼저 백업한 뒤 파일을 복사해야 불일치를 줄일 수 있다. ([Immich 백업·복원](https://docs.immich.app/administration/backup-and-restore/))

Compose에서는 앱 설정과 실제 데이터를 분리한다. 아래는 특정 NAS에서 직접 실행한 명령이 아닌 경로 예시다.

```yaml
services:
  immich-server:
    volumes:
      - /volume1/docker/immich/upload:/usr/src/app/upload
      - /volume1/docker/immich/library:/usr/src/app/library
      - /volume1/docker/immich/.env:/app/.env:ro
```

NAS 백업에서는 `docker-compose.yml`, `.env`, `upload`·`library`, PostgreSQL 덤프를 함께 확인한다. named volume이면 `docker volume inspect`로 실제 위치를 확인한다.

## 총비용은 SSD 가격보다 복원 시간을 계산한다

가상 사례로 4명이 Immich·Nextcloud·Vaultwarden·Jellyfin을 운영한다고 하자. DB만 SSD에 두면 SSD 고장 시 설정·덤프·Compose 파일을 함께 복원해야 한다. 비용식은 `SSD 또는 HDD + 외장 백업 디스크 + UPS + 복원 시간`이다. 앱이 1~2개이고 DB가 작다면 기존 HDD와 외장 백업으로 충분할 수 있다.

구매 전에는 다음을 기록한다.

- [ ] Compose 파일과 이미지 태그를 보관했다.
- [ ] bind mount의 호스트 경로 또는 named volume의 실제 위치를 확인했다.
- [ ] DB 덤프와 원본 업로드 파일을 같은 시점 기준으로 보관했다.
- [ ] 새 NAS에서 빈 폴더가 아니라 샘플 사진·계정·권한까지 복원해 봤다.
- [ ] `.env` 비밀번호와 암호화 키를 NAS와 다른 위치에도 보관했다.

파일을 직접 관리하고 복원 경로를 단순하게 만들 때는 bind mount, 배포 재현성과 앱 교체가 중요할 때는 named volume이 맞다. 컨테이너 재생성 성공은 백업 검증이 아니다. DB, 원본 파일, Compose 설정을 한 세트로 복원할 수 있을 때만 전용 SSD나 새 NAS 구매가 정당화된다.

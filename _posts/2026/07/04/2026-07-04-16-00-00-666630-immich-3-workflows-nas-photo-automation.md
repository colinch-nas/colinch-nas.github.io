---
layout: post
title: "Immich 3 Workflows - NAS 사진 자동화는 작은 규칙부터 켠다"
description: "Immich 3.0.0 정식 릴리스의 Workflows preview를 NAS Docker 환경에서 안전하게 시험하는 백업, 규칙 설계, 검증 순서다."
date: 2026-07-04
tags: [Immich, Docker, 자체호스팅, NAS설정]
comments: true
share: true
---

![NAS에 저장된 사진 라이브러리와 편집 작업 화면](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?w=800&q=80)
이 그림에서는 사진 앱 기능보다 원본 파일이 NAS 볼륨에 그대로 남아 있는지를 봐야 한다.

Immich 3.0.0으로 올렸다면 Workflows(사진 라이브러리에서 조건과 작업을 묶는 자동화)는 바로 써볼 만하지만, 처음 규칙은 "새로 들어온 사진에 태그 붙이기" 정도로 작게 시작하는 게 낫다. 2026년 7월 1일 공식 릴리스 기준 Workflows는 preview이고, Visual editor와 JSON editor를 함께 제공한다. NAS Docker에서는 기능보다 복구 경로를 앞에 둬야 한다.

내가 헷갈렸던 건 Workflows가 사진 파일을 직접 정리해주는 백업 도구처럼 보인다는 점이었다. 실제로는 Immich 데이터베이스와 라이브러리 상태를 기준으로 동작한다. 그래서 `/photos` 폴더를 손으로 옮기거나, 외부 스크립트가 같은 파일을 동시에 만지면 결과가 꼬일 수 있다.

## 적용 전 확인할 것

| 항목 | 확인 기준 |
|---|---|
| Immich 버전 | `v3` 또는 `v3.0.0` 태그 |
| 실행 방식 | Docker Compose |
| 백업 | DB dump와 사진 라이브러리 스냅샷 분리 |
| 실험 범위 | 새 앨범 1개, 최근 업로드 20장 이하 |

업데이트 직후라면 컨테이너 상태부터 본다.

```bash
docker compose ps
docker compose logs --tail=80 immich-server
```

Workflows를 열기 전에는 데이터베이스 백업을 남긴다. 사진 원본 백업과 DB 백업은 역할이 다르다.

```bash
docker compose exec database pg_dump -U postgres immich > immich-before-workflows.sql
```

## 첫 규칙은 삭제 없는 작업으로 만든다

웹에서 `Utilities -> Workflows`로 들어간다. 빈 workflow를 만들고, 트리거는 새 asset 추가로 둔다. 필터는 "최근 테스트 앨범에 들어온 사진"처럼 범위를 좁힌다. 액션은 삭제, 이동, 공유 링크 생성이 아니라 태그 추가나 앨범 분류처럼 되돌리기 쉬운 작업을 고른다.

처음엔 가족 전체 자동 업로드 폴더에 규칙을 걸었다가, 스크린샷과 영수증까지 같은 분류로 들어가는 걸 보고 바로 껐다. 조건을 사람 기준으로 잡기 전에는 사진 수가 적은 테스트 앨범이 훨씬 안전하다.

## JSON은 백업 파일처럼 남긴다

Immich 3 Workflows는 Visual editor로 만들 수 있지만 JSON으로 복사할 수 있다. 규칙을 바꿀 때마다 NAS의 compose 폴더 옆에 날짜를 붙여 저장한다.

```text
/volume1/docker/immich/workflows/
  2026-07-04-test-album-tagging.json
```

이렇게 해두면 UI에서 실수로 조건을 바꿔도 이전 규칙으로 돌아가기 쉽다. 특히 여러 명이 모바일 앱으로 업로드하는 집에서는 "누가 올렸는지", "어느 앨범인지", "최근 추가인지"를 분리해서 본다.

## 운영에 넣기 전 체크리스트

- 테스트 앨범에서만 10장 이상 실행됐는지 본다.
- 원본 파일 경로가 바뀌지 않았는지 확인한다.
- Workflows 로그와 `immich-server` 로그를 같이 본다.
- 삭제·공유·대량 이동 작업은 최소 일주일 뒤에 켠다.

짧게 정리하면 이렇다. Immich 3.0.0의 Workflows는 NAS 사진 정리를 자동화할 수 있는 출발점이다. 다만 preview 기능이므로 DB 백업, 작은 테스트 앨범, 삭제 없는 액션, JSON 보관 순서로 접근해야 한다. 자동화가 편해 보여도 사진 원본은 한 번 잃으면 복구 비용이 크다.

출처: [Immich v3.0.0 릴리스 노트](https://immich.app/blog/v3.0.0-release), [Immich v3 마이그레이션 가이드](https://immich.app/blog/v3-migration)

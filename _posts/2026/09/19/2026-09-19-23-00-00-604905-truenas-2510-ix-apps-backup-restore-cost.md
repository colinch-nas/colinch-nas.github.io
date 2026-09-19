---
layout: post
title: "TrueNAS SCALE 25.10 ix-apps 백업·복원 한계 - 앱 풀 교체와 저장장치 구매 기준"
description: "TrueNAS SCALE 25.10에서 ix-apps와 host path의 백업 범위를 나누고, 앱 풀 교체 때 필요한 SSD·외부 백업·복구 비용을 판단하는 기준을 정리한다."
date: 2026-09-19
tags: [TrueNAS, HomeLab, Docker, NAS설정, 백업복원]
comments: true
share: true
---

![TrueNAS SCALE 앱 풀과 백업 저장장치를 점검하는 홈서버](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

TrueNAS SCALE 25.10에서 앱을 운영한다면 앱 풀용 SSD를 사기 전에 `ix-apps`와 앱 데이터의 백업 범위를 나눠야 한다. 공식 문서 기준으로 25.10.0의 `ix-apps`는 웹 UI에서 백업·복원이 지원되지 않는다. SMB 파일만 공유하고 앱을 쓰지 않는다면 전용 SSD를 추가 구매할 이유가 적지만, Immich·Nextcloud처럼 데이터베이스와 업로드 파일이 있는 경우에는 앱별 백업까지 준비해야 한다.

## 무엇이 어디에 저장되는가

TrueNAS 24.10 이상은 앱 풀에 숨겨진 `ix-apps` 데이터셋을 만들고 Docker 설정, 카탈로그, 앱 메타데이터를 저장한다. 반면 host path로 지정한 데이터셋은 별도 위치에 남는다. 앱 풀을 바꿀 때 자동 이동을 선택해도 host path 데이터는 따라오지 않는다.

| 구성 | 앱 풀 이동 시 자동 처리 | 별도 백업 필요 | 구매 판단 |
|---|---|---|---|
| SMB만 사용 | 해당 없음 | 파일·설정 백업 | 기존 디스크와 외부 백업이면 충분 |
| 앱 + host path | 앱 정의·iXVolume 중심 | host path, DB 덤프 | 앱용 SSD보다 백업 저장소가 먼저 |
| 앱 + iXVolume 중심 | 앱 데이터 이동 가능 | 복원 절차 자체 검증 | 새 SSD를 사도 복구 테스트 필요 |
| 앱 풀 장애·교체 | 완전한 UI 복원 아님 | 시스템 설정·앱별 백업 | 예비 저장장치와 복구 시간까지 계산 |

## 구매 전에 확인할 체크리스트

1. `Apps` 화면에서 앱별 저장 경로를 적는다. `/mnt/.ix-apps`만 보고 업로드 파일과 PostgreSQL 데이터가 모두 있다고 가정하면 안 된다.
2. `System Settings`에서 시스템 설정 파일을 내려받고, 각 앱의 환경 변수·포트·마운트 경로를 별도 문서로 보관한다.
3. Nextcloud·Immich처럼 DB를 쓰는 앱은 실행 중 파일 복사만 하지 말고 앱이 안내하는 DB 덤프와 업로드 디렉터리 백업을 함께 만든다.
4. 외부 디스크나 다른 NAS로 스냅샷 복제한 뒤, 새 테스트 앱 풀에서 실제로 읽히는지 확인한다. 스냅샷은 삭제·설정 오류에는 유용하지만 NAS 전체 장애를 대신하지 않는다.
5. 새 앱 풀을 만들 때는 SSD를 우선 검토한다. TrueNAS 문서는 반복 읽기·쓰기와 앱 안정성을 위해 SSD 앱 풀을 권장한다. 단, SSD 한 장은 백업이 아니므로 예산에는 외부 백업 저장장치와 교체 시간을 같이 넣는다.

앱 풀을 이동할 때는 `Apps → Configuration → Choose a pool`에서 기존 앱 이동 옵션을 확인할 수 있다. 이 기능은 앱 데이터셋과 iXVolume에는 적용되지만 host path는 이동하지 않는다. `ix-apps`를 SMB나 NFS 공유에 억지로 노출하거나 직접 수정하는 방식도 피해야 한다. 공식 문서는 이 데이터셋이 내부 관리 대상이며, 암호화 풀에 있어도 암호화를 자동 상속하지 않는다고 안내한다.

## 총비용을 계산하는 간단한 양식

가상 예시로 앱 풀 교체 비용은 아래처럼 적는다.

```text
총비용 = 새 SSD/NVMe + 외부 백업 용량 확보 + 필요 시 UPS
       + 복구 중 서비스 중단 시간 × 업무 손실 단가
```

앱이 없거나 host path 백업이 이미 다른 NAS에 있다면 새 SSD 구매를 보류할 수 있다. 반대로 `ix-apps`에만 의존하고 DB 덤프가 없다면 SSD를 추가해도 복구 가능성이 자동으로 생기지 않는다. 2026년 9월 19일 확인 기준으로는 저장장치 구매보다 앱별 데이터 위치 기록과 복원 리허설이 먼저다.

참고 문서: [TrueNAS Apps 저장소 안내](https://apps.truenas.com/getting-started/app-storage/), [TrueNAS 25.10 Apps 문서](https://www.truenas.com/docs/scale/25.10/scaleuireference/apps/), [TrueNAS 백업 안내](https://cdn.truenas.com/docs/scale/gettingstarted/configure/setupbackupscale/)

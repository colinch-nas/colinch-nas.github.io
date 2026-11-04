---
layout: post
title: "NAS SSD 캐시 선택 - Synology DSM 7·QNAP QTS·TrueNAS 27 호환성과 총비용"
description: "2.5GbE 가정용 NAS에서 HDD, SATA SSD, NVMe 캐시를 사용자 수와 작업 유형별로 비교하고 SSD 장애·RAM·전기료까지 포함한 구매 기준을 정리한다."
date: 2026-11-05
tags: [NAS, Synology, QNAP, TrueNAS, SSD, HomeLab]
comments: true
share: true
---

![NAS에 설치하는 SSD 캐시와 HDD 저장장치](https://images.unsplash.com/photo-1597852074816-d933c7d2b988?w=1200&q=80)

그림에서 봐야 할 점은 SSD가 저장장치 전체인지, HDD 앞단의 캐시인지에 따라 호환성과 장애 비용이 달라진다는 사실이다.

2.5GbE로 사진·문서를 1~3명이 읽는 NAS라면 SSD 캐시를 사지 않고 RAM과 CMR HDD에 예산을 쓰는 편이 낫다. 반대로 여러 사용자가 작은 파일을 반복해서 읽거나 VM·DB를 돌린다면 SSD 볼륨 또는 캐시가 의미 있다. 단일 SSD를 읽기-쓰기 캐시로 넣는 선택은 피해야 한다. Synology DSM 7 계열은 읽기-쓰기 캐시에 최소 2개의 SSD가 필요하고, TrueNAS의 L2ARC는 읽기 캐시라서 SSD 장애가 풀 데이터 손실로 이어지지는 않지만 성능이 떨어질 수 있다.

## 장비별 캐시 조건

| 환경 | 기본 선택 | SSD를 추가할 조건 | 피해야 할 선택 |
|---|---|---|---|
| Synology DSM 7, 1~3명 | CMR HDD + 충분한 RAM | Btrfs 볼륨에서 반복적인 랜덤 읽기 | 지원 모델 확인 없는 NVMe 구매 |
| QNAP QTS, 2~8명 | HDD 또는 SSD 볼륨 | 호환 목록에 있는 SSD로 읽기/쓰기 캐시 | QTS와 QuTS hero 옵션을 같은 것으로 가정 |
| TrueNAS 27, ZFS | RAM + HDD/SSD 데이터 VDEV | RAM이 충분하고 작업 데이터가 RAM보다 크며 랜덤 읽기가 많은 경우 | L2ARC를 RAM 부족 해결책으로 사용 |

Synology 공식 문서 기준으로 읽기 전용 캐시는 SSD 1개부터 가능하지만, 읽기-쓰기 캐시는 2~6개 SSD와 RAID 1·5·6·10 구성이 필요하다. QNAP은 모델별 호환 목록을 확인해야 하며 QTS와 QuTS hero의 캐시 동작도 다르다. TrueNAS L2ARC는 ZFS 풀의 읽기 캐시이고 캐시 VDEV는 미러가 아닌 스트라이프다.

## 구매 전 계산은 속도보다 실패 비용부터다

가상 사례로 4베이 NAS에 12TB HDD 2개를 미러로 쓰고, 2.5GbE에서 3명이 사진을 공유한다고 하자. 이때 인터넷·SMB·인덱싱이 병목이면 SSD 2개와 UPS를 사도 체감이 작다. RAM을 늘리거나 사진 데이터베이스만 SSD 볼륨으로 옮기는 편이 낫다.

5~8명이 작은 파일과 VM을 동시에 사용한다면 캐시보다 SSD 2개로 별도 데이터 볼륨을 구성하는 편이 관리가 쉽다. 계산은 아래처럼 단순화할 수 있다.

`4년 총비용 = SSD 구매비 + (평균전력 W × 24 × 365 × 4 ÷ 1000 × kWh 단가) + UPS·교체비`

전력 단가와 실제 소비전력은 지역·장비·부하에 따라 다르므로 구매일의 값을 넣는다. SSD 2개 가격이 HDD 1개 추가보다 높고 사용자가 3명 이하라면 캐시 구매를 보류할 근거가 충분하다.

## 복사 가능한 호환성 체크리스트

- [ ] NAS 모델의 SSD 캐시 지원 여부와 SATA/NVMe 슬롯을 호환 목록에서 확인했다.
- [ ] 읽기 전용인지 읽기-쓰기인지 정했고, 후자라면 SSD 2개 이상과 허용 RAID를 확인했다.
- [ ] Synology는 Btrfs 볼륨, QNAP은 QTS·QuTS hero 차이, TrueNAS는 ZFS 풀의 L2ARC인지 적었다.
- [ ] RAM 사용량과 랜덤 읽기 비율을 7일 이상 관찰했다.
- [ ] 캐시 SSD 고장, 정전, 펌웨어 교체 때 원본 데이터와 백업이 남는지 확인했다.
- [ ] 캐시를 빼도 복구 가능한지와 SSD 2개·UPS의 4년 비용을 비교했다.

Synology는 Storage Manager의 SSD Cache Advisor를 최소 7일 실행해 캐시 크기를 정하라고 안내한다. TrueNAS도 RAM이 부족한 상태의 L2ARC는 성능을 높이지 못할 수 있다고 설명한다. 구매 순서는 호환 목록 확인 → RAM·네트워크 병목 확인 → SSD 볼륨과 캐시 비교 → 복구·전기료 계산이 안전하다.

정리하면 1~3명 파일 공유는 HDD와 RAM, 반복 랜덤 읽기는 읽기 캐시, VM·DB는 별도 SSD 볼륨이 기본값이다. 캐시는 저장공간 확장이 아니며, 지원 모델과 파일 시스템을 확인하지 않은 SSD는 성능보다 비용만 늘릴 수 있다.

참고한 공식 문서: [Synology SSD Cache 생성](https://kb.synology.com/en-af/DSM/help/DSM/StorageManager/ssd_cache_create?version=7), [QNAP SSD Cache](https://www.qnap.com/en/solution/ssd-cache), [TrueNAS L2ARC](https://www.truenas.com/docs/references/l2arc/), [TrueNAS 스토리지 설정](https://cdn.truenas.com/docs/scale/gettingstarted/configure/setupstoragescale/). 문서는 2026년 11월 5일 확인했다.

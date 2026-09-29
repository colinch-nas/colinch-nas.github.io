---
layout: post
title: "NAS 용량 확장 호환성 - DSM 7.4 SHR·QTS 5.2·TrueNAS SCALE 25.10 4베이 구매 판단"
description: "4베이 NAS를 고를 때 DSM 7.4 SHR, QTS 5.2 RAID, TrueNAS SCALE 25.10 RAIDZ의 디스크 추가·교체 조건과 백업·복구 비용을 비교한다."
date: 2026-09-30
tags: [Synology, QNAP, TrueNAS, NAS, HomeLab, 저장장치]
comments: true
share: true
---

![4베이 NAS의 디스크 확장과 RAID 구성을 비교하는 저장장치](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

4베이 NAS를 지금 사서 2~3년 뒤 용량을 늘릴 계획이라면, 본체 가격보다 **디스크를 어떤 순서로 추가하거나 교체할 수 있는지**를 우선 봐야 한다. DSM 7.4에서 혼합 용량 디스크를 천천히 바꾸고 싶으면 Synology SHR이 유리하고, QTS 5.2는 기존 RAID 그룹 확장과 새 RAID 그룹 추가를 구분해야 한다. TrueNAS SCALE 25.10은 RAIDZ 확장이 가능해졌지만, 확장 중 데이터 재작성과 백업 검증을 감당해야 한다. 단순 파일 공유만 하고 백업이 없는 경우에는 어느 방식도 안전한 선택이 아니다.

## 4베이에서 갈리는 확장 방식

| 구성 | 용량을 늘리는 대표 방법 | 구매 전에 남는 제약 | 어울리는 사용자 |
|---|---|---|---|
| Synology SHR-1/2 | 빈 슬롯에 디스크 추가 또는 큰 디스크로 순차 교체 | 추가 디스크는 보통 기존 최대 용량 이상이어야 하며, 큰 디스크 교체만으로 즉시 용량이 늘지 않을 수 있음 | 디스크를 한 개씩 구매하는 가정·소형 홈랩 |
| QNAP QTS RAID 5/6 | 같은 RAID 그룹에 디스크 추가, 또는 새 RAID 그룹 추가 | 같은 HDD/SSD 유형이어야 하고, 새 그룹 하나가 고장 나면 풀 전체가 영향을 받을 수 있음 | 처음부터 같은 규격을 여러 개 사고 풀을 나눌 사용자 |
| TrueNAS SCALE RAIDZ1/2 | RAIDZ VDEV에 디스크를 한 개씩 확장하거나 새 VDEV 추가 | 확장 중 기존 데이터가 재작성되고, 기존 블록의 데이터·패리티 비율이 새 풀과 같아지지 않을 수 있음 | ZFS 스크럽·복제·복구 절차를 직접 관리할 사용자 |

Synology 공식 문서는 SHR에 추가하는 디스크가 기존 풀의 가장 큰 디스크 이상이어야 용량 낭비를 줄일 수 있다고 설명한다. RAID 5·6은 가장 작은 디스크 이상이면 추가할 수 있지만, 용량은 작은 디스크 기준으로 계산된다. [Synology 디스크 추가 조건](https://kb.synology.com/en-global/DSM/help/DSM/StorageManager/storage_pool_expand_add_disk?version=6)

QNAP QTS 5.2는 RAID 1·5·6·50·60에 같은 유형의 디스크를 추가할 수 있다고 안내한다. 큰 디스크로 바꾸는 방식은 지원되는 RAID 종류가 더 좁고, 모든 디스크를 한 개씩 교체한 뒤 용량 확장을 실행해야 한다. [QNAP RAID 그룹 확장 문서](https://docs.qnap.com/operating-system/qts/5.2.x/en-us/expanding-a-storage-pool-by-adding-disks-to-a-raid-group-FD20DB.html), [QNAP 디스크 교체 확장 문서](https://www.qnap.com/en/how-to/faq/article/how-can-i-expand-the-storage-pool-static-volume-by-replacing-the-disks-with-larger-capacity-drives)

TrueNAS의 RAIDZ 확장은 기존 데이터를 새 구성으로 읽고 다시 쓰는 작업이다. 풀은 사용 가능하지만 중간 디스크 장애가 나면 복구가 먼저다. 4개 디스크 RAIDZ2를 5개, 6개로 넓혀도 장애 허용 수는 그대로 2개이며, 새로 만든 같은 폭의 VDEV와 표시 용량이 정확히 같다고 가정하면 안 된다. [TrueNAS 풀 관리·RAIDZ 확장 문서](https://cdn.truenas.com/docs/scale/storage/pools/managepools/)

## 가상 비용으로 보는 구매 순서

다음은 실제 판매가가 아닌 판단용 가상 사례다. 4베이 본체를 이미 가지고 있고 12TB HDD 2개로 시작한다고 가정한다.

| 선택 | 지금 필요한 디스크 | 2년 뒤 확장 | 추가로 잡을 비용 |
|---|---:|---|---|
| SHR-1 | 12TB 2개 | 12TB 1~2개를 순차 추가 | 예비 디스크 1개, 외장 백업 디스크 |
| QNAP RAID 1 시작 | 12TB 2개 | RAID 5 전환 또는 새 RAID 그룹 | RAID 변경 중 백업 공간, 새 그룹 장애 위험 |
| TrueNAS RAIDZ2 시작 | 보통 4개 이상 | VDEV 확장 또는 새 VDEV | ECC·UPS·복제 대상·확장 중 여유 시간 |

가상 단가를 HDD 12TB 한 개 22만 원, 외장 백업 20만 원, UPS 12만 원으로 두면, “나중에 디스크 두 개만 추가”라는 계획도 최소 76만 원의 장치 비용이 된다. 여기에 전기료와 장애 때 임시 디스크를 빌리거나 구매하는 비용이 붙는다. 반대로 4베이를 모두 채울 계획이 없고 사용량이 4TB 이하라면, 기존 NAS를 큰 모델로 바꾸기보다 외장 HDD 두 개를 번갈아 백업하는 편이 싸고 복구도 단순할 수 있다.

## 구매 전 확장 체크리스트

- [ ] 24개월 뒤 필요한 유효 용량과 백업 용량을 따로 계산했다.
- [ ] 현재 디스크의 유형(HDD·SSD), 섹터 형식, 최대 용량을 확인했다.
- [ ] “디스크 추가”와 “모든 디스크 교체” 중 어떤 확장인지 문서로 확인했다.
- [ ] 확장 중 장애가 나도 복구할 외장 디스크나 다른 NAS가 있다.
- [ ] RAID 확장이 백업을 대신하지 않는다는 전제를 세웠다.
- [ ] 새 디스크를 넣기 전 SMART 검사, 풀 상태, 복원 가능한 백업을 확인한다.

결정 기준은 간단하다. 디스크를 서로 다른 시기에 사고 싶으면 SHR을 우선 검토한다. QNAP을 고른다면 RAID 그룹을 무작정 하나의 풀로 합치지 말고, 새 그룹 장애 시 전체 데이터가 영향을 받는지 확인한다. TrueNAS는 초기 RAIDZ 폭과 복제 대상을 먼저 정할 수 있을 때 선택한다. 세 플랫폼 모두 구매 직후 작은 파일 하나를 실제로 복원해 보고, 확장 작업 전·후에 백업 목록과 해시를 비교해야 한다.

핵심은 “4베이니까 나중에 늘리면 된다”가 아니다. **확장 방식, 같은 규격 디스크 구매 시점, 외부 복구 사본**을 한 묶음으로 계산했을 때 예산과 운영 난도가 맞는 플랫폼이 정답이다.

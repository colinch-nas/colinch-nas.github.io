---
layout: post
title: "Synology DSM 7·QNAP QTS·TrueNAS SCALE 25.10 4Kn·512e HDD 호환성 - 교체 디스크와 복구 비용"
description: "4Kn과 512e HDD를 Synology DSM 7, QNAP QTS, TrueNAS SCALE 25.10에 넣을 때 확인할 호환성·교체 조건과 가상 총비용을 정리한다."
date: 2026-10-01
tags: [Synology, QNAP, TrueNAS, TrueNASSCALE, NAS, HomeLab]
comments: true
share: true
---

![NAS에 장착할 하드디스크와 서버 저장장치](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

이 그림에서 볼 부분은 디스크 용량보다 **논리 섹터 형식과 NAS가 검증한 모델**이 교체 가능성을 좌우한다는 점이다.

12TB HDD를 새로 사는 2~4베이 사용자라면 512e를 우선 후보로 두고, 4Kn은 NAS 모델과 펌웨어의 호환 목록에 정확한 모델 번호가 있을 때만 고르는 편이 안전하다. 4Kn이 더 최신이거나 빠르다는 이유만으로 바꿀 필요는 없다. QNAP은 4Kn으로 설정해도 성능 향상이 보장되지 않는다고 안내한다. 가격보다 장애 때 바로 교체할 수 있는지가 중요한 선택 기준이다.

## 4Kn·512e·512n은 무엇이 다른가

| 형식 | 물리 섹터 | OS에 보이는 논리 섹터 | NAS 교체 판단 |
|---|---:|---:|---|
| 512n | 512B | 512B | 구형 장비 호환성이 넓지만 신제품 선택지는 줄어드는 편 |
| 512e | 4KiB | 512B로 에뮬레이션 | 기존 NAS와 교체 디스크에 가장 무난한 기준 |
| 4Kn | 4KiB | 4KiB | 모델·펌웨어가 명시적으로 지원할 때만 사용 |

Synology 호환성 목록에는 4K native(4Kn) HDD를 별도로 관리하라는 안내가 있다. 즉 같은 용량·SATA라는 이유만으로 기존 풀의 고장 디스크를 4Kn으로 바꾸면 안 된다. QNAP도 특정 HDD의 호환 목록과 NAS 모델을 함께 확인해야 하며, TrueNAS SCALE 25.10은 풀을 만든 뒤 디스크 형식을 임의로 바꾸는 문제가 아니라, 풀 생성 전 컨트롤러 직결·섹터 정렬·교체 계획을 검증하는 쪽이 안전하다.

## 장비별 구매 기준

| 환경 | 구매 기준 | 피해야 할 판단 |
|---|---|---|
| Synology DSM 7, SHR·RAID | 호환 목록의 모델 번호와 4Kn 여부 확인. 불명확하면 512e | “12TB SATA면 모두 교체 가능”이라고 가정 |
| QNAP QTS, RAID 5·6 | QNAP 호환 목록과 교체 디스크의 섹터 형식 대조 | 4Kn 전환만으로 속도가 빨라진다고 기대 |
| TrueNAS SCALE 25.10, ZFS | 같은 형식의 예비 디스크를 준비하고 풀 생성 전 테스트 | 다른 섹터 형식을 장애 때 처음 시험 |

여기서 중요한 건 원본 디스크가 512e인지 4Kn인지 `smartctl`이나 제조사 사양으로 확인하는 것이다. NAS 화면에 표시되는 용량만으로는 구분되지 않는다. 예비 디스크를 별도 보관한다면 본체와 같은 모델 또는 제조사가 호환을 보장한 모델을 선택하고, 실제 장애 복구가 아니라 빈 디스크로 교체 절차를 확인한다.

## 가상 총비용 계산

실제 판매가가 아닌 2026년 10월 1일 비교용 가정이다. 12TB 512e HDD 1개를 18만 원, 같은 용량 4Kn을 20만 원, 외장 백업 디스크를 18만 원으로 잡았다.

| 구성 | 운영 디스크 4개 | 복구용 예비 1개 | 백업 디스크 | 가상 합계 |
|---|---:|---:|---:|---:|
| 512e 운영 풀 | 72만 원 | 18만 원 | 18만 원 | 108만 원 |
| 4Kn 운영 풀 | 80만 원 | 20만 원 | 18만 원 | 118만 원 |

4Kn이 10만 원 더 비싸다는 결론이 아니라, 호환되지 않아 교체 디스크를 다시 사는 비용까지 계산해야 한다는 예시다. 백업 디스크가 없다면 두 구성 모두 복구 비용이 아니라 데이터 손실 위험을 떠안게 된다. RAID는 백업이 아니며, 형식이 맞아도 복원 테스트를 하지 않은 백업은 안전하다고 단정할 수 없다.

## 구매 전 체크리스트

- [ ] NAS 모델, DSM·QTS·TrueNAS SCALE 버전을 기록했다
- [ ] 제조사 호환 목록에서 전체 모델 번호와 4Kn·512e를 확인했다
- [ ] 기존 풀의 디스크 형식과 예비 디스크 형식이 같은가
- [ ] 호환 목록이 바뀌어도 구할 수 있는 예비 디스크가 있는가
- [ ] 외장 백업 1개와 실제 복원 확인을 비용에 넣었는가

정리하면 2~4베이 NAS의 운영 풀은 검증된 512e가 기본 선택이고, 4Kn은 호환 목록에 정확히 등록된 장비에서만 이점과 비용을 비교해야 한다. 디스크를 싸게 사는 것보다 장애가 난 날 같은 형식의 디스크로 복구를 시작할 수 있게 준비하는 편이 총비용을 줄인다.

### 출처

- [Synology HDD 호환성 목록](https://www.synology.com/en-us/compatibility?category=hdds_no_ssd_trim&change_log_p=1&search_by=products)
- [QNAP 4Kn 섹터 FAQ](https://www.qnap.com/de-de/how-to/faq/article/will-configuring-4kn-4k-native-sector-in-my-hdd-improve-the-nas-performances)
- [QNAP 드라이브 호환성 목록](https://www.qnap.com/en-us/support/drive-compatibility)
- [TrueNAS SCALE 하드웨어 가이드](https://www.truenas.com/docs/scale/25.10/gettingstarted/scalehardwareguide/)

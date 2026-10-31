---
layout: post
title: "NAS HDD CMR·SMR 선택 - Synology DSM 7.3·QNAP·TrueNAS RAID 재빌드 총비용"
description: "Synology DSM 7.3, QNAP, TrueNAS에서 NAS용 HDD를 고를 때 CMR·SMR과 호환 목록, RAID 재빌드 리스크, 예비 디스크까지 포함한 총비용 판단 기준을 정리한다."
date: 2026-10-31
tags: [Synology, QNAP, TrueNAS, 저장장치, RAID, HomeLab]
comments: true
share: true
---

![NAS RAID용 CMR 하드디스크와 저장장치 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

2~8베이 NAS의 RAID용 디스크라면 **CMR NAS HDD와 제조사 호환 목록**을 우선 고르는 편이 안전하다. 단일 디스크에 순차 백업만 한다면 SMR을 검토할 수 있지만 RAID 5·6·ZFS 풀에는 맞지 않는다. 2026년 10월 31일 공식 문서 기준이다.

## CMR·SMR 선택 기준

CMR은 덮어쓰기 성능 변화가 작다. SMR은 기록 밀도는 높지만 지속 쓰기와 RAID 재빌드가 느려질 수 있다. QNAP은 SMR이 재빌드 성능에 영향을 준다고 안내하고, TrueNAS도 ZFS 풀에 SMR을 권장하지 않는다.

| 환경 | 기본 선택 | 이유 | 구매하지 않아도 되는 것 |
|---|---|---|---|
| 1베이, 외장 백업·읽기 중심 | 검증된 단일 HDD | RAID 재빌드가 없다 | Pro 모델, 핫스페어 |
| 2베이 RAID 1, 사진·문서 | CMR NAS HDD 2개 | 한 장 장애 후 복구 시간을 줄인다 | 7200RPM 엔터프라이즈 HDD |
| 4~8베이 RAID 5/6·RAIDZ | 같은 계열 CMR HDD | 재빌드 중 지연·이탈 위험을 낮춘다 | SMR 할인 모델, 혼합 풀 |
| 24시간 CCTV 전용 | 감시용 HDD, RAID면 CMR | 연속 쓰기 기준 | 데스크톱 HDD |

WD도 RAID에는 CMR을 권장한다. 제품명만 보지 말고 정확한 모델 번호의 데이터시트에서 기록 방식을 확인해야 한다.

## 장비별로 호환 목록을 다시 보는 이유

Synology는 2025년 이후 모델과 DSM 7.3 이상에서 호환 정책이 달라졌다. 목록에 없는 HDD가 기존 마이그레이션에는 허용돼도 새 풀 생성에는 제한될 수 있으며, 이전 모델에는 소급되지 않는다.

QNAP은 호환 목록을 만들 때 핫플러그와 RAID 재빌드도 시험한다. 구매 순서는 `NAS 모델 → OS 버전 → 정확한 모델 → 호환 목록 → CMR·SMR`이다.

## 총비용은 HDD 가격만 더하지 않는다

가상 사례로 4베이 NAS에 12TB HDD 4개를 넣는다면 아래 비용을 함께 계산한다.

```text
초기비용 = NAS 본체 + CMR HDD 4개 + UPS(선택) + 예비 HDD 1개
운영비   = 전력요금(평균 W × 24 × 365 ÷ 1000 × kWh 단가)
장애비용 = 배송·교체 시간 + 복원용 백업 디스크 + 데이터 접근 중단 비용
```

SMR 4개가 싸도 재빌드 지연과 작업 중단 비용이 생기면 절약액이 줄어든다. 단일 디스크에 읽기 위주 보관만 하고 별도 백업이 있다면 비싼 CMR이나 핫스페어를 사지 않아도 된다. RAID는 백업이 아니다.

## 구매 전 체크리스트

- [ ] NAS 모델과 DSM·QTS·TrueNAS 버전을 확인했는가
- [ ] 호환 목록에서 용량까지 같은 정확한 모델을 찾았는가
- [ ] 제조사 사양서에서 CMR·SMR과 작업량 등급을 확인했는가
- [ ] 같은 풀에 CMR과 SMR을 섞지 않았는가
- [ ] RAID 재빌드 중에도 보존할 별도 백업이 있는가
- [ ] 4베이 이상이면 예비 HDD와 UPS 비용을 함께 계산했는가

짧게 정리하면 **RAID는 CMR, 단일 읽기 중심 디스크는 조건부 SMR**이다. 호환 목록과 재빌드 조건까지 확인해야 할인 가격이 실제 절약인지 알 수 있다.

참고 문서: [Synology 호환 정책](https://kb.synology.com/en-global/DSM/tutorial/Drive_compatibility_policies), [QNAP 재빌드 안내](https://www.qnap.com/en-uk/how-to/faq/article/will-the-wd-smr-based-hard-drives-affect-the-raid-rebuilding-speed), [TrueNAS 하드웨어 가이드](https://www.truenas.com/docs/scale/27/gettingstarted/tnhardwareguide/)

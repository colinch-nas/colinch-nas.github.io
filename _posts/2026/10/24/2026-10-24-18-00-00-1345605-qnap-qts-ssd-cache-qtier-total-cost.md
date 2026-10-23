---
layout: post
title: "QNAP QTS 5.2 SSD 캐시·Qtier 호환성 - 2·4베이 NAS 총비용 판단"
description: "QNAP QTS 5.2에서 SSD 캐시·Qtier·SSD 저장 풀을 고를 때 M.2 장착 방식, RAM 제한, 백업과 UPS까지 포함해 구매 여부를 판단한다."
date: 2026-10-24
tags: [QNAP, QTS, SSD, Qtier, NAS, 백업, HomeLab]
comments: true
share: true
---

![QNAP NAS의 HDD와 M.2 SSD 저장장치 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

그림에서 볼 부분은 M.2 슬롯의 존재보다 SSD가 **캐시인지 저장 풀인지**, 그리고 고장 뒤 어떤 경로로 복구하는지다.

QNAP QTS 5.2에서 1~2명이 사진·문서만 읽는다면 SSD를 사지 않는 편이 낫다. Docker·VM·DB를 3명 이상이 함께 쓰면 SSD 캐시나 저장 풀을 검토할 이유가 생긴다. “M.2면 모두 저장소”라고 가정하면 장착 뒤 용도가 막힐 수 있다. 2026년 10월 24일 공식 문서를 확인한 구매 기준이며, 특정 장비를 직접 시험한 결과는 아니다.

## 세 가지 선택은 서로 다르다

| 선택 | 원본 데이터 위치 | 맞는 환경 | 놓치기 쉬운 비용·제한 |
|---|---|---|---|
| SSD 캐시 | HDD 볼륨 | 랜덤 읽기·쓰기와 작은 파일이 반복됨 | SSD 2개, RAM, UPS, 캐시 복구 점검 |
| Qtier | HDD·SSD 혼합 저장 풀 | 데이터 접근 빈도에 따라 자동 이동이 필요함 | 이동 중 성능 변동, 풀 운영 복잡도 |
| SSD 저장 풀 | SSD 볼륨 | VM·DB·컨테이너를 빠르게 운영함 | SSD 장애 시 앱 데이터 복원, 별도 백업 |

QNAP 공식 비교표도 SSD 캐시는 랜덤 I/O, Qtier는 SSD·HDD 계층화, 올플래시 풀은 지속적인 랜덤 I/O에 맞는다고 구분한다. 영화 파일을 순차 재생하거나 1GbE가 병목이면 캐시 체감이 작다.

## 장착 방식이 호환성을 가른다

QTS 5.2 공식 표 기준으로 내부 M.2와 QNAP QM2 카드의 NVMe·SATA M.2는 캐시와 저장 풀에 사용할 수 있다. 반면 **타사 M.2-PCIe 어댑터의 NVMe는 캐시는 가능해도 저장 풀·정적 볼륨에는 쓸 수 없다.** PCIe 폼팩터 SSD도 저장 풀에는 제한이 있다.

가상 계산으로 1TB SSD 2개(각 12만 원), QM2 카드·방열 장치 10만 원, UPS 12만 원이면 약 46만 원이다. 쓰기 캐시의 전원 장애와 SSD 내구도, 외장 HDD 백업을 함께 계산해야 한다.

## 구매 전 체크리스트

- [ ] QTS 5.2.x와 NAS 모델의 캐시·Qtier·저장 풀 지원 여부를 확인했다.
- [ ] 내부 M.2, QM2, 타사 어댑터 중 장착 방식을 기록했다.
- [ ] 현재 RAM으로 계획한 캐시 용량이 공식 표의 제한 안에 있는지 확인했다.
- [ ] 읽기 전용 캐시로 충분한지, 쓰기 캐시와 UPS가 필요한지 구분했다.
- [ ] SSD 고장 때 HDD 원본 또는 앱 백업에서 복원할 수 있는지 작은 파일로 점검했다.

1~2명이 대용량 사진·영상만 다루고 1GbE가 병목이면 SSD보다 백업 디스크와 네트워크 교체가 먼저다. VM·DB 지연이 반복되면 캐시보다 SSD 저장 풀을 별도 볼륨으로 만들고 앱 데이터만 복구 대상으로 삼는 편이 쉽다.

짧게 정리하면 QTS 5.2의 SSD 캐시는 HDD 원본을 빠르게 보조하는 장치이고, Qtier는 데이터를 이동시키는 저장 풀, SSD 풀은 앱 데이터의 실제 위치다. 장착 방식이 저장 풀을 지원하지 않거나 백업·UPS 예산이 빠졌다면 SSD 구매를 미루는 편이 총비용과 복구 가능성 모두에서 유리하다.

참고: [QNAP QTS 5.2 디스크 유형 공식 문서](https://docs.qnap.com/operating-system/qts/5.2.x/en-us/disk-types-B53F91D1.html), [QNAP Qtier와 SSD Cache 비교](https://www.qnap.com/en-us/how-to/faq/article/qtier-and-ssd-cache-comparison), [QNAP SSD Cache 공식 안내](https://www.qnap.com/en/solution/ssd-cache)

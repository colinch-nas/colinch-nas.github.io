---
layout: post
title: "TrueNAS SCALE 25.10 special vdev SSD 선택 - 베이 수·파일 규모별 총비용과 복원 위험"
description: "TrueNAS SCALE 25.10에서 special vdev용 SSD가 필요한 환경과 구매하지 않아도 되는 조건을 베이 수, 파일 규모, UPS·백업 비용으로 판단한다."
date: 2026-09-29
tags: [TrueNAS, OpenZFS, SSD, HomeLab, 백업복원]
comments: true
share: true
---

![TrueNAS SCALE 25.10 저장장치와 SSD special vdev 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

그림에서 볼 부분은 HDD를 SSD로 바꾸는 문제가 아니라, 메타데이터용 SSD를 추가하면서 베이·UPS·백업 예산까지 함께 늘어난다는 점이다.

TrueNAS SCALE 25.10의 **special vdev**는 파일 위치와 할당 테이블 같은 메타데이터를 저장하고, 설정에 따라 작은 파일 블록도 맡는다. VM·컨테이너·수십만 개의 작은 파일을 자주 읽는 6베이 이상 홈랩이면 검토할 가치가 있지만, 영화·사진처럼 큰 파일을 순차 재생하는 2~4베이 NAS라면 SSD를 사지 않고 RAM과 백업에 예산을 쓰는 편이 낫다. 이 글은 실제 장비 테스트가 아니라 공식 문서와 가상 구성에 따른 구매 판단이다.

## 베이 수와 파일 규모별 판단

| 환경 | 권장 구성 | special vdev 판단 | 빠뜨리기 쉬운 비용 |
|---|---|---|---|
| 2베이, 문서·사진 중심 | HDD 미러 + 외장 백업 | 구매하지 않음 | 복원용 외장 디스크 |
| 4베이, 대용량 미디어 | RAIDZ 또는 미러 + 외부 백업 | 보통 불필요 | UPS와 백업 디스크 |
| 6베이 이상, VM·Docker·작은 파일 다수 | HDD RAIDZ2 + SSD 미러 special vdev | 조건부 추천 | SSD 2개, UPS, 여분 SSD |

TrueNAS 공식 문서는 special vdev에 데이터 vdev와 같은 수준의 장애 허용을 권장한다. SSD 한 장을 special vdev로 넣으면 성능 옵션이 아니라 풀 전체 접근성을 위협하는 단일 장애점이 된다. RAIDZ 데이터 풀에 추가한 metadata vdev는 사실상 제거할 수 없다는 점도 구매 전에 확인해야 한다. [TrueNAS 25.10 Fusion Pools 문서](https://www.truenas.com/docs/scale/25.10/scaletutorials/storage/fusionpoolsscale/)

## 가격보다 먼저 계산할 총비용

가격은 판매처와 시점에 따라 달라지므로 아래처럼 장비 가격을 직접 대입한다.

`총비용 = SSD 2개 + UPS + 외부 백업 디스크 + 교체 여분 - SSD를 사지 않을 때의 비용`

가상 사례로 6베이 장비에 HDD 4개 RAIDZ2를 만들고 SSD 2개를 special vdev 미러로 추가한다고 하자. NAS에 전용 SSD 슬롯이 없으면 SSD 2개 때문에 데이터 디스크 베이를 포기하거나 HBA·확장 케이지를 추가해야 한다. 이 경우 SSD 가격만 비교하면 안 된다. UPS가 없고 백업도 같은 NAS 안에만 있다면 special vdev보다 먼저 전원 보호와 외부 복원을 준비해야 한다. SSD에 내부 캐시가 있는 경우 UPS를 권장한다는 안내도 공식 문서에 있다.

## 복원 가능한 구성 체크리스트

- [ ] special vdev를 단일 SSD로 만들지 않았는가
- [ ] SSD 모델의 TBW, 펌웨어, NAS 호환성을 확인했는가
- [ ] 풀 설정 파일과 중요 데이터가 NAS 밖에도 있는가
- [ ] SSD 장애를 가정해 풀 접근과 교체 절차를 문서화했는가
- [ ] 분기마다 임의 파일을 다른 장치로 복원해 열어봤는가
- [ ] 전원 장애 때 UPS가 NAS를 안전하게 종료하는지 확인했는가

TrueNAS는 RAIDZ1·2·3에서 패리티 디스크 수만큼 장애를 허용하지만, 이는 백업이 아니다. 공식 저장장치 문서도 stripe는 디스크 하나만 고장 나도 데이터를 잃는다고 설명한다. special vdev를 넣는다면 같은 돈으로 백업 디스크와 복원 시간을 먼저 확보했는지 비교해야 한다. [TrueNAS 저장장치 구성 문서](https://www.truenas.com/docs/scale/25.10/gettingstarted/configure/setupstoragescale/)

짧게 판단하면 **2~4베이·대용량 미디어·백업 예산이 부족한 환경은 구매하지 않는 쪽**, **6베이 이상·작은 파일과 VM이 많고 SSD 미러·UPS·외부 백업까지 감당할 수 있는 환경은 도입 검토**가 맞다. 성능 향상만 보고 SSD 한 장을 넣는 선택은 피해야 한다.

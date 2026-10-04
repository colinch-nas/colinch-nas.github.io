---
layout: post
title: "NAS CMR·SMR 디스크 호환성 - 2·4·8베이별 백업·복구 총비용"
description: "NAS용 CMR과 SMR 하드디스크를 2·4·8베이와 TrueNAS SCALE 25.10 기준으로 비교하고, 구매비보다 큰 resilver·백업 복구 비용을 계산한다."
date: 2026-10-05
tags: [NAS, TrueNAS, Synology, QNAP, 저장장치, 백업]
comments: true
share: true
---

![NAS용 CMR·SMR 하드디스크와 백업 저장장치 선택](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

그림에서 볼 부분은 디스크 단가보다 장애 때 다시 쓰는 시간과 백업 장비 비용이 판단을 바꾼다는 점이다.

활성 풀에 넣을 디스크라면 2베이든 8베이든 CMR을 우선 선택하는 편이 안전하다. 특히 TrueNAS SCALE 25.10의 ZFS 풀, Docker·VM, 스냅샷·스크럽을 함께 쓰는 환경에는 SMR을 싸다는 이유로 섞지 않는 것이 좋다. 사진을 한 번 저장하고 거의 지우지 않는 단일 외장 백업 디스크라면 제조사와 파일시스템이 허용하는 범위에서 SMR을 검토할 수 있지만, 그 디스크를 RAID 복구용으로 생각하면 안 된다. 이 글은 장비를 직접 시험한 결과가 아니라 2026년 10월 5일 확인한 공식 문서와 가상 비용으로 판단한 기준이다.

## CMR과 SMR의 차이가 NAS에서 커지는 순간

SMR은 트랙을 겹쳐 기록해 같은 용량을 낮은 비용으로 만들 수 있지만, 작은 블록을 반복해서 덮어쓸 때 내부 재배치가 발생한다. TrueNAS 공식 하드웨어 가이드는 DM-SMR·HA-SMR이 긴 resilver와 낮은 쓰기 성능을 만들 수 있어 권장하지 않으며 HM-SMR은 지원하지 않는다고 설명한다. [TrueNAS SCALE 25.10 하드웨어 가이드](https://www.truenas.com/docs/scale/25.10/gettingstarted/scalehardwareguide/)

| 환경 | 우선 선택 | SMR을 고려할 수 있는 조건 | 빠뜨리기 쉬운 비용 |
|---|---|---|---|
| 2베이, 미러·SHR, 문서·사진 | CMR | 단일 외장 백업 디스크, 쓰기 빈도 낮음 | 교체 중 백업 중단 시간 |
| 4베이, RAIDZ·RAID5·컨테이너 | CMR | 활성 풀에는 사실상 제외 | 예비 CMR 1개, UPS, 외부 백업 |
| 8베이, VM·스냅샷·다중 사용자 | CMR 또는 엔터프라이즈 CMR | 콜드 아카이브 전용으로만 검토 | 긴 resilver 동안의 장애 노출과 복구 인력 |

Synology DSM이나 QNAP QTS에서 “인식된다”는 사실은 장시간 재구축 성능까지 보장하지 않는다. 모델별 호환성 목록과 디스크 제조사의 CMR·SMR 표기를 함께 확인해야 한다. Seagate도 제품군별 기록 방식을 별도 표로 제공하므로, 판매 페이지의 “NAS용” 문구만으로 판단하지 않는다. [Seagate CMR·SMR 목록](https://www.seagate.com/products/cmr-smr-list/)

## 구매비가 아니라 총비용으로 계산하기

가상 사례로 4TB 디스크 4개를 산다고 하자. CMR 한 개를 15만 원, SMR 한 개를 11만 원으로 가정하면 초기 차이는 16만 원이다. 그러나 SMR 풀에서 교체·resilver가 길어져 예비 디스크와 외부 백업을 추가하면 차이는 쉽게 사라진다.

`총비용 = 본체 + 활성 풀 디스크 + 예비 디스크 + 외부 백업 용량 + UPS + 복구 중 운영 중단 비용`

TrueNAS는 교체 디스크가 기존 디스크와 같거나 커야 하고, 교체 뒤 자동으로 resilver를 시작한다. 큰 데이터셋에서는 오래 걸리므로 한 디스크의 resilver가 끝나기 전에 다음 디스크를 바꾸면 안 된다. [TrueNAS 디스크 교체 문서](https://www.truenas.com/docs/scale/25.10/scaletutorials/scaletutorialsprint/)

## 구매 전 체크리스트

- [ ] 제조사 데이터시트에서 CMR·SMR 방식을 확인했는가
- [ ] 활성 풀의 모든 디스크가 같은 쓰기 workload를 감당하는가
- [ ] 고장 시 즉시 넣을 같은 용량 이상의 예비 CMR이 있는가
- [ ] NAS 밖에 복원 가능한 백업이 있고 최근 복원 확인 날짜가 있는가
- [ ] 디스크 교체·resilver 동안 UPS와 여유 시간이 있는가
- [ ] SMR을 쓴다면 RAID가 아닌 저쓰기 콜드 아카이브로 격리했는가

짧게 정리하면, 활성 NAS 풀은 CMR을 기준으로 잡고 절약하려면 디스크가 아니라 베이 수와 백업 구조를 우선 줄여야 한다. SMR은 “NAS에 꽂히는 디스크”일 수는 있어도 “장애 때 빠르게 복구되는 디스크”라는 뜻은 아니다. 예산이 부족한 2베이라면 SMR 풀보다 CMR 미러와 별도 백업을 작은 용량으로 시작하는 편이 복구 비용을 예측하기 쉽다.

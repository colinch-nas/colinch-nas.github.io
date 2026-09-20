---
layout: post
title: "Synology DSM 7.4 SSD 캐시 TBW·PLP 호환성 - RAM과 복구 비용까지 계산하는 기준"
description: "Synology DSM 7.4 SSD 캐시를 살지 말지 판단하는 글이다. 2베이·4베이 NAS의 RAM, TBW·PLP, 읽기·쓰기 패턴, UPS와 복구 비용을 함께 비교한다."
date: 2026-09-20
tags: [Synology, DSM, SSD캐시, NAS보안, HomeLab]
comments: true
share: true
---

![Synology NAS SSD 캐시와 하드디스크 저장소](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)
이 그림에서 볼 것은 SSD를 추가하는 것보다 원본 HDD, 캐시 SSD, 백업 장치를 서로 다른 역할로 나누는 구조다.

Synology DSM 7.4 계열에서 작은 파일이 많은 VM·데이터베이스·여러 사용자의 SMB 작업이 느리다면 SSD 캐시를 검토할 만하다. 반대로 사진 원본을 순차 재생하거나 1GbE가 병목인 1~2인 가정 NAS라면 SSD를 사지 않는 편이 낫다. 캐시는 백업이 아니며, 호환 목록과 RAM·전원 장애 조건을 확인하지 않고 사면 비용만 늘어난다.

## 용도별 구매 판단

| 환경 | 캐시 기대 효과 | 구매 판단 |
|---|---|---|
| 사진·영상 순차 읽기, 사용자 1~2명 | 작음 | HDD와 백업 디스크 우선 |
| Docker DB·Nextcloud 작은 파일 | 있음 | SSD 캐시 또는 SSD 볼륨 비교 |
| VM·iSCSI·여러 사용자 동시 작업 | 큼 | 읽기/쓰기 캐시를 검토 |
| 1GbE 공유기·스위치 | 네트워크에서 제한 | 캐시보다 네트워크 교체 비용부터 계산 |

Synology 공식 백서는 읽기 전용 캐시는 SSD 1개로 만들 수 있지만, 읽기·쓰기 캐시는 같은 종류의 SSD 2개 이상이 필요하다고 설명한다. DSM은 SSD 캐시 1GB마다 약 400KB의 시스템 메모리를 사용하고, 기본 메모리의 최대 25%만 캐시 생성에 쓴다. 1TB 캐시라면 계산상 약 400MB가 필요하지만, 실제 지원 상한은 모델·CPU·호환 목록에 따라 달라진다. [Synology SSD Cache 백서](https://global.download.synology.com/download/Document/Software/WhitePaper/Os/DSM/All/enu/Synology_SSD_Cache_White_Paper_enu.pdf)

## SSD 사양에서 TBW와 PLP를 보는 이유

캐시는 자주 읽는 블록을 복사하는 기능이라 용량이 크다고 빨라지는 것이 아니다. Synology는 실제로 자주 접근하는 데이터보다 약간 큰 캐시를 권장하고, SSD Cache Advisor는 7~30일 사용 패턴을 분석한다. 순차 접근이나 단일 영상 스트리밍에서는 성능 향상이 작다고 명시한다.

쓰기 캐시는 전원 장애 때 아직 HDD에 내려가지 않은 데이터를 다루므로 SSD의 내구도(TBW/DWPD)와 정전 보호(PLP)를 함께 본다. 소비자용 SSD 두 개가 싸 보여도, 쓰기량이 많은 Docker DB를 매일 돌리는 환경에서는 교체 주기와 복구 작업이 총비용에 들어간다. UPS가 캐시 SSD의 PLP를 대신하는 것도 아니다. UPS는 정전 시 정상 종료 시간을 벌고, PLP는 SSD 내부에서 이미 받은 데이터를 안전하게 처리하는 조건이다.

## 가상 총비용 계산

실제 판매 가격이 아닌 비교용 가상 사례다. 4베이 NAS에 1TB SSD 2개를 캐시로 추가한다고 가정하면 비용은 `SSD 2개 + 호환 어댑터/트레이 + UPS + 3년 내 교체 가능성`이다. 예산이 45만 원이라면 같은 돈으로 백업용 HDD, 예비 디스크, UPS 중 빠진 항목을 채울 수 있는지 비교한다. 캐시를 끄거나 SSD 한 개가 고장 났을 때도 원본 볼륨과 백업에서 복구되는지 확인하지 못했다면 구매를 미룬다.

QNAP도 SSD 캐시 용량을 RAM과 연결하고, QTS·QuTS hero의 캐시 방식과 요구 메모리가 다르다고 안내한다. 제조사가 달라도 “M.2 슬롯이 있다 = 캐시가 호환된다”로 판단하면 안 된다. [QNAP SSD Cache 공식 표](https://www.qnap.com/en/solution/ssd-cache)

## 구매 전 체크리스트

- [ ] NAS 모델의 DSM 7.4 사양과 SSD 호환 목록에 모델이 등록됐는가
- [ ] 읽기 전용 1개인지, 읽기·쓰기 2개 이상인지 결정했는가
- [ ] SSD의 TBW/DWPD와 PLP 유무를 확인했는가
- [ ] SSD 용량 계산 뒤 남는 RAM이 Docker·VM에도 충분한가
- [ ] UPS와 별도 백업이 있고, 캐시를 끈 상태에서 복원 절차를 적어 두었는가
- [ ] `SSD Cache Advisor` 분석 기간을 거친 뒤 필요한 용량만 선택했는가

요점은 간단하다. 작은 랜덤 I/O가 반복되고 네트워크가 병목이 아니라면 내구도와 전원 보호가 갖춰진 SSD 캐시가 선택지가 된다. 그 조건이 아니면 캐시보다 백업 디스크와 복원 검증에 돈을 쓰는 편이 운영 실패 비용을 줄인다.

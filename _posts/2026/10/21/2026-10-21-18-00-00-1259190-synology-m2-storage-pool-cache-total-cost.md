---
layout: post
title: "Synology DSM 7.2 M.2 SSD 저장 풀 - DS923+·DS1522+ 캐시 비교와 총비용"
description: "Synology DSM 7.2에서 DS923+·DS1522+의 M.2 SSD 저장 풀과 SSD 캐시를 용도·호환성·복구 위험·총비용 기준으로 선택하는 방법을 정리한다."
date: 2026-10-21
tags: [Synology, DSM, NAS, SSD, Docker, HomeLab]
comments: true
share: true
---

![Synology NAS와 M.2 NVMe SSD 저장 풀 구성 예시](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?auto=format&fit=crop&w=1400&q=80)

이 그림에서 볼 것은 HDD 볼륨 전체를 빠르게 만드는 캐시와, 앱·VM 파일을 SSD에 따로 두는 저장 풀의 차이다.

DS923+나 DS1522+에서 Docker, 데이터베이스, 가상 머신처럼 작은 파일을 자주 읽고 쓰면 M.2 SSD 저장 풀이 판단하기 쉽다. 반대로 대용량 사진·영상 파일을 HDD에 순차적으로 저장하고 읽는 용도라면 SSD를 추가하지 않는 편이 낫다. SSD 캐시는 모든 작업을 자동으로 빠르게 만드는 장치가 아니며, DSM 7.2 이상에서도 모델과 SSD 호환 목록을 따로 확인해야 한다.

## 캐시와 저장 풀은 비용보다 역할이 다르다

| 선택 | SSD 수 기준 | 잘 맞는 작업 | 놓치기 쉬운 제한 |
|---|---:|---|---|
| 읽기 전용 캐시 | 1개 이상 | 반복 조회가 많은 작은 파일 | 쓰기 성능 개선을 기대하면 안 됨 |
| 읽기·쓰기 캐시 | 2개 이상, 이중화 권장 | DB·VM의 랜덤 읽기·쓰기 | SSD 내구도와 전원 장애 위험을 함께 봐야 함 |
| M.2 저장 풀 | 1개부터 가능하지만 운영 데이터는 RAID1 권장 | Docker, VM, 썸네일, DB | M.2를 뽑으려면 NAS를 완전히 꺼야 함 |

Synology 공식 문서 기준으로 DS923+와 DS1522+는 M.2 SSD 저장 풀을 지원한다. 이 기능은 DSM 7.2 이상에서 제공되며, DSM 자체는 M.2에 설치할 수 없다. M.2 저장 풀은 Synology 호환 SSD만 사용하도록 안내되고, Synology High Availability(SHA)는 지원하지 않는다. 캐시는 DSM 7.4 사양상 읽기 전용과 읽기·쓰기 모드를 제공하며, 캐시 1GiB마다 약 400KiB의 시스템 메모리를 요구하고 사전 설치 메모리의 25%를 넘길 수 없다.

## 가상 비용으로 비교하는 기준

실제 가격은 SSD 모델과 판매 시점에 따라 달라지므로 아래는 2026년 10월 21일에 판단 방법을 설명하기 위한 가상 계산이다. 800GB SSD 한 개를 15만 원으로 가정하면 읽기·쓰기 캐시와 RAID1 저장 풀 모두 SSD 두 개, 초기 SSD 비용 30만 원이 든다.

차이는 사용 가능한 공간과 복구 범위다. 캐시는 HDD 볼륨의 복사본이므로 SSD가 고장 나도 원본 볼륨이 남지만, 저장 풀은 그 안에 둔 Docker 데이터와 VM 디스크 자체가 사라질 수 있다. 저장 풀을 선택하면 앱 데이터 백업을 별도로 구성해야 한다. 따라서 30만 원만 비교하지 말고 `SSD 30만 원 + 외부 백업 매체 + 교체·복구에 걸리는 시간`으로 계산해야 한다.

## 구매 전 체크리스트

- [ ] NAS 모델이 DSM 7.2 이상이고 M.2 저장 풀 지원 목록에 있는가
- [ ] SSD 모델이 Synology 호환 목록에 있고, TBW·전원 손실 보호 조건을 확인했는가
- [ ] 앱·VM을 저장 풀에 둘 경우 두 SSD RAID1과 별도 백업을 준비했는가
- [ ] 캐시라면 메모리 요구량과 실제 랜덤 I/O가 있는지 확인했는가
- [ ] M.2 교체 때 종료 작업과 복구 시간을 감당할 수 있는가

대용량 미디어 보관만 한다면 구매하지 않아도 된다. Docker·VM·DB가 HDD에서 병목을 만들고, 백업까지 감당할 수 있을 때만 M.2 저장 풀이 비용을 정당화한다. 반대로 기존 HDD 볼륨의 반복 읽기·쓰기 패턴을 측정하지 않은 채 캐시부터 사는 선택은 피하는 편이 안전하다.

참고: [Synology M.2 SSD 저장 풀 지원 모델](https://kb.synology.com/en-my/DSM/tutorial/Which_models_support_M_2_SSD_storage_pool), [Synology SSD 캐시 고려사항](https://kb.synology.com/en-ca/DSM/tutorial/What_are_Some_Considerations_for_Creating_SSD_Cache), [DSM 7.2 기능 소개](https://www.synology.com/en-us/DSM72)

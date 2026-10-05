---
layout: post
title: "Synology DSM 7.4·QNAP QTS 5 NAS 하이버네이션 - Docker 사용 환경의 전기료와 총비용"
description: "Synology DSM 7.4와 QNAP QTS 5에서 HDD 하이버네이션이 실제로 작동하는 조건을 비교하고, Docker·백업·전기료를 포함한 NAS 운영 총비용 계산법을 정리한다."
date: 2026-10-06
tags: [Synology, QNAP, DSM, Docker, NAS보안, HomeLab]
comments: true
share: true
---

![NAS 하이버네이션과 24시간 운영 전력 비용](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

저장 파일만 가끔 읽는 2~4베이 NAS라면 HDD 하이버네이션과 예약 종료가 전기료를 줄이는 데 유리하다. 반대로 Docker, Home Assistant, 감시카메라, 클라우드 동기화를 24시간 돌리는 환경이라면 절전 기능을 믿고 큰 NAS를 사기보다 항상 회전하는 상태의 전력과 백업 비용을 계산해야 한다. 이 글의 기준일은 2026년 10월 6일이며, 특정 장비에서 실제 측정한 값이 아니라 공식 조건과 가상 계산을 구분한다.

## 절전이 안 되는 이유는 디스크가 아니라 서비스다

Synology DSM 7.4는 `제어판 → 하드웨어 및 전원 → 드라이브 하이버네이션`에서 유휴 시간을 정한다. 하지만 공식 문서에는 Container Manager, Active Backup, Cloud Sync, Synology Drive, Plex, SMB 접근, 썸네일 생성, 데이터 스크러빙, SSD 캐시 등이 디스크를 깨우거나 하이버네이션을 막을 수 있다고 적혀 있다. Docker 컨테이너를 SSD에 옮겨도 로그·DB·미디어 인덱스가 HDD 공유 폴더에 있으면 HDD는 계속 깨어 있을 수 있다.

QNAP QTS 5도 디스크 대기 모드를 제공하지만, QNAP 공식 FAQ는 Container Station 실행 여부를 하이버네이션 실패 원인으로 안내한다. 외장 HDD는 NAS가 대기 모드를 직접 제어하지 못하는 제한도 있다. 따라서 “M.2 슬롯이 있으니 HDD가 잔다”가 아니라 서비스별 데이터 경로를 봐야 한다.

| 환경 | 구매·운영 판단 | 빠뜨리기 쉬운 비용 |
|---|---|---|
| 사진·문서 백업, 하루 1~2회 접근 | 2베이와 예약 종료부터 검토 | WOL 실패 시 원격 접근 불편, 백업 시간 |
| Docker·Home Assistant 상시 실행 | HDD 상시 회전을 기본값으로 계산 | SSD 앱 풀, RAM, UPS, 냉각 |
| CCTV·Plex·Synology Drive | 하이버네이션보다 안정적인 24시간 운용 우선 | 녹화 디스크, 교체 디스크, 전기료 |
| 외장 USB 백업만 연결 | 백업 후 분리하는 방식이 유리 | USB 디스크 절전 호환성, 랜섬웨어 노출 |

## 가상 사례로 총비용 계산하기

가격은 판매처와 시점에 따라 달라지므로 아래 숫자는 실제 견적이 아닌 계산 예시다. 4베이 NAS가 평균 35W, 2베이 NAS가 18W를 사용하고 전기요금을 kWh당 160원으로 가정하면 다음 식으로 비교할 수 있다.

`연간 전기료 = 평균 W ÷ 1000 × 24 × 365 × kWh 단가`

4베이는 약 24만 5천 원, 2베이는 약 12만 6천 원이 5년간 누적된다. 여기에 4베이용 HDD 4개, 예비 디스크 1개, 외장 백업 디스크, UPS를 더한다. 반대로 2베이는 처음 싸도 용량 부족으로 USB 디스크나 새 NAS를 추가하면 네트워크 복사와 복원 검증 비용이 생긴다. 절전으로 아끼는 전기료보다 장애 때 백업을 복원할 수 있는지가 우선이다.

## 구매 전 30분 점검표

- [ ] NAS 본체만이 아니라 HDD 회전 상태에서 소비전력을 확인했는가
- [ ] DSM 7.4 또는 QTS 5에서 사용할 패키지와 컨테이너의 데이터·로그 경로를 적었는가
- [ ] SMB, Cloud Sync, 색인, SMART, 스크러빙 예약 시간이 겹치지 않는가
- [ ] 하이버네이션을 끈 상태의 1년 전기료를 계산했는가
- [ ] 백업 디스크는 백업 후 분리되며, 복원 테스트를 할 수 있는가
- [ ] 전원 장애 때 안전 종료할 UPS 용량과 USB 호환성을 확인했는가

구매하지 않아도 되는 조건도 분명하다. 하루에 한두 번 파일을 열고 백업만 하는데 Docker나 감시 기능이 없다면 SSD 캐시나 고성능 10GbE보다 예약 종료와 외장 백업이 먼저다. 반대로 컨테이너와 녹화 서비스를 계속 운영한다면 하이버네이션 성공을 전제로 한 절약 계산을 버리고, 상시 전력·냉각·UPS·복구 디스크를 포함해 비교해야 한다.

공식 확인 자료는 [Synology DSM 하이버네이션 방해 요소](https://kb.synology.com/en-uk/DSM/tutorial/What_stops_my_Synology_NAS_from_entering_System_Hibernation), [Synology 하이버네이션 모드](https://kb.synology.com/en-ph/DSM/tutorial/What_is_the_difference_between_HDD_Hibernation_System_Hibernation_and_Deep_Sleep), [QNAP 외장 HDD 대기 제한](https://www.qnap.com/en-us/how-to/faq/article/can-the-nas-control-external-hdds-standby-mode), [QNAP 디스크 대기 문제 해결](https://www.qnap.com/en-me/how-to/faq/article/why-cant-my-nas-drives-enter-standby-mode)이다.

핵심은 베이 수가 아니라 실제 서비스가 디스크를 깨우는지다. 하이버네이션이 작동하지 않는 NAS는 절전형 장비가 아니라 24시간 서버로 보고 총비용을 계산해야 한다.

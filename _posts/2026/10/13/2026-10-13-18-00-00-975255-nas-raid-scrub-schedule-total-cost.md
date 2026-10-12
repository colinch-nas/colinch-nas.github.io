---
layout: post
title: "NAS RAID 스크럽 주기 - DSM 7.3·QTS 5·TrueNAS SCALE 25.10 무결성 검사와 총비용"
description: "Synology DSM 7.3, QNAP QTS 5, TrueNAS SCALE 25.10의 RAID 스크럽 차이와 2·4·8베이 NAS의 주기, 백업 충돌, 추가 장비 비용을 판단한다."
date: 2026-10-13
tags: [Synology, QNAP, TrueNAS, DSM, NAS보안]
comments: true
share: true
---

![NAS 저장장치와 무결성 검사를 상징하는 서버 스토리지](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?auto=format&fit=crop&w=1600&q=80)

RAID 스크럽은 새 디스크가 아니라 저장된 데이터의 체크섬·패리티 오류를 검사하는 유지보수다. 사진·문서 위주 2베이는 월 1회, 4~8베이는 사용량이 낮은 주말 월 1회를 출발점으로 삼는다. 백업이 하나뿐이면 주기만 줄여도 안전해지지 않는다.

## 플랫폼별 차이

2026년 10월 13일 공식 문서를 확인한 기준이며, 메뉴는 모델별로 다를 수 있다.

| 환경 | 검사 대상과 기본 동작 | 주의할 실패 조건 |
|---|---|---|
| Synology DSM 7.3 | SHR·RAID의 데이터 스크러빙. 공유 폴더 체크섬이 켜져야 파일 데이터 검사 가능 | 성능 우선으로 두면 검사 시간이 길어지고, 성능 저하를 피하려다 예약 자체를 끄기 쉽다 |
| QNAP QTS 5 / QuTS hero | RAID 스크러빙은 RAID 그룹을 검사하고, QuTS hero 풀 스크러빙은 ZFS 손상 블록 복구를 시도 | QNAP 공식 예시는 30시간 작업이 재부팅과 겹치면 처음부터 다시 시작한다고 안내한다 |
| TrueNAS SCALE 25.10 | 풀 생성 때 일요일 0시 스크럽이 기본 생성되며 Storage Health에서 일정과 Threshold Days를 조정 | 스크럽은 백업이 아니다. 풀 자체가 사라지면 검사 결과도 함께 잃는다 |

“매주”보다 작업 완료 시간을 확보하는 것이 중요하다. TrueNAS 25.10은 Storage Health, QNAP은 Storage & Snapshots의 Pool Scrubbing Schedule에서 조정한다. Synology는 체크섬을 켜지 않은 공유 폴더까지 보호하지 않는다.

## 베이 수와 사용량별 판단

| 규모 | 권장 출발점 | 별도 장비를 사지 않아도 되는 조건 |
|---|---|---|
| 2베이·1~2명 | 월 1회, 새벽 시간 | 외장 디스크나 다른 장치에 원본 백업이 있고, 검사 중 느려져도 업무가 없다 |
| 4베이·3~5명 | 월 1회, 백업·SMART와 날짜 분리 | 사용량이 70% 이하이고 주말에 완료 창을 확보할 수 있다 |
| 8베이·6명 이상 | 월 1회부터 시작, 완료 시간 기록 후 조정 | 업무시간 외에 충분한 창이 있고 UPS가 정전 중 중단을 줄인다 |

가상 사례로 8베이 풀이 30시간 걸리고 평균 80W, 전력 단가가 160원/kWh라면 한 번에 `0.08 × 30 × 160 = 384원`, 월 1회 연 4,608원이다. 디스크 교체·업무 지연·UPS 가격은 제외했다. 전기료보다 백업과 재부팅 충돌로 작업을 반복하는 운영 비용을 우선 줄여야 한다.

## 예약 전 체크리스트

- [ ] 마지막 백업에서 실제 파일 1개를 복원했다.
- [ ] 스크럽·재동기화·SMART 장기 검사를 겹치지 않았다.
- [ ] 업무시간·Docker 인덱싱·대용량 복사를 제외했다.
- [ ] 재부팅 제외, Threshold Days, 오류 알림을 확인했다.
- [ ] 다른 장치나 외장 디스크에 백업이 남아 있다.

오류가 발견되면 “복구 완료”로 보지 말고 로그·SMART·백업 파일을 함께 확인한다. Synology는 데이터 체크섬 조건이 필요하고, QNAP은 중단 작업을 다시 시작해야 할 수 있다. TrueNAS도 오류 수정 또는 알림을 제공할 뿐, 세 기능 모두 백업 복원을 대신하지 않는다.

참고: [Synology Data Scrubbing](https://kb.synology.com/en-uk/PAS/help/PAS/StorageManager/storage_pool_data_scrubbing?version=1_0), [QNAP RAID Scrubbing](https://www.qnap.com/ko-kr/how-to/faq/article/how-do-i-schedule-raid-scrubbing-and-avoid-conflicts-with-nas-reboots-in-the-nas), [TrueNAS SCALE 25.10 Managing Pools](https://www.truenas.com/docs/scale/25.10/scaletutorials/storage/managepoolsscale/). 이 글은 해당 문서와 가상 전력 계산을 바탕으로 했으며, 특정 장비에서 직접 실행한 테스트 결과가 아니다.

2베이는 월 1회와 복원 확인, 4베이는 작업 분리와 알림, 8베이는 완료 시간과 UPS까지 함께 관리한다. 스크럽은 추가 장비나 백업의 대체품이 아니다.

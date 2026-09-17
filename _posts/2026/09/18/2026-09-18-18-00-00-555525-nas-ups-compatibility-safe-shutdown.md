---
layout: post
title: "Synology DSM 7.3·QNAP QTS 5.2·TrueNAS SCALE 25.10 UPS 호환성 - 안전 종료와 구매 기준"
description: "Synology DSM 7.3, QNAP QTS 5.2, TrueNAS SCALE 25.10에서 UPS를 고를 때 USB·SNMP 호환성, 네트워크 구성, 안전 종료와 복구 테스트까지 구매 전에 확인할 기준을 정리한다."
date: 2026-09-18
tags: [Synology, QNAP, TrueNAS, UPS, HomeLab, NAS보안]
comments: true
share: true
---

![NAS와 UPS를 함께 구성한 홈서버 전원 보호](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

Synology DSM 7.3, QNAP QTS 5.2, TrueNAS SCALE 25.10을 운영하면서 정전 때 NAS를 안전하게 끄고 싶다면, UPS의 VA 숫자보다 **내 NAS가 그 UPS를 어떤 방식으로 인식하는지**를 먼저 확인해야 한다. 1대의 2베이 NAS만 보호한다면 USB 통신 UPS로 충분한 경우가 많고, NAS·미니PC·스위치를 함께 끌 계획이면 SNMP 또는 한 대를 UPS 서버로 쓰는 구성이 유리하다. 단순 멀티탭 대체가 목적이고 정전이 드문 환경이라면 UPS 구매보다 외장 백업과 복원 검증이 먼저다.

이 그림에서 볼 것은 배터리만이 아니라 UPS 데이터 케이블이 NAS와 연결되는 흐름이다.

## 제품군별 호환성 차이

| 환경 | 확인할 통신 방식 | 안전 종료 구성 | 구매 전 걸림돌 |
|---|---|---|---|
| Synology DSM 7.3 | 호환 목록의 USB 또는 SNMP | Safe Mode 진입 후 서비스 중지·볼륨 언마운트 | USB 허브 연결은 지원되지 않음 |
| QNAP QTS 5.2 | 모델별 호환 목록의 USB 또는 SNMP | 정전 시간 또는 배터리 상태 기준 자동 보호·종료 | NAS 모델별 지원 목록을 따로 확인해야 함 |
| TrueNAS SCALE 25.10 | NUT 드라이버의 USB·시리얼·네트워크 | UPS Service의 shutdown mode와 timer | 드라이버가 맞지 않으면 인식만 되고 종료가 안 될 수 있음 |

Synology는 호환 목록의 UPS를 두 번째 Synology 제품에 Network UPS Server로 공유할 수 있지만, USB 장치는 본체 포트에 직접 연결해야 한다고 안내한다. [Synology 호환성 목록](https://www.synology.com/en-us/compatibility?category=upses&search_by=category)에서 NAS 모델과 UPS 모델을 함께 선택해야 한다. QNAP도 [QTS 5.2 UPS 문서](https://docs.qnap.com/operating-system/qts/5.2.x/en-us/uninterruptible-power-supply-ups-20AC99C1.html)와 공식 호환 목록 확인을 전제로 USB·SNMP 연결을 지원한다. TrueNAS는 [25.10 하드웨어 가이드](https://www.truenas.com/docs/scale/25.10/gettingstarted/scalehardwareguide/)와 [UPS Service 문서](https://www.truenas.com/docs/scale/systemsettings/services/upsservices/)에서 NUT 기반 통신과 graceful shutdown을 설명한다.

## 용량보다 먼저 계산할 것

UPS는 NAS만 연결했을 때의 최대 소비전력을 기준으로 고른다. 예를 들어 NAS 35W, 공유기 10W, 스위치 8W, 미니PC 25W라면 합계는 78W다. 여기에 부팅 순간과 배터리 노화를 고려해 1.5배를 적용하면 약 117W다. 이 값보다 높은 정격 출력과, NAS가 종료를 끝낼 수 있는 배터리 시간을 확인한다.

전기료는 UPS 구매비와 별도로 계산한다. 가상 사례로 평균 50W 장비를 24시간 켜고, 전력 단가를 1kWh당 160원으로 가정하면 `0.05 × 24 × 365 × 160 = 연 70,080원`이다. 실제 요금은 누진 구간과 측정 위치에 따라 달라지므로 카탈로그 소비전력만으로 구매 결론을 내리면 안 된다. NAS와 UPS를 모두 살 필요 없이, 밤에만 백업하는 장비라면 자동 전원 예약과 외장 디스크가 더 싼 선택일 수 있다.

## 구매 후 반드시 할 복원·정전 테스트

UPS가 호환 목록에 있어도 실제 종료까지 확인하지 않으면 백업 장비가 아니다. 아래 순서로 부하를 낮춘 상태에서 테스트한다.

```text
[ ] UPS 배터리 잔량과 교체일 기록
[ ] NAS가 USB/SNMP 상태를 Online으로 표시하는지 확인
[ ] NAS·공유기·스위치가 같은 UPS 또는 통신 가능한 전원에 연결됐는지 확인
[ ] NAS 설정에서 종료 조건과 지연 시간을 기록
[ ] 전원 입력을 제거하고 배터리 상태·알림·로그 확인
[ ] 설정한 시간 뒤 서비스 중지와 안전 종료 확인
[ ] 전원을 복구하고 자동 부팅·공유 폴더·Docker/VM 상태 확인
[ ] 백업 파일 하나를 다른 장비에서 실제로 열어 복원 확인
```

Synology는 Safe Mode에서 서비스를 멈추고 데이터 볼륨을 언마운트한다. QNAP은 배터리 사용 상태에서 자동 보호 또는 종료 모드로 동작한다. TrueNAS는 기본값을 그대로 믿기보다 UPS 로그에서 올바른 드라이버와 shutdown timer가 동작했는지 확인해야 한다. 실제 정전 테스트는 이 글에서 수행한 결과가 아니라 사용자의 장비에서 직접 재현해야 하는 절차다.

짧게 판단하면, 2베이 Synology 한 대는 공식 호환 USB UPS와 직접 연결하는 구성이 가장 단순하다. QNAP과 TrueNAS를 함께 운영하거나 미니PC까지 묶는다면 SNMP UPS 또는 USB 연결 NAS를 NUT/Network UPS Server의 master로 정하고, 네트워크 장비도 UPS에 연결해야 한다. 호환 목록을 확인할 수 없는 저가 UPS, USB 허브를 거치는 연결, 복원 파일을 한 번도 열어보지 않은 백업은 구매를 미뤄야 하는 조건이다.

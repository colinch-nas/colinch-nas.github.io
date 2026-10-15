---
layout: post
title: "TrueNAS SCALE 25.10 UPS 호환성 - USB·NUT 드라이버와 NAS 여러 대 구성 비용"
description: "TrueNAS SCALE 25.10과 Synology·QNAP NAS에 UPS를 연결할 때 USB HID·NUT 드라이버와 네트워크 마스터 호환성을 확인하고, 구매 전 정전 테스트와 총비용을 계산하는 기준을 정리한다."
date: 2026-10-15
tags: [TrueNAS, NAS보안, HomeLab, Synology, QNAP]
comments: true
share: true
---

![NAS와 UPS 전원 백업 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

TrueNAS SCALE 25.10이나 여러 대의 NAS를 운영한다면 UPS는 배터리 용량보다 **NAS에서 상태를 읽고 안전 종료까지 확인되는지**가 우선이다. APC·CyberPower처럼 TrueNAS 문서와 NUT(Network UPS Tools) 호환 정보가 확인되는 제품을 고르는 편이 안전하다. 반대로 USB 전원만 공급하고 데이터 통신이 안 되는 모델, 최신 드라이버가 필요한 제품은 싸도 구매하지 않는 편이 낫다.

## 호환성은 제조사 이름보다 연결 방식으로 판단한다

TrueNAS는 UPS 서비스에 NUT를 사용하고, 장치별 지원 여부를 하드웨어 호환 목록에서 확인하라고 안내한다. QNAP은 NAS 모델별 호환 목록을 따로 확인하도록 하며 USB와 SNMP 연결을 지원한다. 따라서 “USB 포트가 있다”는 설명만으로는 부족하다.

| 환경 | 확인할 항목 | 운영 판단 |
|---|---|---|
| TrueNAS SCALE 25.10 | NUT 드라이버와 모델·펌웨어 조합 | 구매 전 NUT 장치 목록 확인 |
| Synology DSM | UPS 호환 목록, USB 또는 네트워크 UPS 지원 | 모델별 자동 종료 메뉴 확인 |
| QNAP QTS·QuTS hero | NAS 모델별 호환 목록, USB/SNMP | 여러 대면 한 대를 UPS 마스터로 지정 |
| 가정용 소형 NAS 1대 | USB 데이터 케이블, 자동 종료 지연 | SNMP 카드까지 살 필요가 없는 경우가 많음 |

특히 2026년에 확인된 TrueNAS 커뮤니티 사례처럼 UGREEN US3000은 USB 장치로 보이더라도 TrueNAS SCALE 25.10에 포함된 NUT 버전과 필요한 HID 서브드라이버가 맞지 않아 인식되지 않을 수 있다. 이 사례는 특정 제품 전체가 불량이라는 뜻이 아니라, **펌웨어·NUT 버전까지 묶어 호환성을 봐야 한다**는 의미다.

## NAS 여러 대라면 UPS 한 대로 묶을 때의 조건

UPS에 USB로 직접 연결한 NAS를 마스터로 두고, 같은 LAN의 다른 NAS를 네트워크 슬레이브로 연결할 수 있다. QNAP 공식 절차도 이 구조를 지원하지만, 스위치와 공유기까지 꺼지면 슬레이브가 종료 신호를 받지 못한다. NAS만 UPS에 꽂고 네트워크 장비를 일반 멀티탭에 두는 구성은 복구 순서가 꼬일 수 있다.

가상 사례로 NAS 2대가 각각 평균 80W, 스위치·공유기가 25W를 쓴다고 가정하면 부하는 약 185W다. 다만 UPS 용량은 이 계산값만 보지 말고 실제 소비전력과 순간 부하를 함께 확인해야 한다. 여기에 UPS 본체 가격만 더하지 말고 교체 배터리, USB/SNMP 관리 비용, 정전 중 계속 켜둘 네트워크 장비 전력까지 포함해야 한다. 정확한 런타임은 VA 숫자만으로 계산하지 말고 제조사 런타임 표와 실측으로 확인한다.

## 구매 전 10분 테스트 체크리스트

- [ ] 제조사 호환 목록에서 NAS 모델과 UPS SKU를 함께 찾았다.
- [ ] TrueNAS라면 NUT 장치 목록에서 드라이버와 모델명을 확인했다.
- [ ] UPS와 NAS를 USB 데이터 케이블로 연결하고 배터리 상태가 표시된다.
- [ ] 전원 플러그를 뽑아 NAS가 배터리 전환을 감지하는지 확인한다.
- [ ] 설정한 지연 시간 뒤 NAS가 정상 종료되는지 확인한다.
- [ ] 여러 대라면 스위치·공유기도 UPS에 연결하고 슬레이브 종료를 확인한다.
- [ ] 정전 뒤 전원이 복구될 때 자동 부팅 정책이 의도와 같은지 확인한다.

이 테스트를 실제로 수행하지 않은 상태라면 글이나 판매 페이지의 “호환” 문구를 운영 검증으로 착각하면 안 된다. 구매 후에는 한 달에 한 번 배터리 상태와 알림을 확인하고, 분기마다 짧은 정전 모의 테스트를 기록하는 편이 복구 비용을 줄인다.

### 짧은 요점

TrueNAS SCALE 25.10 UPS 선택은 VA가 큰 제품을 고르는 문제가 아니다. NUT 드라이버, USB 데이터 통신, NAS별 자동 종료, 네트워크 장비 전원까지 확인한 뒤 구매해야 한다. NAS 한 대면 검증된 USB 모델이 단순하고, 여러 대면 네트워크 UPS 마스터·슬레이브 구성이 총비용과 복구 위험을 함께 줄인다.

### 출처

- [TrueNAS UPS 서비스 공식 문서](https://www.truenas.com/docs/scale/systemsettings/services/upsservices/)
- [TrueNAS 하드웨어 가이드](https://www.truenas.com/docs/scale/gettingstarted/scalehardwareguide/)
- [QNAP UPS 사용 공식 FAQ](https://www.qnap.com/en-us/how-to/faq/article/how-do-i-use-a-ups-with-my-qnap-nas)
- [QNAP 여러 NAS와 UPS 구성 공식 FAQ](https://www.qnap.com/en-us/how-to/faq/article/how-do-i-set-up-multiple-nas-devices-using-one-ups)
- [TrueNAS SCALE 25.10 UGREEN US3000 호환성 사례](https://forums.truenas.com/t/ugreen-us3000-ups-usb-vid-2b89-ffff-not-accessible-by-nut-on-truenas-scale-25-10/66980)

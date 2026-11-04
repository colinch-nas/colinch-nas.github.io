---
layout: post
title: "NAS UPS 호환성 - Synology DSM 7·QNAP QTS·TrueNAS SCALE 25.10 정전 종료와 총비용"
description: "2·4·8베이 NAS와 공유기·스위치 구성에서 USB·SNMP·네트워크 UPS 호환성, 자동 종료와 복구 테스트, 배터리까지 포함한 총비용 판단 기준을 정리한다."
date: 2026-11-04
tags: [NAS, Synology, QNAP, TrueNAS, HomeLab, UPS, NAS보안]
comments: true
share: true
---

![NAS와 UPS, 공유기, 스위치의 전원 보호 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

*이 그림에서 볼 부분은 UPS 용량보다 NAS·스위치·공유기가 같은 정전 대응 경로에 있는지다.*

2베이 NAS 한 대만 보호한다면 USB UPS가 가장 단순하고, NAS와 스위치·공유기까지 함께 종료해야 하면 네트워크 UPS 또는 USB를 연결한 서버형 구성이 유리하다. 반대로 정전 때 NAS를 몇 분 더 켜두는 것만 원하고 자동 종료 테스트를 하지 않을 사람이라면 비싼 SNMP 모델을 살 이유가 적다. UPS는 전원을 오래 공급하는 장비가 아니라, 저장 중인 데이터를 정리하고 안전하게 끄는 시간을 사는 장비다.

## 연결 방식별 호환성

| 방식 | 잘 맞는 환경 | 확인할 것 | 구매를 보류할 조건 |
|---|---|---|---|
| USB | 2베이 NAS 1대, 소규모 홈서버 | NAS USB 포트, 제조사 호환 목록, 케이블 | USB 인식·종료 시험을 할 수 없을 때 |
| SNMP | 여러 NAS와 네트워크 장비 | UPS 네트워크 카드, 같은 관리망, 라이선스 | NAS 한 대뿐이고 비용을 줄여야 할 때 |
| 네트워크 슬레이브 | UPS에 USB를 연결할 대표 NAS가 있을 때 | 대표 NAS의 NUT/UPS 서버와 클라이언트 지원 | 대표 장비가 먼저 꺼지면 안 되는 구성 |

Synology DSM 7은 USB UPS의 대기 시간을 지정하고, NAS를 네트워크 UPS 서버로 만들어 다른 Synology NAS에 상태를 전달할 수 있다. 다만 USB 허브 연결은 지원하지 않으며, UPS 전원 끄기 기능은 모델별로 달라 정전 시뮬레이션이 필요하다. QNAP QTS는 USB·SNMP·네트워크 슬레이브를 지원하고, 복구 후 이전 전원 상태 유지·항상 켜기·꺼진 상태 유지도 따로 선택한다. TrueNAS SCALE 25.10은 NUT를 사용하므로 모델별 드라이버와 폴링 간격을 확인해야 한다. [Synology UPS 문서](https://kb.synology.com/en-global/DSM/help/DSM/AdminCenter/system_hardware_ups?version=7) · [Synology 호환 목록](https://www.synology.com/knowledgebase/DSM/tutorial/Compatibility_List/What_UPS_Uninterruptible_Power_Supply_devices_do_Synology_product_support) · [QNAP UPS 설정](https://www.qnap.com/en/how-to/tutorial/article/how-to-configure-an-uninterruptible-power-supply-with-a-qnap-nas) · [TrueNAS SCALE 25.10 UPS](https://www.truenas.com/docs/scale/25.10/scaletutorials/systemsettings/services/upsservicesscale/)

## 베이 수보다 실제 부하를 먼저 본다

2베이는 NAS와 공유기만, 4베이는 여기에 스위치와 액세스 포인트를 더하는 식으로 보호 대상을 정한다. 8베이와 홈서버를 함께 묶을 때는 디스크 수가 아니라 실제 소비전력을 와트미터로 재고, 시작 순간의 부하와 배터리 교체 주기를 별도로 계산한다. UPS 가격만 비교하면 배터리, USB·SNMP 카드, 교체 디스크의 비정상 종료 위험을 놓친다.

```text
3년 총비용 = UPS 본체 + 추가 관리 카드 + 3년 내 배터리 교체비
           + 보호할 멀티탭·케이블 + 정전 테스트에 드는 시간
```

위 식의 배터리 교체 주기와 가격은 제조사·사용 환경에 따른 가정이다. 실제 구매 전에는 UPS 정격보다 현재 부하, 콘센트 수, 관리 카드 필요 여부를 먼저 기록한다.

## 발행 전 10분 체크리스트

- [ ] NAS 모델과 DSM·QTS·TrueNAS 버전을 적었다.
- [ ] UPS 제조사·모델이 호환 목록 또는 NUT 드라이버에 있는지 확인했다.
- [ ] USB는 허브를 거치지 않고 NAS에 직접 연결했다.
- [ ] 배터리 사용 중 자동 종료 시간과 복구 후 전원 정책을 정했다.
- [ ] NAS만이 아니라 스위치·공유기 전원도 같은 기준으로 정했다.
- [ ] 실제 전원을 차단해 알림, 안전 종료, UPS 재기동을 순서대로 확인했다.

핵심은 “UPS가 연결됐다”가 아니라 “정전 뒤 데이터가 닫히고, 전원이 돌아왔을 때 필요한 장비만 다시 올라오는가”다. 1대·저부하 환경은 USB UPS로 충분할 수 있고, 여러 NAS·스위치·홈서버를 묶는 환경만 SNMP나 네트워크 서버 구성을 검토하면 된다.

---
layout: post
title: "NAS UPS 호환성 - Synology DSM·QNAP QTS·TrueNAS SCALE 25.10 자동 종료와 총비용"
description: "Synology DSM, QNAP QTS·QuTS hero, TrueNAS SCALE 25.10에서 UPS 호환성과 자동 종료 조건을 비교하고 2·4베이 사용량별 구매·운영 총비용을 계산한다."
date: 2026-10-20
tags: [Synology, QNAP, TrueNAS, NAS보안, HomeLab]
comments: true
share: true
---

![NAS와 UPS 자동 종료 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

*이 그림에서 볼 것은 NAS 자체보다 UPS와 데이터 통신 케이블이 함께 있어야 안전 종료가 된다는 점이다.*

Synology DSM, QNAP QTS·QuTS hero, TrueNAS SCALE 25.10 중 하나만 쓰는 1~2베이 사용자는 USB 통신을 지원하고 공식 호환 목록에 있는 500~750VA UPS가 가장 단순하다. NAS와 공유기·스위치·홈서버를 함께 끌 사용자라면 1000VA급과 네트워크 UPS 구성을 검토할 만하다. 반대로 정전이 거의 없고 백업·복구가 이미 검증된 단독 NAS라면 UPS를 큰 용량으로 살 이유는 적다. UPS의 목적은 몇 시간을 버티는 것이 아니라 쓰기를 정리하고 정상 종료할 시간을 버는 데 있다.

## 세 운영체제에서 달라지는 호환 조건

| 환경 | 확인할 것 | 놓치기 쉬운 제한 |
|---|---|---|
| Synology DSM | Synology UPS 호환 목록, USB 또는 SNMP | USB 허브 연결은 지원하지 않으며, 다른 NAS를 물리려면 Network UPS Server 설정이 필요하다 |
| QNAP QTS·QuTS hero | NAS 모델과 UPS의 QNAP 호환 목록 | USB·SNMP 방식이 다르고, 정전 후 3~5분 종료 시간을 별도로 지정해야 한다 |
| TrueNAS SCALE 25.10 | NUT 장치·드라이버 지원 여부 | 제품명이 같아도 USB VID나 드라이버가 맞지 않으면 자동 인식되지 않을 수 있다 |

Synology는 호환 목록 제품도 펌웨어·하드웨어 변경에 따라 안정성을 보장하지 않는다고 적고, USB 허브가 아닌 NAS 본체에 직접 연결하라고 안내한다. QNAP은 모델별 Compatibility List를 확인한 뒤 USB 또는 SNMP로 연결한다. TrueNAS는 NUT를 사용하므로 [NUT 장치 목록](https://networkupstools.org/stable-hcl.html)과 [SCALE UPS 문서](https://www.truenas.com/docs/scale/24.10/scaleuireference/printview/)를 함께 확인한다.

## 용량보다 통신과 부하를 먼저 계산한다

가상 사례로 NAS 45W, 공유기·스위치 25W, 미니 PC 80W를 합쳐 평소 150W가 흐른다고 하자. UPS 출력 여유를 70% 이하로 두면 필요한 정격은 `150 ÷ 0.7 ≈ 214W`다. 이 경우 500VA급은 NAS만 연결할 때 충분할 수 있지만, 미니 PC까지 넣으면 750~1000VA급이 관리하기 편하다. 실제 런타임은 배터리 상태, 부하율, 모델의 W 정격에 따라 달라지므로 제조사 런타임 표를 기준으로 잡아야 한다.

| 구성 | 권장 판단 | 구매하지 않아도 되는 조건 |
|---|---|---|
| 2베이 NAS 1대, 40~80W | 500~750VA, USB 통신 | 정전 시 자동 종료보다 오프사이트 백업만 필요한 경우 |
| 4~8베이 NAS, 80~180W | 750~1000VA, USB 또는 SNMP | NAS만 보호하고 짧은 순간 정전만 문제인 경우 |
| NAS+미니 PC+네트워크 장비, 150~300W | 1000VA 이상, 네트워크 종료 설계 | 미니 PC를 UPS 배터리 콘센트에서 제외할 수 있는 경우 |

가격은 확인 날짜와 판매처에 따라 바뀌므로 특정 모델 가격을 고정하지 않았다. 총비용은 UPS 본체, 2~3년 뒤 배터리 교체, USB 케이블·SNMP 카드, 배터리 백업 콘센트 수까지 더한다. 이미 UPS가 있다면 새로 사기 전에 NAS가 배터리 상태를 읽는지 확인하는 편이 비용이 낮다.

## 구매 후 반드시 해볼 종료 테스트

실제 장비에서 시험하지 않은 예시 절차다. 구매 전에 다음 체크리스트를 복사해 기록한다.

```text
[ ] NAS 모델·DSM/QTS/QuTS hero/TrueNAS 버전 기록
[ ] UPS 모델, 정격 W·VA, USB VID 또는 SNMP 방식 기록
[ ] 공식 호환 목록과 NUT 드라이버 확인
[ ] NAS·공유기·스위치를 UPS 배터리 콘센트에 연결
[ ] UPS 데이터 케이블을 허브가 아닌 NAS에 직접 연결
[ ] 정전 알림 수신과 배터리 잔량 표시 확인
[ ] 전원 입력을 끊고 안전 종료가 설정 시간 안에 시작되는지 확인
[ ] 종료 뒤 전원을 복구하고 자동 재시작 조건 확인
```

Synology는 Safe Mode에서 서비스를 멈추고 볼륨을 마운트 해제한다. QNAP은 배터리 부족 시 설정한 시간보다 먼저 종료될 수 있다. TrueNAS에서 장치가 응답하지 않으면 폴링 주기나 드라이버를 로그로 확인한다. 근거 없이 Auxiliary Parameters를 넣으면 데이터 손상 위험이 있다. UPS 테스트 로그와 백업 복원 기록을 함께 남겨야 한다.

핵심은 VA 숫자보다 `호환 목록 → 통신 확인 → 안전 종료 테스트` 순서다. 단독 소형 NAS는 USB 500~750VA로 충분할 수 있고, 여러 장비를 묶을 때만 1000VA급과 네트워크 UPS를 계산에 넣으면 된다.


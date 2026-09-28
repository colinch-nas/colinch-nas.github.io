---
layout: post
title: "NAS UPS 호환성 - TrueNAS SCALE 25.10·Synology·QNAP의 USB·네트워크 연결과 총비용"
description: "NAS 1~4인 환경에서 UPS를 살지 판단하도록 TrueNAS SCALE 25.10, Synology DSM, QNAP QTS의 연결 방식·자동 종료·배터리 교체 비용을 비교한다."
date: 2026-09-28
tags: [TrueNAS, Synology, QNAP, HomeLab, NAS보안, UPS]
comments: true
share: true
---

![NAS와 UPS를 USB 또는 네트워크로 연결하는 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

그림에서 볼 부분은 UPS를 NAS 아래에 두는 것보다 **NAS가 정전 신호를 받고 안전하게 종료하는지**가 핵심이라는 점이다.

NAS와 공유기 합계가 60W 안팎인 1인 환경이라면 500~900VA UPS 하나로 시작할 수 있다. 반대로 4인 홈랩에서 NAS·스위치·미니PC까지 한 UPS에 묶으면 USB 호환성과 종료 순서를 먼저 확인해야 한다. 단순히 “정전에도 오래 버티는 배터리”가 목적이고 NAS가 자주 켜져 있지 않다면 UPS를 사지 않고 외장 백업과 자동 종료 설정에 예산을 쓰는 편이 낫다.

## 장비별 연결 방식은 다르다

2026년 9월 28일 공식 문서를 기준으로 정리하면 다음과 같다. 실제 지원 모델은 구매 전 제조사의 호환 목록과 UPS 데이터시트를 다시 확인해야 한다.

| 환경 | 권장 연결 | 정전 때 기대할 동작 | 주의할 점 |
|---|---|---|---|
| TrueNAS SCALE 25.10, NAS 1대 | UPS USB 또는 시리얼 | NUT가 배터리 상태를 읽고 종료 | `BATT`와 `LOWBATT` 조건을 구분해야 함 |
| Synology DSM, NAS 1대 | UPS USB | 서비스 중지·볼륨 마운트 해제 후 안전 모드 | USB에 연결된 NAS만 UPS 상태를 직접 앎 |
| QNAP QTS, NAS 1대 | UPS USB | 지정 시간 후 종료 또는 보호 모드 | 자동 감지 후 종료 정책을 별도로 설정 |
| NAS 2대 이상 | 한 대를 USB 마스터, 나머지는 네트워크 클라이언트 | 마스터가 UPS 정보를 전달 | NAS·스위치가 같은 UPS에 있어야 신호 전달이 유지됨 |

TrueNAS는 NUT(Network UPS Tools)를 사용하고, 공식 문서는 APC와 CyberPower를 상대적으로 잘 동작하는 선택지로 언급한다. Synology는 UPS가 방전되면 서비스를 멈추고 볼륨을 해제하는 Safe Mode를 사용한다. QNAP도 USB 연결 뒤 UPS를 자동 감지하지만, “몇 분 뒤 종료”와 “보호 모드 전환”은 사용자가 정해야 한다.

## UPS 용량보다 먼저 계산할 것

UPS의 VA 숫자를 NAS 소비전력으로 나누면 사용 시간이 나오지 않는다. 배터리 용량, 변환 효율, 부하율, 노후도에 따라 달라지므로 제조사 런타임 표에서 실제 부하 구간을 찾아야 한다.

가상 사례로 NAS 45W, 공유기 10W, 스위치 8W를 합쳐 63W라고 하자. 정전 뒤 5분만 버티고 안전 종료할 목적이면 장시간 런타임 모델보다 USB 통신이 확인된 보급형이 합리적이다. 30분 동안 미니PC까지 계속 돌려야 한다면 UPS 가격뿐 아니라 배터리 교체 주기와 정전 중 작업량까지 비용에 넣어야 한다.

| 선택 | 구매하지 않아도 되는 조건 | 추가 비용으로 생기는 가치 |
|---|---|---|
| UPS 없음 | 정전이 드물고 NAS가 캐시·작업 서버가 아님 | 배터리 교체비와 대기전력 없음 |
| USB UPS 1대 | NAS 1대, 5~10분 후 종료면 충분 | 자동 종료와 파일시스템 손상 위험 감소 |
| 네트워크 UPS 구성 | NAS·서버가 2대 이상 | 한 UPS로 여러 장비의 종료 순서 관리 |
| 순수 정현파 모델 | 일반 NAS 어댑터만 사용 | 민감한 전원장치나 홈랩 장비의 호환 여유 |

가상 구매 예산을 UPS 12만 원, 3년 뒤 배터리 5만 원, 월 대기전력 2kWh로 잡으면 3년 장비비는 `120,000 + 50,000 + (2 × 36 × 전기요금)`이다. NAS가 정전 때마다 갑자기 꺼져 데이터 복구에 반나절이 걸릴 수 있다면 이 금액은 복구 시간 보험에 가깝다. 반대로 재생성 가능한 미디어만 저장하고 외장 백업이 따로 있으면 UPS를 생략할 이유가 충분하다.

## 구매 후 반드시 확인할 체크리스트

- [ ] UPS 제조사 데이터시트에서 USB 통신 방식과 NAS 호환 목록을 확인했다.
- [ ] NAS뿐 아니라 공유기·스위치도 UPS 전원에 연결했다.
- [ ] TrueNAS는 **Services → UPS**, Synology는 **제어판 → 하드웨어 및 전원 → UPS**, QNAP은 전원·UPS 설정에서 종료 조건을 저장했다.
- [ ] 배터리 상태와 마지막 정전 이벤트 알림을 확인했다.
- [ ] 테스트용 정전으로 NAS가 실제 종료되는지 확인했다. 전원 케이블을 뽑기 전 쓰기 작업을 멈춘다.
- [ ] NAS 설정 백업과 중요 데이터 백업이 UPS와 별도 위치에 있다.

UPS는 RAID나 스냅샷을 대신하지 않는다. TrueNAS 공식 문서도 설정 파일, 부트 환경, 데이터 백업을 별도로 유지하라고 안내한다. UPS 테스트에서 NAS가 꺼졌다는 사실만 확인하지 말고, 다시 켠 뒤 공유 폴더·컨테이너·백업 작업이 정상인지까지 확인해야 한다.

### 짧게 정리

NAS 한 대와 5분 안전 종료가 목적이면 USB UPS가 가장 단순하다. 여러 장비를 함께 운영하면 네트워크 UPS의 마스터·클라이언트 호환성을 확인해야 한다. 구매 판단은 VA보다 실제 부하, 자동 종료 동작, 3년 배터리 비용, 그리고 UPS가 없어도 복원 가능한 백업을 함께 계산할 때 맞아진다.

출처: [TrueNAS SCALE 하드웨어 가이드](https://www.truenas.com/docs/scale/gettingstarted/scalehardwareguide/), [TrueNAS 25.10 UPS API 문서](https://api.truenas.com/v26.0/api_methods_ups.update.html), [Synology UPS 안내](https://kb.synology.com/hu-hu/DSM/help/DSM/AdminCenter/system_hardware_ups?version=6), [QNAP UPS 사용 안내](https://www.qnap.com/ko-kr/how-to/faq/article/how-do-i-use-a-ups-with-my-qnap-nas)

---
layout: post
title: "NAS USB 포트 부족 - DSM 7.4·QTS 5.2·TrueNAS SCALE 25.10에서 UPS·백업 연결과 총비용"
description: "NAS USB 포트 하나에 UPS와 외장 백업 디스크를 동시에 연결할 때 생기는 호환성 문제를 DSM 7.4·QTS 5.2·TrueNAS SCALE 25.10 기준으로 비교하고 총비용과 구매 보류 조건을 정리한다."
date: 2026-11-13
tags: [Synology, QNAP, TrueNAS, DSM, HomeLab, NAS보안, 백업]
comments: true
share: true
---

![NAS의 UPS·외장 백업·USB 장치 연결 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)
이 그림에서 볼 부분은 USB 장치 개수가 아니라 **정전 신호와 백업 데이터가 서로 다른 경로로 살아 있어야 한다는 점**이다.

NAS USB 포트가 하나뿐이고 UPS와 외장 백업 디스크를 모두 써야 한다면 USB 허브부터 사면 안 된다. Synology DSM 7 계열은 USB 허브를 지원하지 않고 UPS를 본체에 직접 연결해야 한다. 1~2명이 사진·문서만 보관한다면 UPS를 네트워크 모델로 바꾸기보다 백업 디스크를 작업 때만 연결하는 편이 저렴하다. 반대로 TrueNAS SCALE 25.10이나 QNAP QTS 5.2에서 여러 장비를 자동 종료한다면 UPS는 USB·SNMP·네트워크 방식으로 분리하고 백업 디스크를 별도 장치로 둔다. 2026년 11월 13일 공식 문서 기준이며 특정 장비를 직접 시험한 결과는 아니다.

## 포트 하나에 장치를 몰아 꽂으면 생기는 문제

| 환경 | UPS 연결 | 외장 백업 | 구매 판단 |
|---|---|---|---|
| Synology DSM 7.4, USB 포트 1개 | UPS 직접 연결 | 다른 시간에 디스크 연결 | 허브는 사지 않음 |
| QNAP QTS 5.2, NAS 1대 | USB 또는 SNMP | 후면 USB·eSATA | UPS 통신을 우선 확인 |
| TrueNAS SCALE 25.10, 서버 2대 이상 | NUT master 또는 SNMP | 별도 백업 서버·디스크 | 네트워크 UPS 구성이 편함 |
| USB Zigbee 동글까지 사용 | 호스트의 직접 연결 필요 | NAS 외부 또는 다른 서버 | 별도 Home Assistant 호스트 우선 |

QNAP은 외장 USB·eSATA 저장장치를 지원하고, UPS는 USB 연결 뒤 network UPS master로 다른 장비에 상태를 전달하거나 SNMP로 연결할 수 있다. TrueNAS는 NUT master가 네트워크로 다른 컴퓨터에 상태를 전달한다. 포트 부족은 USB를 늘리는 문제가 아니라 UPS 통신 경로를 바꾸는 문제다.

## 가상 비용으로 비교하는 세 가지 구성

실제 판매가가 아닌 비교용 가정이다. UPS 12만 원, 3년 뒤 배터리 5만 원, 외장 백업 디스크 15만 원으로 잡았다.

| 구성 | 초기 장비비 | 3년 소모품 포함 | 적합한 경우 |
|---|---:|---:|---|
| USB UPS + 교대 연결 백업 디스크 | 27만 원 | 32만 원 | NAS 1대, 백업을 수동 실행 |
| USB UPS + 별도 백업 서버 | 42만 원 이상 | 47만 원 이상 | 자동 백업과 종료를 모두 원함 |
| SNMP·네트워크 UPS | 45만 원 이상 | 50만 원 이상 | NAS·미니PC·스위치가 2대 이상 |

허브를 2만 원 추가해도 Synology의 미지원 조건은 바뀌지 않는다. 구매 전에는 `NAS 모델 → OS 버전 → UPS 연결 방식 → 백업 경로` 순서로 확인한다. Zigbee 동글을 NAS에 꽂는다면 Docker 권한과 재부팅 후 장치 경로도 따로 검증한다.

## 구매 전 10분 점검표

- [ ] NAS 모델의 UPS 호환 목록과 USB 포트 수를 확인했다.
- [ ] UPS의 USB·SNMP·network slave 지원 여부를 확인했다.
- [ ] 외장 백업 디스크를 다른 장치에서 읽을 수 있는지 정했다.
- [ ] 정전 모의시험에서 신호 수신·서비스 중지·볼륨 언마운트를 확인한다.
- [ ] Zigbee 동글은 별도 Home Assistant 호스트에 둘 수 있는지 검토했다.

판단식은 `3년 총비용 = UPS + 배터리 교체 + 백업 저장장치 + 네트워크 관리 비용 + 복구 시간`이다. 복구 파일을 다른 장치에서 열어 본 적이 없다면 포트 확장보다 백업 검증이 먼저다.

짧게 정리하면 Synology는 UPS를 직접 연결하고 백업 디스크를 교대 사용하는 구성이 가장 단순하다. QNAP은 USB·SNMP·네트워크 UPS 중 모델에 맞는 방식을 선택할 수 있다. TrueNAS SCALE은 NUT master/slave로 여러 장비를 묶기 좋지만 스위치까지 UPS 전원에 연결해야 한다. 허브를 사도 공식 호환성이 생기지 않으므로, 포트가 부족한 NAS라면 네트워크 UPS 또는 별도 백업 호스트 중 하나를 선택하는 것이 실제 비용과 복구 가능성을 함께 낮춘다.

참고한 공식 문서: [Synology 외부 장치](https://kb.synology.com/en-br/DSM/help/DSM/AdminCenter/system_externaldevice_desc?version=7), [Synology UPS](https://kb.synology.com/en-ph/DSM/help/DSM/AdminCenter/system_hardware_ups?version=7), [QNAP QTS 5.2 외장 저장장치](https://docs.qnap.com/operating-system/qts/5.2.x/en-us/external-storage-B6C934B5.html), [QNAP UPS 설정](https://docs.qnap.com/operating-system/qts/5.0.x/en-us/configuring-the-ups-settings-2AD2FE5E.html), [TrueNAS SCALE UPS 서비스](https://www.truenas.com/docs/scale/systemsettings/services/upsservices/).

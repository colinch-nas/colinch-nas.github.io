---
layout: post
title: "NAS CCTV 카메라 2·4·8대 선택 - Synology DSM 7·QNAP QTS 5.2 저장장치와 총비용"
description: "Synology DSM 7 Surveillance Station과 QNAP QTS 5.2 QVR Pro를 카메라 2·4·8대 기준으로 비교하고, 30일 녹화 용량·라이선스·복구 비용까지 계산한다."
date: 2026-11-12
tags: [Synology, QNAP, DSM, HomeLab, NAS, 보안]
comments: true
share: true
---

![NAS CCTV 카메라와 저장장치 구성](https://images.unsplash.com/photo-1557597774-9d273605dfa9?w=1200&q=80)
*이 그림에서 볼 부분은 카메라 가격보다 녹화 기간, 라이선스, NAS 디스크 교체 경로가 함께 비용을 만든다는 점이다.*

카메라가 2대면 기존 NAS로 시작해도 되지만, 4대부터는 무료 채널 수와 30일 저장공간을 함께 계산해야 한다. Synology DSM 7의 Surveillance Station은 일반 NAS에 기본 2개 라이선스가 있고, QNAP QTS 5.2의 QVR Pro는 추가 채널 라이선스와 QVR Pro Gold 조건을 확인해야 한다. 카메라가 1~2대이고 이미 클라우드 녹화를 쓰는 집이라면 NAS CCTV를 새로 사지 않는 편이 낫다. 아래 수치는 장비를 직접 시험한 결과가 아니라 2026년 11월 12일 공식 문서와 명시한 가정으로 계산한 구매 기준이다.

## 카메라 수와 함께 정할 조건

| 환경 | 우선 확인할 것 | 구매 판단 |
|---|---|---|
| 2대, 7일 보관 | 기본 채널, 기존 여유 용량 | 기존 NAS와 별도 HDD로 충분할 수 있음 |
| 4대, 30일 보관 | 무료 채널 초과, HDD 내구성 | 4베이 NAS와 전용 감시용 HDD를 비교 |
| 8대, 30일 이상 | 채널 라이선스, CPU·메모리, 복구 시간 | NAS 한 대에 몰지 말고 백업·NVR 대안을 계산 |

Synology는 카메라 모델별 지원 목록을 확인하는 게 기준이다. 목록에 없는 카메라도 ONVIF나 RTSP로 연결할 수 있지만, 연결 성공과 모든 이벤트 기능 지원은 같은 뜻이 아니다. 특히 H.265 카메라는 Surveillance Station 9.2.1 이상에서 일부 움직임 감지 방식이 카메라 측 감지로 바뀔 수 있다. QNAP도 카메라 연결 후 스트림과 녹화 설정을 확인해야 하며, QVR Pro 문서에는 1·4·8채널 확장 라이선스가 별도 항목으로 안내된다.

## 30일 녹화 용량 계산

가상 계산은 카메라 한 대당 평균 4Mbps, H.265, 24시간 녹화다. `4Mbps × 86,400초 ÷ 8`이면 하루 약 43.2GB다. 실제 용량은 움직임 녹화, 오디오, 비트레이트, 보존 정책에 따라 달라진다.

| 카메라 | 이론상 30일 | 20% 여유 포함 권장치 |
|---:|---:|---:|
| 2대 | 약 2.6TB | 약 3.1TB |
| 4대 | 약 5.2TB | 약 6.2TB |
| 8대 | 약 10.4TB | 약 12.4TB |

이 계산은 RAID usable 용량 기준이 아니다. 예를 들어 4대 구성에서 8TB 디스크 두 개를 미러로 묶으면 표시 용량은 약 8TB지만, 파일시스템 여유와 장애 대응 공간을 남기면 30일 보관 목표가 빠듯해질 수 있다. 녹화 전용 볼륨을 만들고, 중요한 영상은 NAS 밖으로 내보낼 복사 경로도 정해야 한다.

## 라이선스와 총비용을 같이 보기

가상 사례로 4대 카메라, 30일 보관, 기존 네트워크 스위치 보유 조건을 잡았다. 실제 판매가는 지역·모델·시점에 따라 달라서 숫자를 단정하지 않고 비용 항목만 비교한다.

| 항목 | Synology DSM 7 | QNAP QTS 5.2 |
|---|---|---|
| 기본 채널 | 일반 NAS 기본 2개 | 모델·앱 조건 확인 필요 |
| 4대 구성 | 추가 2개 라이선스 | QVR Pro Gold와 채널 확장 조건 확인 |
| 저장장치 | 6.2TB 이상 usable + 여유 | 같은 기준, QVR 녹화 경로 확인 |
| 이전·복구 | 구매 라이선스는 이전 가능, 기본 라이선스는 이전 불가 | License Center와 QNAP ID 복구 절차 확인 |
| 사지 않아도 되는 경우 | 2대·짧은 보관·기존 여유 용량 | 같은 조건, 별도 NVR이 더 단순할 수 있음 |

카메라 본체와 HDD만 합산하면 안 된다. `카메라 × 대수 + NAS 또는 NVR + 녹화용 HDD + 추가 라이선스 + UPS + 외부 백업 디스크`가 초기비용이고, 여기에 고장 디스크 교체와 녹화 영상 내보내기 시간이 운영비로 붙는다. 8대 이상이면 라이선스보다 NAS의 동시 녹화 성능과 복구 시간을 우선 확인한다. 구매 전 각 카메라의 해상도·코덱·ONVIF 프로파일을 적어 두면 “연결은 되지만 이벤트 검색이 안 되는” 교체 비용을 줄일 수 있다.

## 설치 전 복구 체크리스트

- [ ] 카메라 모델, 펌웨어, H.264/H.265, ONVIF·RTSP 지원을 기록한다.
- [ ] 2·4·8대 중 실제 채널 수와 기본·추가 라이선스를 구분한다.
- [ ] 목표 보관일과 비트레이트로 usable 용량을 계산하고 20% 이상 여유를 둔다.
- [ ] NAS 장애 때 카메라의 엣지 녹화나 외부 백업에서 영상을 꺼낼 수 있는지 확인한다.
- [ ] 샘플 영상 1개를 다른 PC에서 재생하고, NAS 교체 후 설정·라이선스 복구 절차를 문서화한다.

Synology의 공식 안내는 지원 목록, Generic/ONVIF, 사용자 정의 RTSP 경로를 구분한다. QNAP은 QVR Pro Gold를 활성화한 뒤 채널 확장 라이선스를 설치하는 순서를 안내한다. 따라서 가장 싼 조합은 “카메라를 많이 연결하는 조합”이 아니라, 목표 보관 기간과 복구 경로를 끝까지 확인한 조합이다. 2대·7일 보관이면 기존 NAS를 재활용하고 외부 백업만 추가하는 선택이 합리적이고, 4대 이상이면 라이선스와 HDD 교체 비용을 본체 가격과 함께 비교해야 한다.

참고 문서: [Synology Surveillance Station 라이선스](https://www.synology.com/en-ca/store/DEVICE%20LICENSE%20%28X%201%29), [Synology 카메라 호환·ONVIF 안내](https://kb.synology.com/en-global/SurveillanceStation/help/SurveillanceStation/Tutorial/generic_camera_setup), [Synology H.265 변경 안내](https://kb.synology.com/en-in/Surveillance/tutorial/What_functions_will_be_adjusted_due_to_the_change_in_support_for_H265_on_Surveillance_Station_and_DS_cam), [QNAP QVR Pro 사용자 안내서](https://docs.qnap.com/application/qvr-pro/qvrpro1.4-ug-03-en-us.pdf), [QNAP QTS 5.2 라이선스 안내](https://docs.qnap.com/operating-system/qts/5.2.x/en-us/acquiring-apps-and-licenses-266E0577.html)

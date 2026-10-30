---
layout: post
title: "Synology·QNAP NAS CCTV 2·4·8대 - 보관기간·라이선스·HDD 총비용 판단"
description: "Synology Surveillance Station과 QNAP QVR 기준으로 카메라 2·4·8대의 저장 용량, 라이선스, 감시용 HDD와 백업 비용을 계산해 NAS와 전용 NVR 선택 기준을 정리한다."
date: 2026-10-31
tags: [Synology, QNAP, NAS, HomeLab, NAS보안]
comments: true
share: true
---

![NAS에 연결된 실외 보안 카메라와 저장장치](https://images.unsplash.com/photo-1558008258-3256797b43f3?auto=format&fit=crop&w=1600&q=80)

Synology Surveillance Station이나 QNAP QVR에서 카메라가 2대이고 30일 보관이면 기존 NAS의 여유 공간으로 시작할 수 있지만, 4대 이상을 24시간 녹화한다면 베이 수보다 먼저 비트레이트와 라이선스를 계산해야 한다. 파일 저장과 CCTV를 함께 쓸 계획이면 NAS가 유리하고, 녹화만 필요하고 카메라 수가 많으면 전용 NVR이 더 단순하다. 아래 수치는 2026년 10월 31일 공식 문서를 확인한 기준이며 실제 카메라 설정에 따라 달라지는 계산 예시다.

## 카메라 수보다 보관기간이 용량을 결정한다

계산식은 `카메라 수 × 평균 비트레이트(Mbps) × 86,400 ÷ 8`이다. 가상으로 H.265 카메라 4대가 각각 4Mbps로 연속 녹화한다고 가정하면 하루 약 172.8GB, 30일 약 5.18TB가 나온다. 파일시스템 여유와 녹화 변동을 20% 잡으면 약 6.2TB를 준비해야 한다.

| 구성 예시 | 계산 가정 | 30일 원본 | 구매 판단 |
|---|---:|---:|---|
| 2대 | 2Mbps씩 | 약 1.30TB | 2베이 RAID1과 별도 백업으로 시작 가능 |
| 4대 | 4Mbps씩 | 약 5.18TB | 파일 저장과 분리할 4베이가 편함 |
| 8대 | 4Mbps씩 | 약 10.37TB | 전용 NVR 또는 6베이 이상을 검토 |

이 그림에서 봐야 할 것은 카메라 화소보다 실제 녹화 비트레이트와 보관일수가 구매 용량을 좌우한다는 점이다. Synology Surveillance Station도 녹화 설정에서 필요한 공간을 추정하며, 사전 녹화 시간이 길어지면 NAS가 처리할 수 있는 카메라 수가 줄 수 있다고 안내한다([공식 녹화 설정 문서](https://kb.synology.com/en-sg/SurveillanceStation/help/SurveillanceStation/recording_settings_recording?version=9)).

## NAS와 전용 NVR의 비용 차이는 디스크 밖에서 난다

Synology NAS는 기본 카메라 라이선스가 최소 2개지만 추가 카메라에는 별도 라이선스가 필요하다([Synology 구매 가이드](https://event.synology.com/en-sg/surveillance-buyers-guide)). QNAP은 QVR NAS Selector에서 카메라 수와 저장기간을 입력해 필요한 용량과 지원 NAS를 계산한다([QNAP 선택기](https://www.qnap.com/en/qvr-nas-selector)). 따라서 총비용은 본체와 HDD만 더하면 안 된다.

| 항목 | NAS를 이미 보유한 경우 | CCTV 전용으로 새로 사는 경우 |
|---|---|---|
| 본체 | 추가 비용 0원일 수 있음 | NAS 또는 NVR 본체 필요 |
| 저장장치 | 기존 여유 공간 확인 | 녹화 전용 HDD와 RAID 여유 필요 |
| 라이선스 | 카메라 3대부터 확인 | NVR 포함 라이선스 정책 확인 |
| 백업 | 녹화 파일까지 외장 디스크에 복사 | 장애 시 교체 장비와 보관 디스크 필요 |
| 전기·UPS | 기존 소비전력에 추가 | 24시간 본체·스위치·카메라를 함께 계산 |

감시용 HDD는 무조건 필수라는 뜻은 아니다. Synology도 HDD 선택에서 감시용 HDD와 일반 NAS용 HDD를 사용 환경별로 구분한다([HDD 선택 기준](https://kb.synology.com/en-sg/DSM/tutorial/How_to_choose_the_right_HDD_for_my_Synology_NAS)). Seagate SkyHawk의 공식 사양은 일반형 180TB/년, AI형 550TB/년 작업량을 제시하므로, 카메라 수와 지속 쓰기량이 낮은 가정용이면 호환 목록에 있는 NAS HDD가 구매비를 줄일 수 있다([SkyHawk 사양](https://www.seagate.com/products/specialized-drives/video-analystics/skyhawk-hard-drive/)).

## 구매 전 복구 체크리스트

- 카메라별 평균 비트레이트와 야간 모드를 확인한다.
- `카메라 수 × 일일 용량 × 보관일수 × 1.2`로 여유 용량을 계산한다.
- NAS 모델의 녹화 카메라 수, 코덱, 사전 녹화 지원 여부를 확인한다.
- Synology는 추가 라이선스 수, QNAP은 QVR 지원 채널과 QTS·QuTS hero 조건을 확인한다.
- RAID는 본체 장애와 삭제를 막는 백업이 아니므로, 중요한 이벤트 영상은 다른 디스크나 오프사이트에 복사한다.
- 테스트로 1개 카메라의 24시간 녹화 파일을 삭제한 뒤, 백업에서 다른 폴더로 복원하고 재생한다.

짧게 판단하면 2대·7일 보관은 기존 NAS 활용이 경제적이고, 4대·30일 보관부터는 저장공간과 백업 디스크를 별도 예산으로 잡아야 한다. 8대 이상이거나 영상 분석까지 켜면 NAS의 파일 서비스 비용과 카메라 라이선스를 합산해 전용 NVR과 비교하는 편이 안전하다.

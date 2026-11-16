---
layout: post
title: "NAS CCTV 저장용량 계산 - Synology DSM 7.4·QNAP QTS 5.2에서 1·4·8대 카메라 총비용"
description: "Synology DSM 7.4와 QNAP QTS 5.2로 CCTV를 녹화할 때 카메라 수·비트레이트·보관일수·HDD와 라이선스·백업까지 계산해 NAS 구매 여부를 판단한다."
date: 2026-11-16
tags: [Synology, QNAP, DSM, QTS, NAS, HomeLab]
comments: true
share: true
---

![NAS CCTV 녹화 저장용량과 하드디스크 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

*이 그림에서 볼 것은 NAS 본체보다 카메라 수와 연속 녹화용 디스크가 비용을 좌우한다는 점이다.*

카메라 1~2대를 움직임 감지로 짧게 보관한다면 기존 2베이 NAS에 녹화를 추가해도 된다. 4대 이상을 24시간 녹화하거나 30일 보관하려면 NAS의 CPU·카메라 라이선스·HDD 여유 공간·별도 백업을 함께 계산해야 한다. 이 글은 장비를 직접 시험한 결과가 아니라 2026년 11월 16일 확인한 제조사 문서와 비트레이트 가정에 따른 구매 판단이다.

## 카메라 수보다 먼저 보관일수를 계산한다

가상 계산은 카메라 1대당 평균 4Mbps, 하루 24시간, H.265 연속 녹화로 잡았다. 실제 카메라의 설정·움직임·코덱에 따라 달라지므로 구매 전 카메라가 표시하는 평균 비트레이트를 넣어 다시 계산해야 한다.

`4Mbps × 86,400초 ÷ 8 = 약 43.2GB/일`

| 카메라 | 7일 원본 | 30일 원본 | 20% 여유 포함 |
|---:|---:|---:|---:|
| 1대 | 302GB | 1.30TB | 1.56TB |
| 4대 | 1.21TB | 5.18TB | 6.22TB |
| 8대 | 2.42TB | 10.37TB | 12.44TB |

RAID 표기 용량을 그대로 보지 말고, 파일시스템·스냅샷·교체 여유를 빼야 한다. 4베이에 8TB HDD 4개를 넣어도 RAID5의 이론상 데이터 영역은 약 24TB이며, 여기에 백업 디스크를 따로 마련해야 한다. 녹화 전용 풀과 가족 사진·문서 풀을 분리하지 않으면 카메라의 지속 쓰기가 일반 파일 서비스와 장애 범위를 공유한다.

## DSM 7.4와 QTS 5.2의 구매 차이

| 조건 | Synology DSM 7.4 | QNAP QTS 5.2 | 구매 판단 |
|---|---|---|---|
| 1~2대, 7일, 움직임 녹화 | Surveillance Station 기본 라이선스 범위 확인 | QVR Elite 기본 채널 범위 확인 | 기존 NAS 재사용 우선 |
| 4대, 30일, 연속 녹화 | NVR Selector와 카메라 호환 목록 확인 | QVR NAS Selector에서 채널·용량 확인 | 4베이 이상과 감시용 HDD 검토 |
| 8대, 분석·원격 재생 | 모델별 FPS·분석 한도 확인 | QVR Elite 채널·RAM·앱 제한 확인 | 전용 NVR 또는 상위 NAS도 비교 |

Synology 공식 사양은 Surveillance Station이 20,000개 이상의 IP 카메라와 ONVIF 장치를 지원한다고 안내하지만, 실제 채널 수와 분석 기능은 모델·해상도·프레임 수에 따라 달라진다. QNAP은 QVR NAS Selector에서 필요한 저장용량과 카메라 수를 계산하도록 제공한다. “NAS에 앱을 설치할 수 있음”과 “원하는 카메라 8대를 안정적으로 녹화함”은 같은 말이 아니다.

## 백업과 총비용에서 빠지는 항목

녹화 영상은 원본 사진보다 다시 만들 수 없을 때가 많지만, 항상 전부 백업할 필요는 없다. 사건 발생 시 보관할 클립만 다른 장치에 복사하고, 24시간 영상은 보관기간이 끝나면 삭제하는 정책이 비용을 낮춘다.

- NAS 본체와 감시용 HDD: RAID 여유와 교체용 디스크를 포함한다.
- 라이선스: Synology 추가 카메라 라이선스와 QNAP QVR 제품별 채널 정책을 확인한다.
- 전기료·UPS: 연속 녹화 NAS와 PoE 스위치가 함께 켜지는 시간을 계산한다.
- 복원 경로: 중요한 클립을 다른 NAS·외장 디스크·클라우드 중 한 곳에 복사하고 실제 재생까지 확인한다.

구매 전에는 `카메라 수 → 평균 Mbps → 보관일수 → 20% 여유 → RAID usable → 백업 대상` 순서로 적는다. 이 계산에서 2베이의 유효 용량이 부족하면 디스크만 큰 것으로 해결하지 말고 4베이 NAS나 별도 NVR을 비교한다. 반대로 1대 카메라를 7일만 보관하고 이미 백업 가능한 NAS가 있다면 새 NAS와 SSD를 살 이유가 적다.

참고: [Synology Surveillance Station 공식 사양](https://www.synology.com/en-us/dsm/7.4/software_spec/surveillance_station), [QNAP QVR NAS Selector 안내](https://www.qnap.com/en/how-to/tutorial/article/how-to-add-recording-space-and-manage-channels-on-qvr-surveillance)

---
layout: post
title: "Jellyfin 4K 트랜스코딩 - Intel N100/N150 미니PC와 ARM NAS 총비용 판단"
description: "Jellyfin 10.11 기준 4K HDR 트랜스코딩을 위해 Intel N100·N150 미니PC와 ARM NAS를 비교하고, 사용자 수·톤매핑·전력·백업까지 포함한 선택 기준을 정리한다."
date: 2026-10-27
tags: [Jellyfin, HomeLab, Docker, NAS, Intel, ARM]
comments: true
share: true
---

![Jellyfin 4K 트랜스코딩용 Intel 미니PC와 ARM NAS 비교](https://images.unsplash.com/photo-1518770660439-4636190af475?w=1200&q=80)

*이 그림에서 볼 부분은 저장용 NAS와 트랜스코딩용 장비를 한 상자에 넣을지 분리할지다.*

Jellyfin으로 집 안 재생만 하고 모든 기기가 원본 영상을 직접 재생한다면 기존 ARM NAS를 바꿀 필요가 없다. 반대로 외부 접속, 자막 번인, 4K HDR을 SDR 화면으로 바꾸는 톤매핑이 필요하면 Intel N100/N150 미니PC를 별도 서버로 두는 편이 실패 비용이 낮다. ARM NAS가 무조건 부족한 것은 아니지만, 모델별 VPU와 컨테이너 권한 확인 없이는 구매 판단을 하기 어렵다.

## 호환성에서 먼저 볼 것

Jellyfin 공식 문서는 Intel QSV, VA-API, NVIDIA NVENC, AMD AMF, Rockchip RKMPP 등을 하드웨어 가속 방식으로 안내한다. 다만 가속은 디코딩·스케일링·톤매핑·자막 번인을 모두 해결한다는 뜻이 아니다. 특히 4K Dolby Vision을 4K H.264 SDR로 변환하는 작업은 일반적인 한 스트림 기준도 아니므로 “4K 지원” 문구만 보고 고르면 안 된다.

| 구성 | 잘 맞는 조건 | 걸리는 조건 | 구매 판단 |
|---|---|---|---|
| ARM NAS 단독 | 원본 재생, 1~2명, 호환 클라이언트 | 자막 번인·HDR 톤매핑, 모델별 VPU 차이 | 기존 장비가 있으면 추가 구매 없음 |
| N100/N150 미니PC + NAS | 외부 재생, 2~4명, Docker 분리 | 미니PC·SSD·전원 비용 추가 | 가장 무난한 분리형 |
| 상위 x86 NAS 단독 | VM·컨테이너·미디어를 한 장비에서 운영 | 초기 가격과 전력, 장애 범위가 큼 | 새로 구매할 때만 검토 |

Jellyfin 10.11부터는 x86 CPU의 SSE4.1 요구사항도 확인해야 한다. 공식 하드웨어 선택 문서는 Intel 12세대 N 시리즈를 후보로 들지만, “동시에 몇 개의 4K 스트림”을 보장하지 않는다. 실제 결과는 입력 코덱, 프레임레이트, 자막 종류, 네트워크, 톤매핑 여부에 따라 달라진다.

## 사용자 규모별 구성

가상 사례로 1인 시청, 가족 2명, 외부 사용자 4명을 나눴다. 아래의 스트림 수는 제조사 보장값이 아니라 구매 전 테스트를 나누는 기준이다.

| 사용 규모 | 권장 구조 | 확인할 항목 |
|---|---|---|
| 1~2명·집 안 | ARM NAS 단독 | Direct Play 비율, 자막 형식 |
| 2~4명·외부 포함 | N100/N150 + NAS 저장소 | QSV/VA-API 장치 전달, HDR 톤매핑 |
| 4명 이상·동시 변환 | 더 높은 x86 또는 GPU 서버 | 실제 샘플 파일 2~3개로 동시 재생 |

미니PC를 쓰면 Jellyfin 컨테이너는 `/dev/dri` 장치를 받아야 하고, 호스트와 Docker 이미지의 드라이버·FFmpeg 조합도 맞아야 한다. 설정을 했는데도 Dashboard의 Playback 정보가 `Software`로 남으면 CPU만 쓰는 상태다. 재생 중 해상도를 낮춰 트랜스코딩을 일으킨 뒤 GPU 사용률과 로그를 확인해야 한다.

## 전력과 총비용

아래는 가격을 임의로 단정하지 않기 위한 가상 계산식이다. 2026년 10월 27일의 실제 판매가는 모델과 SSD 용량에 따라 다시 확인한다.

```text
5년 총비용 = 장비 가격 + SSD 가격 + (평균 소비전력(W) × 24 × 365 × 5 ÷ 1000 × kWh 단가)
```

기존 NAS를 저장 전용으로 계속 쓰고 N100 미니PC만 추가하면 초기 비용은 늘지만, NAS를 고성능 모델로 교체하는 비용과 장애 시 저장소까지 함께 멈추는 위험을 분리할 수 있다. 반대로 Jellyfin을 거의 Direct Play로 쓰는 집이라면 미니PC와 전기료를 추가하지 않는 것이 더 합리적이다. 백업은 어느 구성에서도 별도다. 미디어 원본, Jellyfin 메타데이터, Docker Compose와 설정 폴더를 다른 장치에 보관해야 한다.

## 구매 전 체크리스트

- [ ] 재생 기기별로 H.264·HEVC·AV1 Direct Play 여부를 적었다.
- [ ] 4K HDR 샘플과 자막 포함 샘플을 각각 준비했다.
- [ ] ARM NAS의 VPU, Docker 권한, Jellyfin 이미지 아키텍처를 확인했다.
- [ ] N100/N150 미니PC의 `/dev/dri`와 Linux 드라이버 지원을 확인했다.
- [ ] 미니PC 장애 때 NAS 파일과 Jellyfin 설정을 복원할 백업이 있다.

구매하지 않아도 되는 조건은 분명하다. 모든 클라이언트가 원본 재생을 하고, 외부 사용자와 톤매핑이 없으며, 현재 ARM NAS의 CPU 사용률이 안정적이면 새 장비보다 백업과 네트워크 점검이 우선이다. 반대로 4K HDR 변환이 핵심이면 “NAS에 Jellyfin이 설치된다”보다 하드웨어 가속이 실제로 켜지는지가 선택 기준이다.

출처: [Jellyfin 하드웨어 가속 공식 문서](https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/), [Jellyfin 하드웨어 선택 가이드](https://jellyfin.org/docs/general/administration/hardware-selection/), [Jellyfin Intel 가속 문서](https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/intel/)

---
layout: post
title: "Jellyfin 하드웨어 트랜스코딩 NAS 호환성 - 1·3·5인 Intel Quick Sync와 전기료 총비용"
description: "Jellyfin 공식 Docker를 NAS에 운영할 때 Intel Quick Sync 호환성, 1·3·5인 동시 재생 조건, 별도 미니PC와 GPU의 전기료 총비용을 비교한다."
date: 2026-10-17
tags: [Jellyfin, Intel, QuickSync, Docker, NAS, HomeLab]
comments: true
share: true
---

![Jellyfin 미디어 서버와 NAS 하드웨어 트랜스코딩 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

이 글의 선택 기준은 단순하다. 집 안 기기 대부분이 원본을 직접 재생하면 기존 NAS를 유지하고, 외부 접속·자막 번인·4K HDR 변환이 반복되면 Intel Quick Sync가 있는 NAS나 별도 미니PC가 유리하다. AMD 기반 NAS처럼 iGPU가 없거나 Docker에서 `/dev/dri`를 넘길 수 없는 장비라면 “지원한다”는 CPU 사양만 보고 구매하면 안 된다.

## 1·3·5인 기준 선택표

동시 인원은 가상 계획값이다. 실제 부담은 영상 코덱, 해상도, 자막 번인, 클라이언트가 지원하는 원본 형식에 따라 달라진다.

| 사용 상황 | 권장 구성 | 피할 구성 | 비용 판단 |
|---|---|---|---|
| 1인, 집 안 TV·PC 위주 | 기존 NAS + Direct Play | CPU만으로 4K 변환 | 새 장비를 사지 않는 편이 유리 |
| 3인, 모바일·원격 재생 혼합 | Intel N100/N150 NAS 또는 미니PC | ARM NAS에 무조건 Docker 설치 | 6~10W급 호스트가 별도 GPU보다 단순 |
| 5인, 4K HDR·자막 번인 반복 | Intel i3 이상 미니PC 또는 검증된 GPU | 보급형 NAS 단독 운영 | 호스트와 UPS·백업 비용까지 계산 |

Intel N150은 Intel 사양표에서 Quick Sync Video를 지원하고 프로세서 기본 전력은 6W다. 다만 이 수치는 시스템 전체 소비전력이 아니다. 메모리, NVMe, 팬, 전원 어댑터, NAS 디스크를 더해야 한다. 6W라는 숫자만으로 월 전기료를 예측하면 틀린다.

## NAS 내부 트랜스코딩과 별도 미니PC 비교

Jellyfin은 Intel QSV, NVIDIA NVENC, AMD VA-API 등 여러 가속 방식을 지원한다. 하드웨어가 있어도 디코딩·톤 매핑·자막 번인 중 일부가 소프트웨어로 남으면 CPU 사용률이 올라갈 수 있다. 공식 문서도 파이프라인의 일부 단계는 하드웨어 가속이 제한될 수 있다고 설명한다.

| 항목 | QSV 내장 NAS | 별도 Intel 미니PC | NVIDIA GPU 서버 |
|---|---:|---:|---:|
| 추가 장비 | 없음 | 본체·SSD 필요 | GPU·전원·냉각 필요 |
| 저장장치 접근 | 단순 | SMB/NFS 권한 필요 | 공유 스토리지 구성 필요 |
| 전력 가정 | NAS 전체 전력 | 호스트 전력 + NAS 대기전력 | GPU 유휴·부하 전력 변동 큼 |
| 복구 부담 | NAS 설정 백업 | Compose·호스트 백업 | 드라이버·컨테이너까지 백업 |
| 적합한 경우 | 1~3인, 단순 구성 | NAS CPU가 약하거나 5인 이상 | 이미 GPU가 있고 고부하가 반복됨 |

전기료는 아래처럼 계산하면 된다. 예를 들어 별도 미니PC가 평균 12W, NAS가 미디어 재생 중 평균 35W, 전기요금을 kWh당 160원으로 가정하면 1년 비용은 `(0.012 + 0.035) × 24 × 365 × 160 = 약 66,000원`이다. 이 값은 실제 측정값이 아니라 비교용 가정이다. 구매 전 콘센트형 전력계로 유휴·재생·트랜스코딩 세 구간을 따로 측정해야 한다.

## Docker에서 호환성을 확인하는 방법

Linux 호스트라면 GPU 장치가 있는지 확인한 뒤, Jellyfin 컨테이너에 전달한다. 명령은 장비에서 직접 실행하기 전 경로와 그룹 권한을 확인해야 한다.

```bash
# Intel/AMD 그래픽 장치와 render 노드가 있는지 확인한다.
ls -l /dev/dri
getent group render
```

Compose에는 보통 아래처럼 render 장치를 넣는다. Synology DSM의 Container Manager, TrueNAS SCALE 앱, Proxmox LXC는 권한 전달 방식이 서로 달라 이 블록을 그대로 붙여 넣으면 안 된다.

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    devices:
      - /dev/dri/renderD128:/dev/dri/renderD128
    group_add:
      - "RENDER_GROUP_ID"
```

Jellyfin 대시보드의 재생 정보에서 `Transcoding (하드웨어)`가 표시되는지, 로그에서 QSV 또는 VA-API가 선택됐는지 확인한다. 영상이 재생됐다는 사실만으로 가속 성공을 판단하면 안 된다. Direct Play였을 가능성이 있기 때문이다. 테스트 파일은 H.264 1080p, HEVC 4K, 자막 번인이 필요한 파일 세 가지로 나누는 편이 낫다.

## 구매하지 않아도 되는 조건과 백업 항목

다음 조건이면 새 NAS나 GPU를 사지 않아도 된다.

- TV·PC·모바일 앱이 HEVC와 오디오 코덱을 직접 재생한다.
- 외부 사용자가 없고, 동시 재생이 1명 이하라서 Direct Play가 유지된다.
- 현재 NAS에서 CPU 사용률과 재생 끊김이 문제가 되지 않는다.

반대로 미니PC를 추가한다면 미디어 폴더만 연결하지 말고 Jellyfin의 `/config`, `/cache` 중 `/config`, Compose 파일, 사용자·라이브러리 메타데이터를 백업한다. 새 호스트에서 복원한 뒤 계정 로그인, 라이브러리 스캔, 자막 번인, 원격 접속을 확인해야 한다. 미디어 원본은 NAS에 남아도 서버 설정이 없으면 운영 복구 시간이 길어진다.

판단을 한 줄로 줄이면 이렇다. 1인 Direct Play는 기존 NAS, 3인 이하의 가끔 있는 변환은 QSV NAS, 5인·원격·4K HDR 변환은 별도 Intel 미니PC를 우선 검토한다. 전용 GPU는 이미 보유했거나 반복적인 고부하가 확인될 때만 총비용이 맞는다.

참고한 공식 문서:

- [Jellyfin 하드웨어 가속 공식 문서](https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/)
- [Jellyfin 컨테이너 설치와 render 장치 전달](https://jellyfin.org/docs/general/installation/container/)
- [Intel Processor N150 사양표](https://www.intel.com/content/www/us/en/products/sku/241636/intel-processor-n150-6m-cache-up-to-3-60-ghz/specifications.html)

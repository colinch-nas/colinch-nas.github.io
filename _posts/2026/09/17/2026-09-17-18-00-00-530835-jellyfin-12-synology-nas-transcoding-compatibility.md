---
layout: post
title: "Jellyfin 12.0 Synology NAS 호환성 - 동시 시청자 수별 트랜스코딩 구매 기준"
description: "Jellyfin 12.0을 Synology NAS에서 운영할 때 DS224+와 DS923+의 트랜스코딩 차이, 사용자 수·자막 조건, 별도 미니PC가 유리한 구매 기준을 정리한다."
date: 2026-09-17
tags: [Jellyfin, Synology, NAS, Docker, HomeLab]
comments: true
share: true
---
![Jellyfin 미디어 서버를 실행하는 NAS와 TV 화면](https://images.unsplash.com/photo-1593305841991-05c297ba4575?w=1200&q=80)

집 안에서 1명이 지원되는 파일을 직접 재생한다면 Synology NAS에 Jellyfin 12.0을 올려도 별도 서버를 살 필요가 없다. 반대로 외부 사용자가 2명 이상이고 4K HEVC, HDR, 자막 번인을 자주 쓰면 NAS 모델명보다 하드웨어 가속 지원 여부가 먼저다. 이 글은 실제 장비 테스트 결과가 아니라 2026년 9월 17일 확인한 공식 사양과 Jellyfin 문서에 따른 구매 판단 기준이다.

## DS224+와 DS923+를 같은 NAS로 보면 안 되는 이유

| 환경 | 확인할 조건 | 구매 판단 |
|---|---|---|
| 1명·거실 TV·Direct Play | TV가 영상·음성 코덱을 모두 지원 | 기존 NAS에 설치, 업그레이드 보류 |
| 1~2명·가끔 1080p 변환 | Intel Quick Sync와 권한 설정 가능 | Intel iGPU NAS를 우선 검토 |
| 2명 이상·4K HDR·자막 번인 | GPU 가속, HDR 톤 매핑, SSD 캐시 필요 | 별도 Intel 미니PC+기존 NAS가 안전 |
| 파일·사진·컨테이너가 주용도 | Jellyfin은 직접 재생 위주 | 트랜스코딩 때문에 고가 NAS를 사지 않는다 |

Synology DS224+의 공식 사양은 Intel Celeron J4125, 2GB 메모리, 2베이이며 접근 시 소비전력은 14.69W다. DS923+는 AMD Ryzen R1600, 4GB 메모리, 4베이다. 그러나 베이 수가 많다고 Jellyfin 트랜스코딩이 자동으로 좋아지는 것은 아니다. Jellyfin 공식 문서가 검증한 가속 방식은 Intel QSV, NVIDIA NVENC, AMD AMF·VA-API 등이며, 운영체제와 드라이버까지 맞아야 한다. 모델의 CPU 이름만 보고 “4K 몇 개”라고 단정하면 안 된다.

## 동시 시청자 수보다 먼저 보는 체크리스트

1. Jellyfin 대시보드에서 재생 중인 파일이 `Direct Play`, `Direct Stream`, `Transcode` 중 무엇인지 확인한다.
2. 클라이언트 TV와 휴대폰이 원본의 H.264·HEVC·AV1 영상, 오디오, 자막 형식을 지원하는지 확인한다.
3. 자막을 영상에 입히는 번인은 영상 트랜스코딩을 일으킬 수 있으므로 외부 자막 표시가 가능한 클라이언트를 우선한다.
4. Docker 컨테이너에 `/dev/dri` 같은 GPU 장치를 연결하고 Jellyfin에서 하드웨어 가속을 켠다. 단, 이 설정은 NAS 모델과 DSM 패키지 환경별로 달라 직접 재현하지 않은 명령을 그대로 복사하지 않는다.
5. 구매 전 목표 동시 변환 수를 “최대 파일 크기 × 동시 스트림 수”로 잡고, 업로드 회선도 함께 계산한다. Jellyfin은 원격 사용에 최소 20Mbps 업로드를 안내한다.

Jellyfin 12.0으로 올릴 때는 버전 문자열을 파싱하는 모니터링이나 컨테이너 태그도 확인한다. 공식 릴리스 안내상 기존 10.11.x와 달리 서버 버전은 12.0.0으로 보고되며, 10.11.0 업그레이드 때는 데이터·설정 디렉터리 수동 백업이 요구됐다. 버전 변경 직후에는 미디어 폴더보다 설정과 데이터베이스를 먼저 백업하고, 테스트 계정으로 재생·자막·원격 접속을 확인하는 편이 안전하다.

## 구매하지 않아도 되는 경우

원본 파일을 TV가 바로 재생하고, 가족 구성원이 같은 네트워크에서 1~2명만 사용하며, 원격 접속을 쓰지 않는다면 NAS 교체보다 클라이언트 호환성 정리가 비용이 적다. 반대로 트랜스코딩이 꼭 필요하지만 NAS CPU가 불확실하면 기존 NAS는 저장소로 두고 Intel Quick Sync가 있는 중고 미니PC를 Jellyfin 서버로 분리하는 선택도 있다. 전기료·메모리·새 HDD·백업 디스크까지 합친 금액이 새 고급 NAS보다 낮은지 비교해야 한다.

### 짧은 판단표

- Direct Play 중심: 현재 NAS 유지
- 1080p 변환 1~2개: Intel iGPU NAS 또는 별도 미니PC
- 4K HDR 원격 변환: NAS 베이 수보다 검증된 GPU 서버와 업로드 회선
- Jellyfin 12.0 업데이트: 백업 후 태그·클라이언트·자막 재생을 함께 점검

근거: [Jellyfin 12.0 릴리스](https://jellyfin.org/posts/jellyfin-release-12.0/), [Jellyfin 하드웨어 선택](https://jellyfin.org/docs/general/administration/hardware-selection/), [하드웨어 가속 문서](https://jellyfin.org/docs/general/post-install/transcoding/hardware-acceleration/), [Synology DS224+ 사양](https://www.synology.com/en-ca/products/DS224%2B), [Synology DS923+ 매뉴얼](https://kb.synology.com/en-us/HIGs/DS923p_HIG/1)

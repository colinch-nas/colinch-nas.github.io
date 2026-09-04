---
layout: post
title: "Synology ActiveProtect Manager 2.0 출시 - 일반 NAS 사용자가 도입할지 판단하는 기준"
description: "Synology ActiveProtect Manager 2.0의 2026년 9월 업데이트 내용을 정리하고, DS923+ 같은 일반 DiskStation 사용자에게 필요한 백업 구성과 도입 기준을 실제 운영 관점에서 설명한다."
date: 2026-09-05
tags: [Synology, NAS보안, 백업전략, Proxmox, HomeLab]
comments: true
share: true
---

![Synology ActiveProtect Manager 2.0 NAS 백업 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

Synology가 2026년 9월 3일 ActiveProtect Manager 2.0(APM 2.0)을 출시했다. 다만 이건 DS923+에 설치하는 DSM 패키지 업데이트가 아니다. 전용 ActiveProtect Appliance용 소프트웨어다. 일반 NAS 사용자라면 새 장비를 바로 사기보다, Proxmox VE나 여러 클라우드까지 한 화면에서 백업해야 하는지부터 판단하는 게 맞다.

## 이번 업데이트에서 달라진 점

Synology 공식 발표 기준으로 APM 2.0은 AWS EC2, Azure VM, Proxmox VE, Nutanix AHV, Google Workspace를 보호 대상으로 추가했고 플랫폼 간 복구도 지원한다. 기존처럼 “NAS 폴더를 다른 NAS에 복사”하는 수준이 아니라, 가상 머신과 SaaS 데이터까지 중앙에서 관리하는 기업용 백업에 가깝다.

| 확인 항목 | APM 2.0 | 일반 Synology NAS + Hyper Backup |
|---|---|---|
| 주 대상 | ActiveProtect DP-Series | DiskStation·RackStation |
| 백업 범위 | VM, 클라우드, Google Workspace 등 | 파일·폴더·일부 앱 데이터 |
| 복구 방식 | 플랫폼 간 복구 | 같은 NAS 또는 호환 장비로 복구 |
| 적합한 환경 | 여러 사이트·가상화 서버 | 가정·소규모 홈랩 |

여기서 가장 많이 헷갈린 부분은 “APM 2.0이 나왔으니 기존 NAS에 설치하면 되겠지”라는 가정이다. 공식 발표의 적용 대상은 ActiveProtect 데이터 보호 어플라이언스이며, 일반 DiskStation의 Active Backup 패키지를 APM 2.0으로 바꾸는 방식이 아니다.

## 일반 NAS 사용자는 어떻게 구성할까

DS923+에 Docker와 Jellyfin을 운영하고, 별도 미니PC에서 Proxmox VE를 돌리는 환경이라면 아래처럼 역할을 나누는 편이 현실적이다.

```text
[Proxmox VM] ── Active Backup for Business 또는 vzdump
       │
       ├── 1차 저장: Synology NAS
       └── 2차 저장: USB 외장 디스크 또는 다른 장소의 NAS
```

DSM에서는 `Hyper Backup → 데이터 백업 작업`을 열고 NAS의 설정, Docker Compose 파일, 사진 원본처럼 잃으면 곤란한 데이터를 별도 저장소로 보낸다. Proxmox VM은 NAS 공유 폴더에 백업 파일을 쌓되, NAS 자체가 고장 났을 때를 대비해 USB 디스크나 원격 저장소로 한 번 더 복사한다.

APM 2.0을 검토할 만한 기준은 명확하다.

- Proxmox 호스트가 2대 이상이고 VM 복구 절차를 중앙에서 관리해야 한다.
- AWS·Azure와 사내 서버의 백업 정책을 한 화면에서 통합해야 한다.
- 백업본을 복구하기 전에 악성코드 검사를 적용해야 한다.

반대로 NAS 한 대에 사진, 문서, 미디어만 저장한다면 APM보다 3-2-1 백업 구성이 먼저다. RAID는 디스크 장애 대응이지 백업이 아니며, 랜섬웨어로 암호화된 파일도 그대로 미러링할 수 있다.

## 도입 전에 확인할 것

APM 2.0은 모든 Synology NAS에 무료로 추가되는 기능이 아니라 DP-Series 어플라이언스에서 추가 비용 없이 제공되는 업데이트다. 따라서 장비 가격, 지원 플랫폼, 보관 기간, 오프사이트 저장 비용을 함께 계산해야 한다. 특히 “AI 기반 보안”이라는 표현만 보고 가정용 NAS의 보안 문제가 자동으로 해결된다고 생각하면 안 된다.

내 기준으로는 일반 사용자는 지금 쓰는 NAS에서 아래 세 가지만 점검하면 충분하다.

| 점검 | 통과 기준 |
|---|---|
| 복구 테스트 | 실제 파일 1개와 Docker 설정을 복원해 열어 봄 |
| 분리 보관 | NAS와 항상 연결된 디스크 외에 한 곳이 더 있음 |
| 계정 보호 | 관리자 계정 비활성화, 2단계 인증, 외부 공개 포트 최소화 |

APM 2.0은 홈서버용 NAS의 필수 업그레이드라기보다, 가상화·클라우드·지점 백업을 함께 운영하는 조직을 위한 선택지다. 가정용 환경에서는 제품을 추가하기 전에 현재 백업이 실제로 복구되는지부터 확인하는 편이 비용과 위험을 동시에 줄인다.

참고: [Synology ActiveProtect Manager 2.0 공식 발표](https://www.synology.com/en-global/company/news/article/apm2official), [Synology ActiveProtect Manager 2.0 소개](https://blog.synology.com/activeprotect-manager-2-0-proactive-data-protection-solution-for-modern-businesses)

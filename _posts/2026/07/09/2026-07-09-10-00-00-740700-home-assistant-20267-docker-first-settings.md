---
layout: post
title: "Home Assistant 2026.7 Docker 첫 설정 - NAS에서 재시작과 백업부터 고정하기"
description: "Home Assistant 2026.7.1을 NAS Docker에 처음 올린 뒤 compose, 포트, 백업, 외부 공개 범위를 바로 점검하는 실전 설정 순서."
date: 2026-07-09
tags: [HomeAssistant, Docker, 홈서버, 스마트홈]
comments: true
share: true
---

![NAS에서 실행하는 Home Assistant 컨테이너](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)
이 그림에서는 Home Assistant를 단독 장비가 아니라 NAS 안의 서비스 하나로 볼 때 관리해야 할 범위를 봐야 한다.

Home Assistant 2026.7.1을 NAS Docker에 처음 올렸다면 자동화부터 만들지 말고 컨테이너 재시작, `/config` 위치, 8123 포트 공개 범위부터 고정하면 된다. 2026년 7월 9일 기준 공식 2026.7 릴리스에는 새 자동화 편집기, Activity 타임라인, Update all, ZHA 관리 화면 변화가 들어갔다. 기능은 좋아졌지만 NAS에서는 서비스가 조용히 재시작되는지, 백업 파일이 어디에 남는지가 우선이다.

내가 확인한 환경은 Synology DSM 7.2.2의 Container Manager, QNAP Container Station 계열, Docker Compose 방식이다. 처음엔 `latest` 태그로 올려도 괜찮다고 생각했는데, 월간 릴리스 직후에는 원인 분리가 어려웠다. 2026.7.0에서 2026.7.1 패치로 넘어가는 것만 해도 자동화, 통합, HACS 업데이트가 같이 움직이면 어디서 깨졌는지 바로 안 보인다.

## 설치 전 기준

| 항목 | 권장값 | 이유 |
|---|---:|---|
| 이미지 태그 | `2026.7.1` | 월간 업데이트 직후 원인 추적이 쉽다 |
| 외부 포트 | 내부망 `8123`만 | 포트포워딩보다 VPN이 안전하다 |
| 설정 경로 | NAS 공유 폴더 밖의 전용 폴더 | 가족 공유 폴더 정리 중 삭제를 피한다 |
| 재시작 정책 | `unless-stopped` | NAS 재부팅 후 자동 복구된다 |

NAS에 설정 폴더를 만든 뒤 권한부터 확인한다. 여기서 소유자 권한이 꼬이면 설치는 되는데 통합 추가나 백업 생성에서 실패한다.

```bash
mkdir -p /volume1/docker/homeassistant/config
ls -ld /volume1/docker/homeassistant/config
```

Compose 파일은 짧게 시작하는 편이 낫다. USB Zigbee 동글, Bluetooth 프록시, Matter Server를 한꺼번에 붙이면 첫 장애 원인이 너무 많아진다.

```yaml
services:
  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:2026.7.1
    container_name: homeassistant
    restart: unless-stopped
    network_mode: host
    volumes:
      - /volume1/docker/homeassistant/config:/config
      - /etc/localtime:/etc/localtime:ro
```

`network_mode: host`는 Home Assistant가 같은 LAN의 스마트홈 장치를 찾기 좋다. 대신 NAS 방화벽에서 8123이 내부망에만 열렸는지 봐야 한다. 공유기에서 바로 포트포워딩하지 않는다. 외부 접속은 Tailscale VPN이나 Cloudflare Tunnel처럼 인증 경계가 있는 방식으로 나중에 붙이는 게 낫다.

## 첫 실행 뒤 체크

| 순서 | 확인 위치 | 실패할 때 보는 것 |
|---:|---|---|
| 1 | `http://NAS-IP:8123` | 포트 충돌, NAS 방화벽 |
| 2 | Settings -> System -> Backups | `/config/backups` 생성 여부 |
| 3 | Settings -> Repairs | 통합 오류, 권한 경고 |
| 4 | Activity | 자동 발견 장치와 사용자 조작 기록 |

여기서 헷갈렸던 건 백업이다. Home Assistant UI에서 만든 백업은 NAS 전체 백업이 아니다. `/config` 안의 설정 백업에 가깝다. 그래서 NAS 스냅샷이나 Hyper Backup, QNAP HBS 같은 작업에 `/volume1/docker/homeassistant`가 포함되는지 따로 확인해야 한다.

업데이트는 바로 `Update all`을 누르지 않는다. Core 컨테이너 이미지는 compose로 올리고, HACS와 ESPHome 장치 펌웨어는 하루 간격을 둔다. 실제 조명이 켜지는 자동화 1개, 센서 1개, 모바일 알림 1개만 확인해도 초기 문제의 80%는 잡힌다.

짧게 정리하면 이렇다. Home Assistant 2026.7.1 NAS Docker 설치는 compose 태그 고정, 내부망 8123, `/config` 백업 포함 여부가 핵심이다. 자동화 편집기나 ZHA 화면은 그 뒤에 만져도 늦지 않다. 출처는 [Home Assistant 2026.7 공식 릴리스 노트](https://www.home-assistant.io/blog/2026/07/01/release-20267/)와 [Home Assistant 설치 문서](https://www.home-assistant.io/installation/)다.

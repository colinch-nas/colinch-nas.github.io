---
layout: post
title: "Home Assistant 2026.7.1 NAS 설치 방법 비교 - Docker와 HAOS VM 중 무엇을 고를까"
description: "Home Assistant 2026.7.1을 NAS에 설치할 때 Docker와 HAOS 가상머신을 비교하고, Synology Container Manager에서 바로 실행하는 Compose 설정과 백업 기준을 정리한다."
date: 2026-07-15
tags: [HomeAssistant, Docker, NAS설정, 홈서버, 스마트홈]
comments: true
share: true
---

![Home Assistant 2026.7.1 NAS 설치와 스마트홈 구성](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

Home Assistant 2026.7.1을 NAS에 올린다면, 처음부터 모든 기능을 쓸 수 있는 HAOS 가상머신(VM)과 가볍게 운영하는 Docker Container 중 하나를 골라야 한다. Synology DS923+처럼 이미 Docker를 쓰는 NAS라면 Container가 빠르고, Zigbee USB 동글과 애드온까지 한 화면에서 관리하려면 HAOS VM이 덜 막힌다.

## 2026.7에서 달라진 점과 설치 선택

7월 1일 공개된 Home Assistant 2026.7은 목적에 맞는 트리거·조건을 자동화 편집기의 기본 방식으로 바꿨고, 업데이트 화면에 `Update all`을 추가했다. ZHA(Zigbee Home Automation) 장치 관리 화면도 손봤다. 기존 자동화는 유지되지만, 운영 방식에 따라 업데이트 전 백업을 남기는 습관이 필요하다.

| 설치 방식 | 맞는 환경 | 아쉬운 점 |
|---|---|---|
| Docker Container | NAS에 이미 Container Manager가 있고 CPU·메모리를 아끼고 싶을 때 | 애드온 없음, 백업·업데이트를 직접 관리 |
| HAOS VM | Home Assistant 전용 서버처럼 Supervisor와 애드온을 쓰고 싶을 때 | RAM 4GB 이상 권장, USB 동글 연결이 한 단계 더 필요 |
| 별도 Raspberry Pi | NAS 장애와 스마트홈 장애를 분리하고 싶을 때 | 장치 하나와 저장장치 관리가 늘어남 |

내 환경은 Synology DSM 7.2.2, RAM 10GB, Docker Container Manager를 쓰는 구성이다. NAS에 Jellyfin과 Uptime Kuma도 있어 HAOS VM 대신 Container로 설치했다. 다만 Zigbee 동글을 NAS에 꽂아 여러 컨테이너에서 번갈아 쓸 계획이라면 처음부터 VM이나 별도 라즈베리파이를 선택하는 편이 낫다.

## Synology NAS에 Docker로 설치하기

먼저 `/volume1/docker/homeassistant/config` 폴더를 만들고, Container Manager의 프로젝트에서 아래 Compose를 실행한다. `network_mode: host`는 Home Assistant가 mDNS와 장치 검색을 내부망에서 제대로 받게 하는 설정이다.

```yaml
services:
  homeassistant:
    container_name: homeassistant
    image: ghcr.io/home-assistant/home-assistant:2026.7.1
    restart: unless-stopped
    network_mode: host
    environment:
      - TZ=Asia/Seoul
    volumes:
      - /volume1/docker/homeassistant/config:/config
```

프로젝트를 배포한 뒤 `http://NAS_IP:8123`으로 접속한다. 첫 화면에서 계정을 만들고 위치와 시간대를 서울로 지정한다. 포트를 따로 매핑하지 않았기 때문에 Compose의 `ports` 항목을 추가하지 않는 것이 포인트다. 이미 8123 포트를 쓰는 서비스가 있으면 컨테이너가 시작되지 않으므로 그때는 기존 서비스를 먼저 확인한다.

## Zigbee 동글을 붙일 때 생기는 함정

Zigbee2MQTT를 같이 쓸 경우 동글의 `/dev/ttyUSB0` 또는 `/dev/ttyACM0` 경로가 재부팅 뒤 바뀔 수 있다. Container Manager에서 장치 경로를 확인하고, 가능하면 `/dev/serial/by-id/` 아래의 고정 경로를 매핑한다. 같은 동글을 ZHA와 Zigbee2MQTT에 동시에 연결하면 통신이 꼬이므로 둘 중 하나만 선택해야 한다.

| 확인 항목 | 설치 직후 기준 |
|---|---|
| 접속 | `NAS_IP:8123` 로컬 접속 |
| 시간대 | `Asia/Seoul` |
| 데이터 경로 | `/volume1/docker/homeassistant/config` |
| 외부 접속 | 포트 직접 공개보다 Tailscale 권장 |
| 업데이트 | 백업 후 이미지 태그를 한 버전씩 변경 |

## 업데이트와 백업 기준

2026.7.1 이미지는 글 작성일 기준 패치 버전이다. `latest` 태그를 쓰면 재배포 시 버전이 갑자기 바뀌므로 위처럼 버전을 고정했다. 업데이트 전에는 Home Assistant의 설정 → 시스템 → 백업에서 전체 백업을 만들고, NAS 스냅샷이나 다른 저장소에도 한 부를 복사한다. Docker Container는 HAOS의 자동 백업 흐름과 다르기 때문에 `/config` 폴더만 믿으면 안 된다.

```bash
# 컨테이너 설정 폴더를 별도 백업 위치로 복사한다.
tar -czf /volume1/backup/homeassistant-config-2026-07-15.tar.gz \
  -C /volume1/docker/homeassistant config
```

스마트홈의 핵심은 홈 자동화보다 복구다. NAS를 재설치해도 `/config`와 백업 파일, Zigbee 동글의 연결 정보를 함께 복원할 수 있어야 한다. 가벼운 운영은 Docker, 애드온과 장치 관리는 HAOS VM으로 나누면 선택 기준이 명확해진다.

- NAS에 이미 Docker가 있으면 Home Assistant Container부터 시작한다.
- Supervisor·애드온·Zigbee를 한 화면에서 관리하려면 HAOS VM을 고른다.
- 2026.7.1처럼 버전을 고정하고 업데이트 전 백업한다.

참고: [Home Assistant 2026.7 릴리스 노트](https://www.home-assistant.io/blog/2026/07/01/release-20267/), [Home Assistant Docker 설치 문서](https://www.home-assistant.io/installation/linux#install-home-assistant-container)

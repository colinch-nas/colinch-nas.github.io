---
layout: post
title: "Home Assistant 설치 방법 비교 2026 — Raspberry Pi vs Synology NAS vs VM"
description: "Home Assistant를 설치하는 세 가지 방법(Raspberry Pi, Synology NAS Docker, Proxmox VM)의 장단점을 2026년 기준으로 비교한다. 어떤 방법이 자신에게 맞는지 판단하는 기준 포함."
date: 2026-06-10
tags: [HomeAssistant, 스마트홈, 홈오토메이션, Proxmox]
comments: true
share: true
---

# Home Assistant 설치 방법 비교 2026 — Raspberry Pi vs Synology NAS vs VM

![Home Assistant 스마트홈 설정](https://images.unsplash.com/photo-1558618047-3c8c76ca7d67?w=800&q=80)

Home Assistant는 스마트홈 허브다. 삼성 SmartThings, 애플 HomeKit, 구글 홈처럼 클라우드에 종속되지 않고 집 안에서 로컬로 모든 기기를 통합 관리한다. 설치 방법은 여러 가지인데, 처음 접하면 어떤 걸 써야 할지 헷갈린다.

## 세 가지 설치 방법 요약

| 방법 | 난이도 | 기능 | 비용 | 전력 소비 |
|------|-------|------|------|---------|
| Raspberry Pi 5 (HAOS) | 낮음 | 풀 지원 | ~10만원 | 5~15W |
| Synology NAS Docker | 중간 | 제한적 | NAS 있으면 추가 없음 | NAS 전력 포함 |
| Proxmox VM (HAOS) | 높음 | 풀 지원 | 미니PC ~20만원~ | 15~30W |

## Raspberry Pi 5 + HAOS (추천 입문)

처음 Home Assistant 시작한다면 이게 정석이다. [Raspberry Pi 5](https://www.raspberrypi.com/products/raspberry-pi-5/)에 Home Assistant OS(HAOS)를 설치하는 방법이다.

HAOS는 Home Assistant 전용 운영체제라서 애드온 시스템이 완전히 지원되고, 자동 업데이트도 된다. 셋업 과정이 가장 단순하다.

```bash
# Raspberry Pi Imager로 HAOS 이미지 굽기
# 1. Raspberry Pi Imager 다운로드
# 2. Operating System → Other specific-purpose OS → Home Assistants
# 3. Home Assistant OS 선택 후 SD카드 또는 SSD에 기록
```

SD카드보다 USB SSD에 설치하는 걸 권장한다. SD카드는 Home Assistant처럼 자주 쓰기 작업이 있는 환경에서 빠르게 수명이 닳는다.

**단점**: Raspberry Pi 5 품귀 현상이 2026년에도 여전히 간헐적으로 있다. 구하면 빠르게 사야 한다.

## Synology NAS Docker (편의성)

NAS가 이미 있다면 추가 비용 없이 Docker 컨테이너로 Home Assistant를 올릴 수 있다.

```yaml
# docker-compose.yml
version: "3.9"
services:
  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:stable
    container_name: homeassistant
    network_mode: host
    environment:
      - TZ=Asia/Seoul
    volumes:
      - /volume1/docker/homeassistant/config:/config
    privileged: true
    restart: unless-stopped
```

```bash
sudo docker compose up -d
# 접속: http://NAS-IP:8123
```

![Home Assistant 설치 완료 화면](https://images.unsplash.com/photo-1451187580094-2cfb76b7c9b7?w=800&q=80)

**결정적인 단점**: Docker 버전의 Home Assistant는 **Home Assistant Core**다. Zigbee 동글처럼 USB 장치를 컨테이너에 패스스루하는 게 까다롭고, 감독자(Supervisor) 기반 애드온 시스템이 지원되지 않는다. Zigbee2MQTT, Mosquitto MQTT 브로커 같은 걸 모두 별도 컨테이너로 직접 구성해야 한다.

편하게 쓰고 싶은 사람한테는 권장하지 않는다. 홈랩 좋아하는 사람한테는 오히려 이 방법이 재밌다.

## Proxmox VE + HAOS VM (풀 기능 + 유연성)

기존 PC나 미니PC를 Proxmox로 돌려서 그 안에 HAOS를 VM으로 설치하는 방법이다. HAOS의 풀 기능을 쓰면서 같은 서버에서 다른 VM도 돌릴 수 있다.

```bash
# Proxmox 웹 UI에서 VM 생성
# HAOS 이미지 다운로드 (qcow2 형식)
wget https://github.com/home-assistant/operating-system/releases/latest/download/haos_ova-stable.qcow2.xz
xz -d haos_ova-stable.qcow2.xz

# Proxmox에 이미지 임포트
qm importdisk 100 haos_ova-stable.qcow2 local-lvm
```

USB Zigbee 동글을 VM에 패스스루하는 것도 가능하고, 스냅샷 기능을 활용해서 업데이트 전 롤백 포인트를 만들 수도 있다.

**단점**: 처음 설치 과정이 복잡하다. Proxmox 자체를 알아야 한다. 이 시리즈 나중에 [Proxmox 설치 편]({% post_url 2026-06-19-10-00-00-185175-proxmox-ve8-minipc-virtualization-server %})에서 자세히 다룬다.

## 어떤 방법을 골라야 하는가

- **처음 스마트홈 시작**: Raspberry Pi 5 + HAOS
- **이미 Synology NAS가 있고 간단히 시작하고 싶다**: NAS Docker (기능 제한 감수)
- **미니PC나 구형 PC가 있고, 다른 서비스도 함께 운영하고 싶다**: Proxmox + HAOS VM

## 삽질했던 부분

NAS Docker로 처음 Home Assistant를 설치했다가 Zigbee 동글을 연결하려다 막혀서 결국 Proxmox로 이사했다. 처음부터 자신이 무엇을 하고 싶은지 알고 방법을 고르는 게 중요하다.

Home Assistant 첫 설정 후 온보딩 화면에서 위치 정보를 설정해야 일출/일몰 기반 자동화가 된다. 위치 정보를 건너뛰면 나중에 **설정 → 시스템 → 일반**에서 다시 설정할 수 있다.

## 한 줄 정리

Raspberry Pi 5 + HAOS가 가장 쉽고 기능이 완전하다. NAS가 있어도 진지하게 쓰려면 HAOS가 돌아가는 전용 기기를 쓰는 게 낫다.

---
다음엔 [Zigbee2MQTT 설정 완전 가이드 — 별도 허브 없이 Zigbee 기기 연결하는 법]({% post_url 2026-06-11-10-00-00-086415-zigbee2mqtt-setup-no-hub %})을 다룬다.

**참고 링크**
- [Home Assistant 공식 설치 가이드](https://www.home-assistant.io/installation/)
- [Raspberry Pi 5 공식 사이트](https://www.raspberrypi.com/products/raspberry-pi-5/)
- [Proxmox VE 공식 사이트](https://www.proxmox.com/en/proxmox-virtual-environment/overview)

---
layout: post
title: "Home Assistant 2026.7.1 NAS 업데이트 - Docker 백업과 복구 점검 순서"
description: "Synology NAS Docker에서 Home Assistant 2026.7.1을 올리기 전 백업, 컨테이너 업데이트 순서, 복구 확인까지 실제 운영 기준으로 정리한다."
date: 2026-07-16
tags: [HomeAssistant, Docker, Synology, NAS보안, 자체호스팅]
comments: true
share: true
---

![Synology NAS에서 Home Assistant Docker 업데이트 전 백업 점검](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

Home Assistant 2026.7.1을 NAS Docker에 적용할 때는 이미지부터 바꾸면 안 된다. 백업 파일이 실제로 만들어지는지 확인하고, Mosquitto와 Zigbee2MQTT를 건드리지 않은 상태에서 Home Assistant만 교체하는 순서가 가장 덜 위험하다. 2026.7에는 자동화 편집기와 Activity 화면 변화가 들어갔고, 백업 압축 해제 처리도 보강됐다.

## 내가 쓰는 환경

| 항목 | 기준 |
|---|---|
| NAS | Synology DS923+ |
| 관리 화면 | DSM 7.2.2 Container Manager |
| Home Assistant | 2026.7.1, Docker Compose |
| 데이터 경로 | `/volume1/docker/homeassistant` |
| 연결 서비스 | Mosquitto, Zigbee2MQTT |

Home Assistant의 릴리스와 컨테이너 이미지 태그는 항상 일치하지 않을 수 있다. `latest`를 무작정 당기기보다 현재 버전을 기록한 뒤 업데이트하는 편이 장애 원인을 찾기 쉽다.

## 1. 업데이트 전 백업 만들기

업데이트 전에 Home Assistant 화면에서 **설정 → 시스템 → 백업**으로 들어가 전체 백업을 만든다. 백업이 끝났다는 알림만 믿지 말고 목록에서 날짜와 용량을 확인한다. 용량이 평소 0KB에 가깝다면 저장 경로 또는 권한 문제일 가능성이 높다.

NAS 폴더도 별도로 복사한다. 이 폴더에는 `configuration.yaml`, 자동화, 애드온이 아닌 Docker 환경 설정과 데이터베이스가 함께 들어 있다.

```bash
cd /volume1/docker
tar -czf homeassistant-before-2026.7.1-$(date +%F).tar.gz homeassistant
ls -lh homeassistant-before-2026.7.1-*.tar.gz
```

압축 파일 목록까지 확인해야 한다. 백업 파일이 존재해도 경로를 잘못 묶으면 복구할 때 빈 폴더만 나오는 경우가 있다.

```bash
tar -tzf homeassistant-before-2026.7.1-2026-07-16.tar.gz | head
```

## 2. Home Assistant만 순서대로 교체하기

Compose 파일에서 이미지 태그를 고정해 둔 경우 `2026.7.1`로 바꾼다. `latest`를 쓰고 있다면 이번 작업 전에 태그를 고정하는 것이 좋다.

```yaml
services:
  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:2026.7.1
    container_name: homeassistant
    volumes:
      - /volume1/docker/homeassistant:/config
    restart: unless-stopped
    network_mode: host
```

이제 Home Assistant 컨테이너만 다시 만든다. 명령어를 실행하는 동안 Zigbee 동글을 점유한 Zigbee2MQTT는 재시작하지 않는다.

```bash
docker compose pull homeassistant
docker compose up -d homeassistant
docker logs -f --tail=100 homeassistant
```

로그에서 `Error loading`이나 `Invalid config`가 반복되지 않는지 보고, 웹 화면의 **설정 → 시스템 → 정보**에서 2026.7.1이 표시되는지 확인한다. 초기 기동은 데이터베이스 크기에 따라 평소보다 오래 걸릴 수 있다.

## 3. 자동화와 Zigbee를 따로 검증하기

이번 버전은 목적별 트리거와 조건을 보여주는 자동화 편집 흐름이 달라졌다. 기존 YAML 자동화가 사라진 것은 아니지만, UI에서 다시 저장하면 표현이 바뀌는지 테스트 자동화로 확인하는 게 안전하다.

| 점검 | 확인 방법 | 실패 시 조치 |
|---|---|---|
| 자동화 | 개발자 도구에서 수동 실행 | YAML 문법과 엔티티 ID 확인 |
| Zigbee 기기 | 센서 상태 1개 확인 | Zigbee2MQTT 로그는 나중에 확인 |
| MQTT | 브로커 연결 상태 확인 | Mosquitto 컨테이너 재시작 금지 |
| 백업 | 최신 백업 날짜·용량 확인 | 다른 NAS나 PC에 복사 |

특히 USB 동글 경로를 `/dev/ttyUSB0`로 잡았다면 재부팅 뒤 바뀔 수 있다. Zigbee2MQTT의 동글 경로는 `/dev/serial/by-id/` 아래의 고정 경로를 쓰고, 같은 동글을 ZHA와 동시에 열지 않는다.

## 업데이트를 되돌려야 할 때

자동화가 연속으로 실패하거나 Home Assistant가 재기동을 반복하면 컨테이너를 계속 재시작하지 않는다. Compose 이미지 태그를 이전 버전으로 되돌리고, 백업한 `/config` 폴더를 별도 위치에 보존한 뒤 로그를 남긴다. 데이터베이스를 덮어쓰기 전에 복사본을 만드는 것이 핵심이다.

짧게 정리하면 `전체 백업 → NAS 폴더 압축 → 이미지 태그 고정 → Home Assistant만 교체 → 자동화·MQTT·Zigbee 확인` 순서다. 월간 릴리스에서는 Core, MQTT 브로커, Zigbee2MQTT를 한꺼번에 올리지 않아야 어떤 변경이 문제를 만들었는지 추적할 수 있다.

참고 문서: [Home Assistant 2026.7 릴리스 노트](https://www.home-assistant.io/blog/2026/07/01/release-20267/), [2026.7 전체 변경 내역](https://www.home-assistant.io/changelogs/core-2026.7)

---
layout: post
title: "Home Assistant 2026.7 설치 방법 비교 - 시놀로지 NAS Docker로 스마트홈 구축"
description: "Home Assistant 2026.7.3 기준 설치 방식 차이를 비교하고, Synology DSM 7.4 Container Manager에서 Docker Compose로 스마트홈 서버를 구성하는 방법을 정리한다."
date: 2026-08-14
tags: [HomeAssistant, Synology, Docker, 스마트홈, NAS설정, Zigbee]
comments: true
share: true
---

![Synology NAS에서 Home Assistant 스마트홈 서버를 구성하는 모습](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

Home Assistant 2026.7.3 기준으로 시놀로지 NAS에는 Home Assistant Container를 설치하는 선택이 가장 현실적이다. 이미 DSM 7.4에서 Container Manager를 쓰고 있다면 별도 미니PC 없이 10분 안에 시작할 수 있다. 다만 OS 버전처럼 앱(Add-on)을 UI에서 설치하는 기능은 빠지므로, Zigbee2MQTT나 Mosquitto도 Docker 컨테이너로 따로 관리해야 한다.

## 설치 방식부터 고르기

Home Assistant 공식 문서에서 현재 운영 대상으로 보는 방식은 Home Assistant OS와 Container다. 예전에 보이던 Core와 Supervised 설치는 새로 시작할 방법으로 잡지 않는 편이 맞다.

| 방식 | 잘 맞는 환경 | 장점 | NAS에서 걸리는 부분 |
|---|---|---|---|
| Home Assistant OS | 라즈베리 파이, 미니PC, 전용 VM | Supervisor와 앱 관리가 편함 | NAS에서는 VM 자원과 USB 패스스루가 필요함 |
| Home Assistant Container | Synology·QNAP Docker 사용자 | 기존 백업·볼륨·프록시 체계 재사용 | 앱, Zigbee2MQTT를 직접 구성해야 함 |

내 테스트 환경은 Synology DS923+, DSM 7.4, Container Manager, `/volume1/docker/homeassistant` 경로다. NAS에서 파일 서버와 스마트홈을 함께 운영한다면 Container가 관리 방식에 잘 맞는다. 반대로 Home Assistant를 처음 접하고 부가 서비스를 많이 쓸 생각이면 미니PC에 Home Assistant OS를 설치하는 편이 덜 번거롭다.

## Synology Container Manager에 설치하기

File Station에서 `/volume1/docker/homeassistant` 폴더를 만들고, 그 안에 아래 Compose 파일을 저장한다. `Europe/Seoul`을 지정하지 않으면 자동화 시간이 UTC로 어긋나는 삽질을 하게 된다.

```yaml
services:
  homeassistant:
    container_name: homeassistant
    image: ghcr.io/home-assistant/home-assistant:2026.7.3
    restart: unless-stopped
    privileged: true
    network_mode: host
    environment:
      TZ: Asia/Seoul
    volumes:
      - /volume1/docker/homeassistant/config:/config
      - /etc/localtime:/etc/localtime:ro
      - /run/dbus:/run/dbus:ro
```

Container Manager에서 프로젝트 → 생성 → 경로를 `/volume1/docker/homeassistant`로 선택하고 실행한다. `network_mode: host`를 쓰면 Home Assistant가 `8123` 포트를 직접 열고, 같은 네트워크의 기기를 자동 발견하기 쉽다. 실행 후 브라우저에서 `http://NAS_IP:8123`으로 접속해 소유자 계정을 만든다.

Zigbee USB 동글을 NAS에 꽂았다면 장치 경로를 확인한다.

```bash
ls -l /dev/ttyUSB* /dev/ttyACM*
```

Compose의 서비스 아래에 실제 경로를 추가한다. 장치 이름은 NAS마다 달라서 `/dev/ttyUSB0`를 그대로 복사하면 안 된다.

```yaml
    devices:
      - /dev/ttyUSB0:/dev/ttyUSB0
```

ZHA를 쓸 때는 Home Assistant에서 해당 포트를 선택한다. Zigbee2MQTT를 별도 컨테이너로 운영할 경우에는 동글을 두 컨테이너에 동시에 연결하지 않는다. 이 상태가 되면 한쪽이 포트를 선점해 “adapter not found” 오류가 반복된다.

## 업데이트와 백업에서 실수한 부분

`stable` 태그만 쓰면 업데이트 시점을 통제하기 어렵다. 운영 중인 NAS에서는 버전을 고정하고, 백업 후 이미지를 바꾸는 순서가 안전하다.

```bash
cd /volume1/docker/homeassistant
docker compose pull
docker compose up -d
docker logs --tail=100 homeassistant
```

업데이트 전 Home Assistant 설정 화면에서 전체 백업을 만들고, `/volume1/docker/homeassistant/config` 폴더도 Hyper Backup 대상에 넣는다. Container 방식은 Supervisor 백업 버튼에 기대지 않으므로 Compose 파일과 `config` 폴더가 실제 복구 재료다.

외부 접속은 `8123` 포트를 공유기에 직접 포워딩하지 않는다. 기존 글에서 설정한 역방향 프록시를 사용하고, Home Assistant의 `trusted_proxies` 설정과 HTTPS를 함께 적용한다. 스마트홈 서버는 조명보다 계정과 토큰이 더 민감한 서비스라 관리자 계정에 2단계 인증도 켜는 게 좋다.

## 짧게 정리

- 전용 장비와 앱 관리가 우선이면 Home Assistant OS
- Synology Docker 환경을 재사용하려면 Home Assistant Container
- DSM 7.4에서는 `network_mode: host`와 고정 버전 이미지가 출발점
- Zigbee 동글은 실제 `/dev/tty*` 경로를 확인하고 한 컨테이너만 사용
- 업데이트 전 `config` 폴더와 Compose 파일을 함께 백업

참고한 공식 문서는 [Home Assistant 설치 방식](https://www.home-assistant.io/installation/), [Linux·Container 설치](https://www.home-assistant.io/installation/linux/), [Container 업데이트](https://www.home-assistant.io/common-tasks/container/), [2026.7 릴리스 노트](https://www.home-assistant.io/blog/2026/07/01/release-20267/)다.

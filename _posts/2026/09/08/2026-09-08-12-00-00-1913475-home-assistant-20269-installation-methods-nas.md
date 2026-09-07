---
layout: post
title: "Home Assistant 2026.9 설치 방법 비교 - Synology NAS에서는 Docker가 맞을까"
description: "Home Assistant 2026.9를 Synology NAS에 설치할 때 Home Assistant OS, Docker Container, 가상머신을 비교하고 실제 Docker Compose 설정과 선택 기준을 정리한다."
date: 2026-09-08
tags: [HomeAssistant, Synology, Docker, 홈서버, 스마트홈]
comments: true
share: true
---

![Home Assistant 2026.9 Synology NAS 설치](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=1200&q=80)

Home Assistant 2026.9를 Synology NAS에 올린다면 Docker Container가 가장 빨리 시작할 수 있는 방법이다. 다만 Home Assistant 공식 권장 설치 방식은 Home Assistant OS다. NAS에 이미 Docker와 백업 체계를 만들어뒀다면 Container, Zigbee·Thread 장치와 애드온을 한 화면에서 관리하고 싶다면 별도 미니PC의 HA OS가 덜 번거롭다.

## 2026.9에서 설치 방식을 다시 고른 이유

2026.9는 Modbus 장치 연결 구조가 정리된 릴리스다. 예전에 보던 Home Assistant Core와 Supervised 설치 가이드는 이제 선택지로 보면 안 되고, 현재 공식 문서가 안내하는 방식은 HA OS와 Container다.

| 방식 | 잘 맞는 경우 | 걸리는 지점 |
|---|---|---|
| Home Assistant OS | 전용 미니PC·Raspberry Pi·가상머신 | NAS 앱과 분리된 장비가 필요 |
| Home Assistant Container | Synology Docker를 이미 운영 중 | 앱(애드온)과 일부 Thread·Z-Wave 관리 기능 없음 |
| HA OS 가상머신 | NAS 자원을 나눠 전용 환경을 만들 때 | USB 코디네이터 패스스루와 메모리 관리 필요 |

NAS에서 Docker를 이미 운영 중이면 Container, 스마트홈이 처음이고 애드온을 많이 쓸 생각이면 별도 장비의 HA OS가 덜 번거롭다.

## Synology NAS에 Home Assistant Container 설치

아래 예시는 DSM 7.4의 Container Manager에서 실행하는 Docker Compose 구성이다. `/volume1/docker/homeassistant` 폴더를 먼저 만들고, Container Manager의 **프로젝트 → 생성**에서 Compose 파일을 붙여 넣는다.

```yaml
services:
  homeassistant:
    container_name: homeassistant
    image: ghcr.io/home-assistant/home-assistant:2026.9
    restart: unless-stopped
    network_mode: host
    privileged: true
    environment:
      TZ: Asia/Seoul
    volumes:
      - /volume1/docker/homeassistant/config:/config
      - /run/dbus:/run/dbus:ro
```

`network_mode: host`는 기기 검색에 사용하는 브로드캐스트와 mDNS(같은 네트워크의 장치를 자동 발견하는 방식)를 놓치지 않게 하려고 넣었다. 포트 매핑 없이 NAS의 8123 포트를 그대로 쓰므로 실행 후 `http://NAS_IP:8123`으로 접속한다.

처음 실행하기 전 **보안 → 방화벽**에서 내부망 관리용 PC의 8123 포트를 허용한다. 인터넷 전체에 8123을 포워딩하지 말고, 외부 접속은 Tailscale VPN이나 HTTPS 역방향 프록시를 사용한다.

## Zigbee 코디네이터를 연결할 때 막히는 부분

Container 설치의 실제 난관은 YAML보다 USB다. **제어판 → 정보 센터 → 장치 → USB 장치**에서 인식 여부를 확인한 뒤, Container Manager 프로젝트 설정의 장치(Device)에 경로를 연결한다.

```yaml
    devices:
      - /dev/ttyUSB0:/dev/ttyUSB0
```

재부팅 뒤 `/dev/ttyUSB0`가 `/dev/ttyUSB1`로 바뀌면 Zigbee2MQTT가 코디네이터를 못 찾는다. 가능하면 `/dev/serial/by-id/` 아래의 고정 경로를 사용하고, USB 3.0 간섭을 피하려고 연장 케이블을 쓰는 편이 낫다.

이 방식에는 HA OS의 앱(애드온) 메뉴가 없다. Mosquitto MQTT, Zigbee2MQTT, ESPHome은 별도 컨테이너로 운영하고 업데이트·백업도 직접 챙겨야 한다.

## 설치 직후 확인할 체크리스트

- `http://NAS_IP:8123`에서 온보딩 화면이 열리고 Core가 `2026.9.x`인가
- 설정 → 시스템 → 백업에서 첫 전체 백업을 만들었는가
- Zigbee 동글이 설정 → 시스템 → 하드웨어에 보이는가
- 공유기에서 8123 포트가 외부로 열려 있지 않고 config 폴더가 백업 대상인가

| 상황 | 추천 선택 |
|---|---|
| Synology NAS를 이미 24시간 켜두고 Docker에 익숙함 | Home Assistant Container |
| Zigbee·Thread·애드온을 한 화면에서 관리하고 싶음 | 별도 장비의 Home Assistant OS |
| NAS 한 대로 끝내고 USB 패스스루도 감당 가능함 | HA OS 가상머신 |

핵심은 업데이트와 주변 장치를 누가 관리하느냐다. 처음 시작한다면 HA OS, DSM 7.4에서 Docker를 관리 중이라면 위 Container 구성이 맞다.

참고 문서:

- [Home Assistant 2026.9 릴리스 노트](https://www.home-assistant.io/blog/2026/09/02/release-20269/)
- [Home Assistant 설치 방식 공식 문서](https://www.home-assistant.io/installation/)
- [Home Assistant OS와 Container 비교](https://www.home-assistant.io/faq/ha-vs-hassio/)

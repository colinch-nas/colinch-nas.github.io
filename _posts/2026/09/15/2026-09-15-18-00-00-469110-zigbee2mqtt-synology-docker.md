---
layout: post
title: "Zigbee2MQTT 2.x 설치 - Synology Docker와 Home Assistant 2026.9 연결"
description: "Synology DS923+ DSM 7.2.2에서 Zigbee2MQTT 2.x와 Mosquitto를 Docker로 설치하고 Home Assistant 2026.9.2에 센서를 자동 등록하는 설정을 정리한다."
date: 2026-09-15
tags: [Synology, Docker, HomeAssistant, Zigbee2MQTT, 스마트홈]
comments: true
share: true
---

![Synology NAS와 Zigbee 스마트홈 센서 구성](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

그림에서 볼 부분은 USB Zigbee 코디네이터가 NAS의 Zigbee2MQTT 컨테이너에 연결되고, 센서 상태는 Mosquitto를 거쳐 Home Assistant로 들어가는 흐름이다.

Synology NAS에 Home Assistant를 이미 올려뒀다면 Zigbee2MQTT를 붙이는 편이 확장성이 좋다. Home Assistant 2026.9.2에서도 MQTT Discovery(기기가 MQTT 메시지로 자신을 자동 등록하는 방식)를 그대로 사용할 수 있고, Zigbee2MQTT 공식 문서도 Docker 설치를 권장한다. 여기서는 Synology DS923+, DSM 7.2.2, Container Manager, Home Assistant 2026.9.2, Mosquitto 2.x, Zigbee2MQTT 2.x를 기준으로 잡았다.

## 준비할 것

| 항목 | 예시 | 확인할 점 |
|---|---|---|
| NAS | Synology DS923+ | Container Manager 설치 |
| 코디네이터 | Home Assistant Connect ZBT-1 또는 Sonoff ZBDongle-P | USB 연장 케이블 권장 |
| MQTT | Mosquitto 2.x | NAS 내부 IP와 계정 준비 |
| 폴더 | `/volume1/docker/zigbee2mqtt` | 컨테이너 재생성에도 설정 보존 |

Zigbee 동글을 NAS에 바로 꽂으면 Wi-Fi 공유기나 USB 3.0 간섭 때문에 통신이 불안정할 수 있다. 50cm 정도 USB 연장 케이블을 준비하면 시행착오를 줄일 수 있다.

## Docker Compose 작성

Container Manager에서 프로젝트를 만들고, 아래 Compose를 붙여 넣는다. `ttyUSB0`는 NAS에서 실제로 확인한 포트로 바꿔야 한다.

```yaml
services:
  zigbee2mqtt:
    image: koenkk/zigbee2mqtt:latest
    container_name: zigbee2mqtt
    restart: unless-stopped
    ports:
      - "8080:8080"
    volumes:
      - /volume1/docker/zigbee2mqtt/data:/app/data
    devices:
      - /dev/ttyUSB0:/dev/ttyUSB0
    environment:
      - TZ=Asia/Seoul
    networks:
      - smarthome

networks:
  smarthome:
    external: true
```

`smarthome` 네트워크가 없다면 프로젝트를 시작하기 전에 SSH에서 한 번만 만든다.

```bash
sudo docker network create smarthome
```

동글 포트를 모르면 SSH로 아래 명령을 실행한다. `/dev/ttyACM0`가 나오면 Compose의 양쪽 경로를 모두 `ttyACM0`로 맞춘다.

```bash
ls -l /dev/serial/by-id
ls -l /dev/ttyUSB* /dev/ttyACM* 2>/dev/null
```

처음 실행하면 `http://NAS_IP:8080`에서 온보딩 화면이 열린다. MQTT 브로커 주소는 같은 Docker 네트워크의 서비스명인 `mosquitto`를 쓰고, 이미 별도 프로젝트라면 `192.168.1.20` 같은 NAS 내부 IP를 쓴다. 공용 MQTT 브로커는 사용하지 않는다.

## Zigbee2MQTT와 Home Assistant 연결

`/volume1/docker/zigbee2mqtt/data/configuration.yaml`에 다음처럼 입력한다. 비밀번호는 예시 값을 그대로 쓰지 말고 Mosquitto 계정으로 교체한다.

```yaml
frontend:
  enabled: true
  port: 8080

mqtt:
  base_topic: zigbee2mqtt
  server: mqtt://mosquitto:1883
  user: zigbee
  password: 여기에_MQTT_비밀번호

serial:
  port: /dev/ttyUSB0
  adapter: ember

homeassistant:
  enabled: true
  discovery_topic: homeassistant
  status_topic: homeassistant/status
```

`adapter` 값은 동글 칩셋에 따라 다르다. ZBT-1처럼 Ember 계열이면 `ember`, TI Z-Stack 계열이면 `zstack`을 쓴다. 틀리면 코디네이터 초기화에서 멈춘다. `USB adapter discovery error`가 나오면 포트와 adapter부터 확인한다.

Home Assistant에서 `설정 → 기기 및 서비스 → 통합 추가 → MQTT`를 선택하고 브로커 주소·계정·비밀번호를 입력한다. MQTT Discovery가 켜진 상태에서 Zigbee2MQTT를 재시작하면 대시보드에 Zigbee2MQTT 브리지가 나타난다. 그 뒤 Zigbee2MQTT 화면에서 `Permit join`을 켜고 센서 페어링 버튼을 누르면 기기와 엔티티가 자동 등록된다.

흐름은 다음과 같다.

```text
Zigbee 센서 → USB 코디네이터 → Zigbee2MQTT → Mosquitto → Home Assistant
```

## 실제로 막히는 지점

- `/dev/ttyUSB0`가 재부팅 후 바뀌면 컨테이너가 동글을 못 찾는다. 가능하면 `/dev/serial/by-id/...`의 고정 경로를 사용한다.
- Zigbee 채널과 Wi-Fi 채널이 겹치면 배터리 센서가 간헐적으로 끊긴다. 처음에는 Zigbee 채널 15, 20, 25 중 현재 Wi-Fi와 덜 겹치는 값을 선택한다.
- 페어링이 끝나면 `Permit join`을 바로 끈다. 계속 열어두면 원치 않는 기기가 네트워크에 들어올 수 있다.
- `latest` 태그는 편하지만 자동 업데이트 때 설정이 깨질 수 있다. 안정화 후에는 실제 이미지 버전을 고정하고 `data` 폴더와 Mosquitto 데이터를 함께 백업한다.

Home Assistant 애드온보다 Docker 프로젝트로 분리하면 Synology에서 로그와 데이터 위치를 확인하기 쉽다. USB 동글을 다른 서버로 옮길 계획이면 Ethernet 코디네이터가 낫다.

핵심은 세 가지다. MQTT는 NAS 내부에서만 열고, 코디네이터 포트는 고정하며, Home Assistant Discovery는 켜되 조인은 필요할 때만 허용한다.

참고 문서: [Zigbee2MQTT Docker 설치 문서](https://www.zigbee2mqtt.io/guide/installation/01_linux.html), [Zigbee2MQTT 어댑터 설정](https://www.zigbee2mqtt.io/guide/configuration/adapter-settings.html), [Zigbee2MQTT Home Assistant 연동](https://www.zigbee2mqtt.io/guide/configuration/homeassistant.html), [Home Assistant MQTT 통합](https://www.home-assistant.io/integrations/mqtt/)

---
layout: post
title: "Zigbee Thread 분리 운영 - NAS Home Assistant에서 Hue 업데이트에 흔들리지 않는 구성"
description: "2026년 Hue Thread·Zigbee 동시 지원 흐름에 맞춰 NAS Docker Home Assistant에서 Zigbee2MQTT와 Matter/Thread를 분리 운영하는 설정법."
date: 2026-07-01
tags: [HomeAssistant, Zigbee2MQTT, Matter, 스마트홈]
comments: true
share: true
---
![Zigbee와 Thread를 분리해 운영하는 NAS 스마트홈 구성](https://images.unsplash.com/photo-1558002038-1055907df827?w=800&q=80)

2026년 7월 기준으로 Zigbee 기기를 이미 안정적으로 쓰고 있다면 Matter/Thread로 한 번에 옮길 이유는 없다. NAS Docker 환경에서는 Zigbee2MQTT는 Zigbee 전용으로 두고, Matter Server와 Thread 보더 라우터는 별도 경로로 관리하는 편이 덜 흔들린다.

Philips Hue는 2026년 후반 일부 Matter-over-Thread 지원 조명에서 Zigbee와 Thread를 동시에 쓰는 업데이트를 예고했다. 방향은 좋다. 다만 이 소식만 보고 기존 Hue Bridge, Zigbee2MQTT, Home Assistant 구성을 바로 갈아엎으면 삽질할 가능성이 높다. 동시 지원은 모든 Hue 제품이 아니라 최신 칩을 쓴 일부 제품부터 적용될 흐름이고, Home Assistant 쪽에서도 Zigbee와 Thread는 진단 도구와 장애 지점이 다르다.

내 기준 운영 구성은 이렇게 나눈다.

| 역할 | 컨테이너/장치 | 고정할 것 |
|---|---|---|
| Home Assistant Core | `homeassistant/home-assistant:2026.6.4` | 이미지 태그 |
| MQTT 브로커 | `eclipse-mosquitto:2` | 계정, 데이터 볼륨 |
| Zigbee | Zigbee2MQTT + USB 동글 | `/dev/serial/by-id` 경로 |
| Matter | Matter Server | 별도 컨테이너 로그 |
| Thread | ZBT-2, SkyConnect, border router | 펌웨어와 네트워크 이름 |

핵심은 Zigbee 동글을 `/dev/ttyUSB0`로 잡지 않는 것이다. NAS를 재부팅하거나 USB 허브를 바꾸면 번호가 바뀐다. 아래처럼 실제 장치 ID를 확인한다.

```bash
ls -l /dev/serial/by-id/
```

Docker Compose에는 긴 by-id 경로를 그대로 넣는다.

```yaml
services:
  zigbee2mqtt:
    image: koenkk/zigbee2mqtt:2
    restart: unless-stopped
    volumes:
      - ./zigbee2mqtt:/app/data
      - /run/udev:/run/udev:ro
    devices:
      - /dev/serial/by-id/usb-ITead_Sonoff_Zigbee_3.0_USB_Dongle_Plus:/dev/ttyUSB0
    ports:
      - "8080:8080"
    environment:
      - TZ=Asia/Seoul
```

`configuration.yaml`에서는 Home Assistant 자동 발견과 MQTT 주소만 단순하게 둔다. 예전에 [Zigbee2MQTT 기본 설정]({% post_url 2026-06-11-10-00-00-086415-zigbee2mqtt-setup-no-hub %})을 할 때는 여기서 포트를 자주 바꿨는데, 실제 운영에서는 바꾸지 않는 게 낫다.

```yaml
homeassistant: true
frontend:
  port: 8080
mqtt:
  server: mqtt://mosquitto:1883
serial:
  port: /dev/ttyUSB0
advanced:
  network_key: GENERATE
```

Matter/Thread 기기는 같은 Compose 파일에 억지로 끼워 넣지 않았다. Matter Server 로그와 Zigbee2MQTT 로그가 섞이면 장애 때 원인을 찾기 어렵다. Hue 조명이 Thread에도 붙고 Zigbee에도 붙는 시대가 와도, 운영 관점에서는 “조명 자동화가 MQTT에서 오는지, Matter에서 오는지”를 방 단위로 나눠야 한다.

주의할 점은 세 가지다. Hue Bridge를 쓰던 조명을 Zigbee2MQTT에 다시 페어링하면 기존 Hue 앱 자동화는 끊긴다. Thread 전환 테스트를 할 때도 공장 초기화가 필요한 제품이 있어 가족이 쓰는 조명부터 건드리면 바로 불편해진다. 또 Home Assistant 2026.7처럼 월간 릴리스가 나오는 날에는 Core, Zigbee2MQTT, Matter Server를 한꺼번에 올리지 말고 하루 간격을 둔다.

출처는 2026년 7월 1일 기준 Home Assistant 공식 릴리스 주기 안내와 2026년 6월 24일 The Verge의 Philips Hue Thread·Zigbee 동시 지원 보도를 확인했다. 실제 NAS 운영에서는 최신 기능보다 장애 범위를 작게 만드는 구성이 더 오래 간다.

짧게 정리하면 이렇다. 기존 Zigbee 자동화는 Zigbee2MQTT에 남긴다. 새 Matter/Thread 기기는 별도 경로로 붙인다. USB 동글은 by-id로 고정한다. 업데이트는 Core, MQTT, Zigbee, Matter를 한 번에 묶지 않는다.

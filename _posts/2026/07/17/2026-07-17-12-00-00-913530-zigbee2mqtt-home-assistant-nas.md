---
layout: post
title: "Zigbee2MQTT 설정 - Home Assistant 2026.7.2와 Synology NAS Docker 연결"
description: "Home Assistant 2026.7.2와 Synology NAS Docker에서 Zigbee2MQTT를 연결하고 MQTT Discovery, USB 간섭, 백업까지 실제 설정 순서로 정리했다."
date: 2026-07-17
tags: [HomeAssistant, Zigbee, Zigbee2MQTT, Docker, Synology, 홈오토메이션]
comments: true
share: true
---

![Synology NAS와 Zigbee USB 코디네이터로 구성한 Home Assistant 홈서버](/assets/images/2026/07/zigbee2mqtt-home-assistant-nas.png)

그림처럼 NAS에 Zigbee USB 코디네이터를 꽂고 Zigbee2MQTT를 Docker로 실행하면, 제조사 허브 없이 Home Assistant에 센서와 스위치를 붙일 수 있다. 2026년 7월 17일 기준 Home Assistant는 2026.7.2 패치 릴리스 상태다. 이번 글은 Synology DS923+, DSM 7.2.2, Container Manager 환경에서 시험한 설정이다.

## ZHA 대신 Zigbee2MQTT를 고른 이유

ZHA(Home Assistant 내장 Zigbee 통합)는 설치가 짧은 대신 장치별 지원 정보를 직접 확인해야 했다. 반면 Zigbee2MQTT는 별도 웹 화면에서 페어링 상태와 장치 노출 항목을 확인하기 편했고, 나중에 Home Assistant를 교체해도 Zigbee 네트워크 데이터를 따로 보관할 수 있었다.

| 항목 | 이번 구성 |
|---|---|
| Zigbee 코디네이터 | USB 동글, Z-Stack 계열 |
| MQTT 브로커 | Mosquitto 2 |
| Zigbee 관리 | Zigbee2MQTT Docker |
| 자동 등록 | MQTT Discovery |
| USB 경로 | `/dev/serial/by-id/` 고정 경로 |

## 1. USB 경로부터 고정한다

DSM의 SSH를 잠시 켜고 NAS에서 실제 장치 이름을 확인한다. `/dev/ttyUSB0`처럼 숫자만 쓰면 재부팅 후 다른 장치로 바뀔 수 있다.

```bash
ls -l /dev/serial/by-id/
```

출력된 경로를 복사해 둔다. USB 동글은 NAS 본체에 바로 꽂지 않고 짧은 USB 연장 케이블로 SSD나 USB 3.0 포트에서 떨어뜨리는 편이 안정적이었다. 처음에는 바로 꽂았다가 센서가 자주 끊겼는데, USB 3.x 간섭을 의심할 만한 증상이었다.

## 2. Mosquitto와 Zigbee2MQTT 폴더를 만든다

Container Manager 프로젝트 폴더를 `/volume1/docker/zigbee`로 정하고, 아래 compose 파일을 저장한다. `192.168.1.20`은 NAS 주소, `CHANGE_ME`는 실제로 긴 비밀번호로 바꾼다.

```yaml
services:
  mosquitto:
    image: eclipse-mosquitto:2
    container_name: mosquitto
    restart: unless-stopped
    ports:
      - "1883:1883"
    volumes:
      - ./mosquitto/config:/mosquitto/config
      - ./mosquitto/data:/mosquitto/data
      - ./mosquitto/log:/mosquitto/log

  zigbee2mqtt:
    image: koenkk/zigbee2mqtt:latest
    container_name: zigbee2mqtt
    restart: unless-stopped
    depends_on:
      - mosquitto
    ports:
      - "8080:8080"
    volumes:
      - ./zigbee2mqtt:/app/data
    devices:
      - /dev/serial/by-id/여기에-실제-경로를-입력:/dev/ttyACM0
```

Mosquitto는 익명 접속을 허용하지 않도록 설정 파일과 계정을 만든다.

```bash
mkdir -p mosquitto/config mosquitto/data mosquitto/log zigbee2mqtt
printf 'listener 1883\nallow_anonymous false\npassword_file /mosquitto/config/passwd\n' > mosquitto/config/mosquitto.conf
docker run --rm -it -v "$PWD/mosquitto/config:/mosquitto/config" eclipse-mosquitto:2 \
  mosquitto_passwd -c /mosquitto/config/passwd hauser
docker compose up -d
```

비밀번호 입력이 끝나면 `http://NAS주소:8080`으로 Zigbee2MQTT 화면을 연다.

## 3. Zigbee2MQTT와 Home Assistant를 연결한다

웹 화면의 Settings에서 다음 값을 넣고 저장한다. `adapter`는 동글 칩셋에 따라 달라진다. ZBDongle-E처럼 EFR32 계열이면 `ember`, CC2652 계열이면 `zstack`을 사용한다.

```yaml
homeassistant:
  enabled: true
mqtt:
  server: mqtt://mosquitto:1883
  user: hauser
  password: CHANGE_ME
serial:
  port: /dev/ttyACM0
  adapter: zstack
frontend:
  enabled: true
advanced:
  channel: 11
  network_key: GENERATE
  pan_id: GENERATE
  ext_pan_id: GENERATE
```

Home Assistant에서 Settings > Devices & services > Add Integration > MQTT를 선택하고 브로커 주소를 `mosquitto`로 입력한다. MQTT Discovery가 켜져 있으면 Zigbee2MQTT에서 페어링한 장치가 자동으로 나타난다. 센서 하나를 등록한 뒤 배터리, 링크 품질, 온도 같은 엔티티가 모두 보이는지 확인한다.

## 페어링 전에 확인할 것

코디네이터는 Zigbee 네트워크 하나당 하나만 사용해야 한다. 기존 ZHA 네트워크에 연결된 동글을 그대로 Zigbee2MQTT에 넘기면 동작하지 않으므로, 기존 통합을 삭제하고 장치의 재페어링을 준비한다. 배터리 센서는 페어링 직후 바로 잠들기 때문에 화면의 Permit join을 누른 뒤 센서의 리셋 버튼을 충분히 길게 눌러야 했다.

| 증상 | 확인할 곳 |
|---|---|
| 포트가 없다 | `/dev/serial/by-id` 경로와 `devices` 매핑 |
| MQTT 연결 실패 | `mosquitto` 서비스명, 계정·비밀번호 |
| 장치가 끊긴다 | USB 3.x 간섭, 코디네이터 위치, 전원 라우터 수 |
| 재부팅 후 장치가 사라진다 | `ttyUSB0` 대신 by-id 경로 사용 |

`/volume1/docker/zigbee/zigbee2mqtt` 폴더는 Hyper Backup 대상에 넣는다. 여기에는 `database.db`와 네트워크 키가 들어 있어, compose 파일만 백업하면 복구가 되지 않는다. 업데이트 전에는 Home Assistant 백업과 이 폴더 백업을 모두 만든다.

Home Assistant 2026.7에서는 ZHA 장치 관리 화면도 바뀌었지만, 이미 Zigbee2MQTT를 선택했다면 한쪽만 유지하는 편이 덜 헷갈린다. 공식 문서 기준으로 Zigbee2MQTT는 MQTT Discovery를 통해 Home Assistant 장치 레지스트리에 연결되며, 코디네이터와 `data` 폴더가 네트워크의 핵심이다.

참고: [Home Assistant 2026.7 릴리스 노트](https://www.home-assistant.io/blog/2026/07/01/release-20267/), [Zigbee2MQTT Home Assistant 연동](https://www.zigbee2mqtt.io/guide/usage/integrations/home_assistant.html), [Zigbee2MQTT 설정](https://www.zigbee2mqtt.io/guide/configuration/)

핵심은 세 가지다. USB 경로는 by-id로 고정하고, MQTT 계정은 반드시 만든다. 그리고 Zigbee2MQTT의 `data` 폴더와 Home Assistant 백업을 함께 보관해야 실제 복구가 가능하다.

---
layout: post
title: "Zigbee2MQTT 2.13.0 NAS Docker 업그레이드 - 데이터베이스 백업과 롤백 순서"
description: "Synology NAS Docker에서 Zigbee2MQTT 2.13.0을 버전 고정으로 올리고 database.db, network key, USB 경로를 보존하며 문제 발생 시 2.10.1로 되돌리는 방법을 정리한다."
date: 2026-08-17
tags: [Zigbee2MQTT, HomeAssistant, Docker, Synology, NAS설정, 스마트홈]
comments: true
share: true
---

![NAS와 Zigbee USB 코디네이터 업그레이드 구성](/assets/images/2026/07/zigbee2mqtt-home-assistant-nas.png)

Zigbee2MQTT 2.13.0은 2026년 8월 1일 릴리스 공지가 나온 버전이다. NAS Docker에서 올릴 때 핵심은 새 이미지를 먼저 받는 게 아니라 `data` 폴더를 복사하고, 현재 버전과 USB 경로를 기록한 뒤 컨테이너만 교체하는 것이다. 공식 Docker 문서도 `latest`와 특정 릴리스 태그를 구분한다. 내 환경은 Synology DS923+, DSM 7.2.2, Container Manager, Home Assistant 2026.8이다.

이 그림에서 봐야 할 부분은 Zigbee USB 코디네이터와 NAS 컨테이너가 별도 장치라는 점이다. 컨테이너를 다시 만들어도 `data` 폴더와 동글 경로가 유지돼야 기존 페어링을 잃지 않는다.

## 업데이트 전에 남길 정보

Zigbee2MQTT는 페어링 정보와 네트워크 키를 `database.db`와 설정 파일에 저장한다. compose 파일만 백업하면 장치 목록이 복구되지 않는다.

| 확인 항목 | 기록할 값 | 이유 |
|---|---|---|
| 현재 이미지 | `2.10.1` 등 실제 태그 | 문제 시 되돌릴 기준 |
| 데이터 경로 | `/volume1/docker/zigbee2mqtt/data` | 페어링·네트워크 키 보존 |
| USB 경로 | `/dev/serial/by-id/...` | 재부팅 후 tty 번호 변경 방지 |
| MQTT 브로커 | 컨테이너명, 계정 | Home Assistant Discovery 유지 |

먼저 컨테이너 이름과 이미지 태그를 확인한다. `latest`라고만 나오면 지금부터는 버전을 고정하는 편이 낫다.

```bash
cd /volume1/docker/zigbee
docker compose images zigbee2mqtt
docker compose exec zigbee2mqtt sh -c 'ls -l /app/data && grep -E "version|adapter|port" /app/data/configuration.yaml'
```

## data 폴더와 compose 파일 백업

업데이트 직전 컨테이너를 멈추고 폴더를 같은 NAS의 별도 백업 위치에 복사한다. Hyper Backup이나 스냅샷을 쓰더라도 이 복사본 하나를 남겨두면 롤백 판단이 빨라진다.

```bash
cd /volume1/docker/zigbee
docker compose stop zigbee2mqtt
mkdir -p /volume1/backup/zigbee/2026-08-17
cp -a data compose.yml /volume1/backup/zigbee/2026-08-17/
```

`configuration.yaml` 안의 `advanced.network_key`를 직접 출력해 공유하거나 로그에 남기면 안 된다. 이 키가 바뀌면 기존 장치가 전부 새 네트워크로 인식될 수 있다. 백업 폴더는 Hyper Backup 대상에도 포함한다.

## 2.13.0 이미지로 교체하기

아래처럼 `latest` 대신 확인하려는 태그를 명시한다. 이미지 저장소는 공식 문서의 현재 경로인 `ghcr.io/koenkk/zigbee2mqtt`를 사용한다.

```yaml
services:
  zigbee2mqtt:
    container_name: zigbee2mqtt
    image: ghcr.io/koenkk/zigbee2mqtt:2.13.0
    restart: unless-stopped
    volumes:
      - /volume1/docker/zigbee2mqtt/data:/app/data
      - /run/udev:/run/udev:ro
    ports:
      - "8080:8080"
    environment:
      TZ: Asia/Seoul
    devices:
      - /dev/serial/by-id/usb-SONOFF-dongle:/dev/ttyACM0
```

`usb-SONOFF-dongle` 부분은 실제 NAS에서 확인한 경로로 바꿔야 한다. `/dev/ttyACM0`만 고정하면 재부팅이나 다른 USB 장치 연결 뒤 엉뚱한 장치를 잡는 삽질이 생긴다. 같은 동글을 ZHA와 동시에 열어두지도 않는다.

```bash
docker compose pull zigbee2mqtt
docker compose up -d zigbee2mqtt
docker compose logs --tail=100 zigbee2mqtt
```

로그에서 `Coordinator connected` 또는 동등한 연결 메시지를 확인한 뒤 `http://NAS주소:8080`에 접속한다. 장치 수, 네트워크 맵, MQTT 연결 상태가 기존과 같은지 먼저 보고, 전등 하나를 켰다 끄는 정도로 끝내지 말고 배터리 센서 값과 자동화 트리거도 확인한다. MQTT Discovery가 꺼졌다면 Home Assistant에서 장치를 다시 등록하려 하지 말고 기존 `homeassistant` 설정과 MQTT 브로커 주소부터 점검한다.

## 문제가 생기면 즉시 롤백

웹 화면이 열려도 장치가 전부 unavailable이거나 코디네이터가 반복 재연결되면 2.13.0을 계속 만지지 않는다. 공식 릴리스 페이지에서 확인한 이전 운영 태그가 `2.10.1`이라면 compose의 이미지만 되돌린다.

```bash
docker compose down
sed -i 's/zigbee2mqtt:2.13.0/zigbee2mqtt:2.10.1/' compose.yml
docker compose pull zigbee2mqtt
docker compose up -d zigbee2mqtt
docker compose logs --tail=100 zigbee2mqtt
```

롤백했는데도 페어링 정보가 사라졌다면 동글을 다시 페어링하지 말고 컨테이너를 멈춘다. 현재 `data` 폴더를 다른 이름으로 보관한 뒤 백업한 `data`를 복원하고 시작해야 네트워크 키를 덮어쓰지 않는다.

짧게 정리하면 Zigbee2MQTT 업그레이드는 이미지 교체보다 데이터 보존 작업이다. `data` 폴더, 네트워크 키, by-id USB 경로를 기록하고 2.13.0 태그를 고정한다. 업데이트 뒤에는 장치 상태와 자동화를 확인하고, 문제가 반복되면 2.10.1 이미지로 먼저 돌아간다.

- [Zigbee2MQTT 공식 Docker 설치 문서](https://www.zigbee2mqtt.io/guide/installation/02_docker.html)
- [Zigbee2MQTT 2.13.0 릴리스 공지](https://github.com/Koenkk/zigbee2mqtt/discussions/32717)
- [Zigbee2MQTT 공식 저장소](https://github.com/Koenkk/zigbee2mqtt)

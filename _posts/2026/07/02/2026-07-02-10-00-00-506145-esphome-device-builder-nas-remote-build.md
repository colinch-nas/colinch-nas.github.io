---
layout: post
title: "ESPHome Device Builder - NAS Docker를 펌웨어 빌드 서버로 쓰기"
description: "ESPHome 2026.6.4 기준 Device Builder 원격 빌드 기능을 NAS Docker에 분리해 Home Assistant 서버 부하를 줄이는 설정법."
date: 2026-07-02
tags: [HomeAssistant, ESPHome, Docker, 스마트홈, 홈서버]
comments: true
share: true
---
![NAS에서 ESPHome 펌웨어 빌드 서버를 운영하는 홈서버 책상](https://images.unsplash.com/photo-1518770660439-4636190af475?w=800&q=80)

ESPHome 기기가 여러 대라면 Home Assistant가 돌아가는 NAS와 펌웨어 빌드 작업을 분리하는 게 낫다. 2026년 7월 2일 기준 ESPHome 2026.6.4는 새 Device Builder를 기본 흐름으로 밀고 있고, 원격 빌더(Remote builder, 다른 장비에서 컴파일을 대신 수행하는 기능)를 쓸 수 있다. 조명 자동화 서버에서 ESP32 컴파일까지 같이 돌리면 업데이트 날 CPU가 튀는 걸 몇 번 겪었다.

## 왜 빌드 서버를 따로 두나

ESPHome은 YAML 몇 줄처럼 보여도 실제 업데이트 때는 PlatformIO가 보드 패키지, 라이브러리, C++ 컴파일을 수행한다. Synology DS923+나 QNAP TS-464처럼 x86 NAS면 버티지만, 동시에 Home Assistant, Zigbee2MQTT, Matter Server, Jellyfin까지 돌면 빌드 중 UI가 느려진다.

특히 ESPHome 2026.6.0부터 legacy dashboard가 Device Builder로 바뀌었다. 공식 변경 로그에는 Device Builder 1.0, firmware job queue, remote builder, bulk action이 들어갔다고 나온다. 이 흐름이면 ESPHome을 "HA 안의 작은 부가기능"으로만 보기보다 별도 작업 서버로 다루는 편이 운영이 편하다.

## NAS Docker에 ESPHome을 띄운다

아래 예시는 `/volume1/docker/esphome-builder`를 쓰는 Synology 기준이다. QNAP이면 `/share/Container/esphome-builder`처럼 실제 공유 폴더 경로로 바꾸면 된다.

```bash
mkdir -p /volume1/docker/esphome-builder/config
mkdir -p /volume1/docker/esphome-builder/cache
```

Compose 파일은 단순하게 둔다. mDNS(같은 네트워크에서 장치를 자동 발견하는 방식) 때문에 `network_mode: host`를 쓴다. 포트 매핑을 따로 쓰면 페어링 목록에 안 뜨는 경우가 있어 여기서 삽질했다.

```yaml
services:
  esphome-builder:
    image: ghcr.io/esphome/esphome:2026.6.4
    container_name: esphome-builder
    network_mode: host
    restart: unless-stopped
    volumes:
      - /volume1/docker/esphome-builder/config:/config
      - /volume1/docker/esphome-builder/cache:/cache
      - /etc/localtime:/etc/localtime:ro
    environment:
      - TZ=Asia/Seoul
      - PLATFORMIO_CORE_DIR=/cache/platformio
```

컨테이너를 올린 뒤 브라우저에서 `http://NAS_IP:6052`로 접속한다.

```bash
cd /volume1/docker/esphome-builder
docker compose up -d
docker logs -f esphome-builder
```

로그에서 `Dashboard is now accessible` 비슷한 줄이 보이면 기본 접속은 된 상태다. 첫 실행 때는 PlatformIO 캐시가 비어 있어 첫 빌드가 느리다. 두 번째부터 빨라지는 게 정상이다.

## Home Assistant 쪽과 페어링한다

Home Assistant에 ESPHome 애드온이나 ESPHome 통합이 이미 있다면 Device Builder 화면으로 들어간다. 메뉴 이름은 버전에 따라 조금 다르지만 흐름은 이렇다.

```text
ESPHome -> Settings -> Send builds -> Known dashboards
```

NAS의 `esphome-builder`가 보이면 Pair를 누른다. 반대로 NAS 쪽 Device Builder 화면에서는 아래 위치를 열어둔다.

```text
Settings -> Build server -> Pairing requests
```

양쪽에 표시되는 fingerprint(페어링 확인용 짧은 식별값)가 같은지 보고 승인한다. 이 화면을 닫아두면 요청이 막힐 수 있다. 나는 여기서 "왜 발견은 되는데 승인 버튼이 안 뜨지" 하고 시간을 날렸다.

## 실제 빌드 전에 확인할 것

ESP8266 구형 플러그를 쓰면 ESPHome 2026.6 계열의 WiFi 기본 보안 변경을 봐야 한다. WPA-only 공유기에 붙어 있던 기기는 `min_auth_mode: WPA`를 명시하지 않으면 업데이트 후 WiFi에 못 붙을 수 있다.

```yaml
wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  min_auth_mode: WPA
```

가능하면 WPA2 이상으로 공유기를 바꾸는 게 맞다. 다만 낡은 IoT 전용 AP를 아직 쓰는 집이라면 업데이트 전에 기기별 YAML을 확인한다. 빌드 서버 분리는 컴파일 부하를 줄여줄 뿐, 깨지는 설정을 자동으로 고쳐주지는 않는다.

## 운영 팁

빌드 서버의 `/cache`는 지우지 않는 편이 좋다. 캐시를 날리면 매번 보드 패키지를 다시 받아 빌드 시간이 길어진다. 대신 ESPHome 버전을 올릴 때는 태그를 숫자로 고정해 한 번에 바뀌지 않게 한다. `latest`를 쓰면 어느 날 Device Builder UI와 빌드 동작이 같이 바뀌어 원인 찾기가 어렵다.

NAS CPU가 약하면 동시에 여러 기기를 bulk update하지 않는다. Device Builder에 작업 큐가 있어도 ESP32 여러 대를 한꺼번에 컴파일하면 팬이 돌고 Home Assistant DB 쓰기까지 같이 느려질 수 있다. 나는 전등, 전력계, 센서처럼 생활에 영향이 큰 순서대로 한 대씩 올린다.

짧게 정리하면 이렇다. ESPHome 2026.6.4에서는 Device Builder를 NAS Docker에 따로 띄우고, host network로 mDNS를 살린 뒤, Home Assistant와 페어링해 빌드를 넘긴다. `/cache`는 유지하고 이미지 태그는 고정한다. 펌웨어 컴파일은 자동화 서버의 본업이 아니므로 분리하는 쪽이 장애 원인을 줄인다.

## 참고한 공식 자료

- [ESPHome 2026.6.0 변경 로그](https://esphome.io/changelog/2026.6.0/)
- [Home Assistant 월간 릴리스 일정 안내](https://www.home-assistant.io/faq/release/)

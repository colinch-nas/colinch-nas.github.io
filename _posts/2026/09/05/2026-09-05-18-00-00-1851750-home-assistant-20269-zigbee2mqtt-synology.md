---
layout: post
title: "Home Assistant 2026.9 Zigbee2MQTT 설정 - Synology NAS에서 USB 코디네이터 연결하기"
description: "Home Assistant 2026.9과 Synology DSM 7.2.2에서 Mosquitto·Zigbee2MQTT를 Docker로 구성하고 USB 코디네이터와 Zigbee 센서를 연결하는 실전 설정법을 정리했다."
date: 2026-09-05
tags: [HomeAssistant, Zigbee, Zigbee2MQTT, Synology, Docker, 스마트홈]
comments: true
share: true
---

![Synology NAS Home Assistant Zigbee2MQTT 구성](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

Home Assistant 2026.9을 Synology NAS에서 운영한다면 Zigbee2MQTT를 붙이는 구성이 가장 확장하기 편하다. Home Assistant 공식 2026.9 릴리스에서도 여러 기기가 하나의 연결을 공유하는 방향이 강화됐지만, Zigbee 센서 연결은 여전히 USB 코디네이터와 MQTT 브로커 구성이 핵심이다. 여기서는 DS923+, DSM 7.2.2, Container Manager 환경을 기준으로 한다.

## 준비물과 구성

Zigbee2MQTT는 Zigbee 기기와 MQTT 사이를 중계하는 서비스다. MQTT(Message Queuing Telemetry Transport)는 센서 상태를 메시지로 전달하는 통신 방식이다.

| 항목 | 예시 값 | 확인할 점 |
|---|---|---|
| NAS | Synology DS923+ | Container Manager 설치 |
| 홈 자동화 | Home Assistant 2026.9 | MQTT 통합 사용 |
| 코디네이터 | USB Zigbee 동글 | NAS에 직접 연결 |
| 브로커 | Eclipse Mosquitto 2.x | 내부 포트 1883 |

USB 동글은 NAS 뒤쪽 포트보다 짧은 USB 연장 케이블에 연결하는 편이 낫다. NAS와 USB 3.0 간섭 때문에 처음에는 기기가 검색되지 않았는데, 30cm 연장선을 쓰고 해결됐다. DSM의 `제어판 → 정보 센터 → USB 장치`에서 동글이 보이는지도 확인한다.

## Docker Compose로 Mosquitto와 Zigbee2MQTT 실행

Container Manager에서 프로젝트를 만들고 `/volume1/docker/zigbee` 폴더를 생성한다. 그 아래에 `compose.yaml`을 저장한다. `ttyUSB0`는 장치마다 달라질 수 있으니 실제 경로로 바꿔야 한다.

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
      - /dev/ttyUSB0:/dev/ttyUSB0
    environment:
      - TZ=Asia/Seoul
```

Mosquitto 설정 파일은 익명 접속을 끄는 최소 설정으로 만든다. 같은 폴더의 `mosquitto/config/mosquitto.conf`에 아래 내용을 넣고 계정은 별도로 생성한다.

```conf
listener 1883
allow_anonymous false
password_file /mosquitto/config/passwd
```

Mosquitto 컨테이너를 올리기 전에 NAS의 SSH 터미널에서 아래 명령으로 비밀번호 파일을 만든다. 파일이 없으면 `allow_anonymous false` 설정 때문에 브로커가 시작되지 않는다.

```bash
docker run --rm -it \
  -v /volume1/docker/zigbee/mosquitto/config:/mosquitto/config \
  eclipse-mosquitto:2 \
  mosquitto_passwd -c /mosquitto/config/passwd ha
```

비밀번호를 Compose 파일에 직접 적지 않아도 돼서 백업 파일이 노출됐을 때 위험이 조금 줄어든다. 이제 Container Manager에서 프로젝트를 배포한다.

## Zigbee2MQTT 초기 설정

프로젝트를 올린 뒤 `http://NAS_IP:8080`을 연다. MQTT 서버는 컨테이너 이름을 사용하므로 `mqtt://mosquitto:1883`으로 입력한다. 계정과 비밀번호는 방금 만든 값을 쓴다. 코디네이터 항목에는 `/dev/ttyUSB0`을 입력하고, 어댑터는 동글 제조사에 맞춘다.

Zigbee2MQTT의 `configuration.yaml`을 직접 수정한다면 핵심은 다음과 같다.

```yaml
homeassistant: true
permit_join: false
mqtt:
  server: mqtt://mosquitto:1883
  user: ha
  password: 여기에_브로커_비밀번호
serial:
  port: /dev/ttyUSB0
frontend:
  enabled: true
  port: 8080
advanced:
  network_key: GENERATE
```

`permit_join`은 평상시 `false`로 둔다. 새 기기를 등록할 때 웹 화면에서 60초만 허용하는 방식이 안전하다. `network_key`를 바꾸면 기존 Zigbee 기기를 전부 다시 페어링해야 하므로 백업·복원 과정에서도 이 값을 보존해야 한다.

## Home Assistant 연결과 실패 지점

Home Assistant에서 `설정 → 기기 및 서비스 → 통합 추가 → MQTT`를 선택한다. 브로커 주소에 NAS의 IP와 1883 포트를 넣고, 같은 계정으로 로그인하면 Zigbee2MQTT가 발견한 기기가 자동으로 나타난다.

가장 많이 막히는 지점은 USB 장치 권한과 경로다. 컨테이너 로그에 `No such file or directory`가 나오면 `/dev/ttyUSB0` 대신 `/dev/ttyACM0`인지 확인한다. `Permission denied`라면 동글을 사용하는 다른 컨테이너를 중지하고, DSM 재부팅 뒤 장치 매핑을 다시 적용한다. 코디네이터를 옮긴 뒤에는 Zigbee 네트워크가 불안정해질 수 있으므로 페어링이 끝난 뒤 위치를 고정한다.

### 운영 전 체크리스트

- MQTT 익명 접속을 끄고 강한 비밀번호를 사용했는가
- Zigbee2MQTT 데이터 폴더와 `network_key`를 NAS 백업에 포함했는가
- `permit_join`을 끈 상태인가
- 동글을 USB 3.0 포트와 공유기 바로 옆에서 떼어 놓았는가
- 외부 공개는 8080·1883 포트 포워딩 없이 VPN으로 제한했는가

핵심은 Zigbee2MQTT 웹 화면이 열리는 것보다 Home Assistant가 MQTT를 통해 기기 상태를 안정적으로 받는지 확인하는 데 있다. DS923+에서는 브로커와 Zigbee2MQTT를 같은 Docker 프로젝트로 묶고, USB 경로와 데이터 폴더만 고정해도 이후 센서 추가가 크게 편해진다.

참고 자료: [Home Assistant 2026.9 릴리스 노트](https://www.home-assistant.io/blog/2026/09/02/release-20269/), [Zigbee2MQTT 공식 문서](https://www.zigbee2mqtt.io/guide/installation/01_linux.html)

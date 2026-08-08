---
layout: post
title: "Zigbee2MQTT 설정 - NAS Docker에서 USB 코디네이터 연결하고 Home Assistant 연동하기"
description: "Zigbee2MQTT 2.x를 NAS Docker에 설치하고 USB 코디네이터, Mosquitto MQTT, Home Assistant를 연결하는 실제 설정과 권한 오류 해결법을 정리한다."
date: 2026-08-09
tags: [Zigbee, Zigbee2MQTT, HomeAssistant, Docker, NAS설정, 스마트홈]
comments: true
share: true
---

![Zigbee2MQTT NAS Docker 스마트홈 구성](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

Zigbee2MQTT 2.x는 NAS Docker에서 운영하기 좋은 Zigbee 게이트웨이다. USB 코디네이터 하나를 NAS에 연결하고 Mosquitto MQTT 브로커와 Home Assistant를 같은 Docker 네트워크에 넣으면, 제조사 허브 없이 센서와 조명을 직접 관리할 수 있다. 내가 처음 설치할 때 가장 오래 걸린 부분은 YAML 문법이 아니라 USB 장치 경로와 컨테이너 권한이었다.

## 이번 구성에서 확인할 것

| 항목 | 설정값 | 확인 이유 |
|---|---|---|
| NAS | Synology DSM 7.2.x, Container Manager | Docker 프로젝트 실행 기준 |
| 코디네이터 | USB Zigbee 어댑터, `/dev/ttyUSB0` | 재부팅 뒤 경로가 바뀌지 않는지 확인 |
| MQTT | Mosquitto 2.x | Zigbee2MQTT와 Home Assistant 사이의 메시지 중계 |
| Zigbee2MQTT | 2.x 계열 | 설정 파일 형식과 권한을 버전에 맞춤 |

그림에서 봐야 할 부분은 Zigbee 장치가 Home Assistant에 직접 붙는 것이 아니라, 코디네이터 → Zigbee2MQTT → MQTT → Home Assistant 순서로 연결된다는 점이다.

## 1. NAS에서 USB 코디네이터 경로 확인

코디네이터를 NAS에 꽂은 뒤 Container Manager의 터미널이나 SSH에서 실제 장치명을 확인한다. DSM에서 장치가 보이지 않으면 컨테이너 설정을 바꿔도 연결되지 않는다.

```bash
ls -l /dev/ttyUSB* /dev/ttyACM* 2>/dev/null
```

`/dev/ttyUSB0` 또는 `/dev/ttyACM0`가 출력되면 그 값을 메모한다. USB 허브를 거치면 재부팅 후 번호가 달라질 수 있어 코디네이터는 NAS 본체에 직접 연결하는 편이 낫다. Wi-Fi 공유기 바로 옆에 두면 간섭도 생기므로 짧은 USB 연장선으로 NAS와 조금 떨어뜨렸다.

## 2. Mosquitto와 Zigbee2MQTT 프로젝트 만들기

공유 폴더에 `zigbee2mqtt/data`를 만들고, 아래 예시처럼 `compose.yaml`을 저장한다. 이미 MQTT 브로커가 있다면 Mosquitto 부분은 빼고 접속 정보만 바꾸면 된다.

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
    image: koenkk/zigbee2mqtt:2
    container_name: zigbee2mqtt
    restart: unless-stopped
    depends_on:
      - mosquitto
    ports:
      - "8080:8080"
    volumes:
      - ./zigbee2mqtt/data:/app/data
    devices:
      - /dev/ttyUSB0:/dev/ttyUSB0
    environment:
      - TZ=Asia/Seoul
```

Mosquitto 설정 폴더에는 최소한 익명 접속을 막는 설정을 넣는다.

```conf
listener 1883
allow_anonymous false
password_file /mosquitto/config/passwd
```

비밀번호 파일은 Mosquitto 컨테이너에서 생성한다. 계정과 비밀번호를 YAML에 그대로 쓰는 것보다 이 파일을 별도로 백업하는 편이 안전하다.

```bash
docker exec -it mosquitto mosquitto_passwd -c /mosquitto/config/passwd hauser
```

## 3. Zigbee2MQTT 초기 설정

첫 실행 뒤 `zigbee2mqtt/data/configuration.yaml`을 열어 MQTT와 어댑터를 지정한다. 코디네이터 제조사에 따라 `adapter` 값이 다를 수 있으므로, 아래 값이 연결되지 않으면 제품 설명서의 칩셋을 다시 확인한다.

```yaml
homeassistant:
  enabled: true

permit_join: false

mqtt:
  base_topic: zigbee2mqtt
  server: mqtt://mosquitto:1883
  user: hauser
  password: "여기에-MQTT-비밀번호"

serial:
  port: /dev/ttyUSB0
  adapter: ember
  baudrate: 115200

frontend:
  enabled: true
  port: 8080
```

`adapter: ember`는 Ember 계열 코디네이터 예시다. TI 계열이면 `zstack`, 일부 구형 장치면 `deconz`가 필요할 수 있다. 로그에 `Adapter disconnected`가 반복되면 이 값과 포트 권한부터 확인한다. DSM의 Container Manager에서 장치 매핑을 추가했는데도 실패한다면 컨테이너 재생성 후 다시 확인해야 했다.

## 4. Home Assistant 연동과 페어링

Zigbee2MQTT 웹 화면 `http://NAS주소:8080`에 접속해 MQTT 연결 상태가 정상인지 확인한다. Home Assistant에서는 설정 → 기기 및 서비스 → 통합구성요소 추가에서 MQTT를 선택하고, 브로커 주소는 NAS의 IP와 `1883` 포트를 입력한다. Docker가 같은 사용자 정의 네트워크에 있다면 `mosquitto`를 호스트로 쓸 수 있지만, 처음에는 NAS IP를 쓰는 쪽이 문제를 좁히기 쉽다.

그 다음 Zigbee2MQTT의 허용 조인을 잠깐 켜고 센서의 페어링 버튼을 누른다. 등록이 끝나면 즉시 `permit_join: false`로 되돌린다. 페어링 중인 장치는 NAS에서 1~2m 떨어진 곳에 두고, 정상 등록 후 원하는 위치로 옮겨 링크 품질을 확인한다.

## 삽질 포인트와 운영 체크리스트

- `/dev/ttyUSB0`가 안 보이면 NAS가 USB 장치를 인식했는지부터 확인한다. 컨테이너 내부에서만 경로를 만들 수는 없다.
- 코디네이터를 두 개의 Zigbee 프로그램이 동시에 열면 한쪽은 반드시 연결에 실패한다.
- `permit_join`을 계속 켜두지 않는다. 주변 장치가 의도치 않게 네트워크에 들어올 수 있다.
- Zigbee2MQTT `data` 폴더와 Mosquitto `data`, `config`를 함께 백업한다. 장치 목록만 복구하고 네트워크 키를 잃으면 재페어링이 필요할 수 있다.
- 외부 접속은 8080과 1883을 공유기에 포워딩하지 않는다. Home Assistant 외부 접속이 필요하면 VPN이나 인증된 역방향 프록시만 사용한다.

처음부터 완벽하게 묶으려 하지 말고, 코디네이터 인식 → MQTT 연결 → Zigbee2MQTT 웹 화면 → Home Assistant 발견 순서로 한 단계씩 확인하면 된다. 핵심은 최신 컨테이너 태그보다 설정 폴더 백업과 USB 경로 고정이다.

참고: [Zigbee2MQTT 공식 문서](https://www.zigbee2mqtt.io/), [Home Assistant 2026.8 릴리스 노트](https://www.home-assistant.io/blog/2026/08/05/release-20268/)

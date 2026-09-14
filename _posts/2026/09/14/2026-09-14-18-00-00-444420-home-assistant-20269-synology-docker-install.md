---
layout: post
title: "Home Assistant 2026.9.2 설치 - Synology NAS Docker로 스마트홈 서버 만들기"
description: "Home Assistant 2026.9.2를 Synology DSM 7.2.x Docker에 설치하는 방법을 정리한다. host 네트워크, 설정 폴더 권한, Zigbee 동글 연결과 업데이트 주의점까지 실제 운영 기준으로 다룬다."
date: 2026-09-14
tags: [HomeAssistant, Synology, Docker, 스마트홈, 홈서버구축, Zigbee]
comments: true
share: true
---

![Synology NAS에서 Home Assistant 2026.9.2를 Docker로 실행하는 홈서버](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

Home Assistant 2026.9.2는 Synology NAS의 Docker에서 운영할 수 있다. DSM 7.2.x, Container Manager 기준으로 `host` 네트워크와 `/config` 볼륨을 사용해 설치한다. 2026.9에서 Modbus 장치 연동 기반이 정리됐지만, Container 방식은 Home Assistant OS처럼 앱(Add-on)을 자동 관리하지 않으므로 업데이트와 백업은 직접 챙겨야 한다.

## 설치 환경과 방식 선택

| 항목 | 기준 |
|---|---|
| NAS | Synology x86_64, 메모리 4GB 이상 권장 |
| DSM | 7.2.x |
| Docker UI | Container Manager |
| Home Assistant | 2026.9.2, `stable` 이미지 |
| 접속 주소 | `http://NAS주소:8123` |

Container 설치는 Docker Engine과 수동 업데이트가 필요하다. Thread·Z-Wave나 앱 설치 화면이 중요하면 Home Assistant OS를 VM으로 운영하는 편이 낫다.

## 공유 폴더와 Compose 파일 만들기

DSM에서 `File Station → docker` 아래에 `homeassistant/config` 폴더를 만든다. Container Manager의 프로젝트 생성 화면에서 프로젝트 이름을 `homeassistant`로 지정하고, 아래 Compose 내용을 붙여 넣는다. `stop_grace_period`를 넣은 이유는 SQLite 데이터베이스가 저장될 시간을 확보하기 위해서다.

```yaml
services:
  homeassistant:
    container_name: homeassistant
    image: ghcr.io/home-assistant/home-assistant:2026.9.2
    restart: unless-stopped
    privileged: true
    network_mode: host
    environment:
      TZ: Asia/Seoul
    volumes:
      - /volume1/docker/homeassistant/config:/config
      - /etc/localtime:/etc/localtime:ro
      - /run/dbus:/run/dbus:ro
    stop_grace_period: 60s
```

`network_mode: host`는 Home Assistant가 NAS의 8123 포트를 직접 사용하게 한다. 자동 검색(mDNS)이 필요한 스마트 전구나 Matter 장치 연결에서 bridge 네트워크보다 시행착오가 적다. NAS에서 8123 포트를 이미 쓰고 있다면 충돌하므로 배포 전에 역방향 프록시 설정도 확인한다.

`/run/dbus`는 Bluetooth 통합에 필요하다. Zigbee USB 동글은 모델마다 장치 경로가 다르므로 SSH에서 실제 경로를 확인한다.

```bash
ls -l /dev/ttyUSB* /dev/ttyACM* 2>/dev/null
```

경로가 확인되면 Compose에 `devices`를 추가한다. 아래는 `/dev/ttyUSB0`로 인식된 경우다.

```yaml
    devices:
      - /dev/ttyUSB0:/dev/ttyUSB0
```

## 실행과 초기 설정

프로젝트를 배포하고 1~2분 뒤 `http://NAS주소:8123`을 연다. 계정을 만들고 위치를 서울로 지정한다. 기존 인스턴스라면 백업 복원을 선택한다.

계속 재시작되면 Container Manager 로그에서 `Permission denied`, `address already in use`를 찾는다. 전자는 config 폴더 권한, 후자는 포트 충돌이다. NAS 방화벽을 켜뒀다면 내부망의 8123 접근도 허용한다.

## 업데이트는 이미지 고정 후 진행

`stable` 태그만 사용하면 이미지 변경 시점을 추적하기 어렵다. 이번처럼 `2026.9.2`를 고정하면 테스트 후 업데이트할 수 있다. 업데이트 전에는 `설정 → 시스템 → 백업`에서 전체 백업을 내려받는다.

```bash
docker compose pull homeassistant
docker compose up -d
docker compose logs -f homeassistant
```

NAS 재부팅 뒤 USB 동글 경로가 바뀌어 Zigbee가 사라지는 경우가 있다. 동글은 짧은 연장 케이블로 NAS와 떨어뜨리고, `/dev/serial/by-id/` 경로가 보이면 그 경로를 고정해 쓰는 편이 안전하다.

## 운영 전 체크리스트

- 관리자 계정에 2단계 인증을 켠다.
- 8123 포트를 공유기에 직접 포워딩하지 않는다. 외부 접속은 Tailscale 또는 인증된 역방향 프록시를 사용한다.
- Home Assistant 백업과 NAS 자체 백업을 분리한다.
- Container 방식에는 Home Assistant OS의 앱 관리 화면이 없다는 점을 기억한다.
- Zigbee 동글을 추가한 뒤에는 컨테이너 재생성 후 장치 경로를 다시 확인한다.

Home Assistant 2026.9.2를 Synology에 올리는 핵심은 `host` 네트워크, 영속 `/config`, 60초 종료 시간이다. 이 세 가지를 맞추면 자동 검색과 재부팅 후 복구가 수월하다. 공식 변경 사항은 [Home Assistant 2026.9 릴리스 노트](https://www.home-assistant.io/blog/2026/09/02/release-20269/), 설치 명령은 [Home Assistant Container 공식 문서](https://www.home-assistant.io/installation/linux)를 기준으로 확인하면 된다.

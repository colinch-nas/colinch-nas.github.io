---
layout: post
title: "Home Assistant 2026.8 Docker 업데이트 - NAS에서 안전하게 올리고 되돌리는 방법"
description: "Home Assistant 2026.8.0을 Synology NAS Docker에서 업데이트하는 절차와 백업, compose 태그 고정, Zigbee 장치 점검 방법을 정리한다."
date: 2026-08-06
tags: [HomeAssistant, Docker, Synology, NAS설정, 스마트홈]
comments: true
share: true
---

![Home Assistant 2026.8 NAS Docker 업데이트](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

Home Assistant 2026.8.0이 2026년 8월 5일 공개됐다. NAS Docker에서 업데이트할 때는 `latest` 이미지를 무작정 당기는 것보다 설정 폴더를 복사하고, `2026.8.0`으로 태그를 고정한 뒤 자동화와 Zigbee 장치까지 확인하는 순서가 안전하다. 공식 문서도 Container 방식은 업데이트를 직접 관리해야 하고, Home Assistant OS처럼 앱과 원클릭 업데이트가 포함되지 않는다고 구분한다.

## 이번 업데이트에서 확인할 점

2026.8은 이름만 보고 바로 운영 서버에 적용하기보다, 백업과 복구 경로를 확인하기 좋은 시점이다. Container 설치는 Home Assistant 앱(Add-on)을 쓸 수 없고 Thread·Z-Wave처럼 별도 앱 구성이 필요한 기능은 추가 작업이 생긴다.

| 항목 | NAS Docker 기준 판단 |
|---|---|
| 이미지 | `ghcr.io/home-assistant/home-assistant:2026.8.0`으로 고정 |
| 설정 경로 | 호스트의 `/volume1/docker/homeassistant/config` |
| 접속 포트 | `8123`, 외부 공개 없이 내부망에서 테스트 |
| 백업 | HA 백업 + NAS 스냅샷 또는 Hyper Backup |

이 그림에서 봐야 할 부분은 Home Assistant 컨테이너와 NAS 저장소가 분리돼 있다는 점이다. 컨테이너를 지워도 `/config`가 남아 있어야 복구할 수 있다.

## 업데이트 전 백업

Synology DS923+와 DSM 7.2.2에서 `/volume1/docker/homeassistant`를 사용한다고 가정한다. SSH를 켰다면 아래처럼 compose 폴더로 이동해 현재 상태를 확인한다.

```bash
cd /volume1/docker/homeassistant
docker compose ps
docker compose exec homeassistant ha backups create
```

`ha backups create`가 동작하지 않는 이미지라면 Home Assistant 화면의 **설정 → 시스템 → 백업**에서 수동 백업을 만든다. 이 백업은 NAS 전체 백업이 아니므로, `config` 폴더가 Hyper Backup이나 스냅샷 대상에 포함됐는지도 따로 확인해야 한다.

## compose 이미지 태그만 바꾸기

기존 `compose.yml`에서 이미지 버전을 `stable` 대신 이번 버전으로 고정한다. `network_mode: host`는 Home Assistant의 자동 검색과 일부 장치 연동 때문에 NAS Docker에서 자주 쓰는 구성이다.

```yaml
services:
  homeassistant:
    container_name: homeassistant
    image: ghcr.io/home-assistant/home-assistant:2026.8.0
    volumes:
      - /volume1/docker/homeassistant/config:/config
      - /etc/localtime:/etc/localtime:ro
      - /run/dbus:/run/dbus:ro
    restart: unless-stopped
    privileged: true
    network_mode: host
    environment:
      TZ: Asia/Seoul
```

`privileged: true`는 편하지만 범위가 넓다. Zigbee 동글이나 Bluetooth를 쓰지 않는 환경이라면 장치 매핑과 권한을 최소화하는 쪽이 낫다. 반대로 Zigbee2MQTT를 별도 컨테이너로 운영 중이면 Home Assistant 컨테이너에 동글을 중복 연결하지 않는다.

```bash
docker compose pull homeassistant
docker compose up -d homeassistant
docker compose logs -f --tail=100 homeassistant
```

로그에 `Error loading`이 반복되지 않고 웹 화면이 열리면 **설정 → 시스템 → 로그**에서 통합 오류를 확인한다. 자동화 하나를 실행해 보고, 조명·스마트 플러그·온습도 센서의 상태가 실제 장치와 맞는지 체크한다. 업데이트 직후에는 UI가 열리는 것만으로 성공했다고 보면 안 된다.

## 문제가 생겼을 때 되돌리기

새 버전에서 커스텀 통합이 깨졌다면 컨테이너를 계속 재시작하지 말고 이미지 태그를 이전에 사용하던 `2026.7.1`로 되돌린다.

```bash
sed -i 's/home-assistant:2026.8.0/home-assistant:2026.7.1/' compose.yml
docker compose pull homeassistant
docker compose up -d homeassistant
```

이 방식은 이미지 롤백이지 설정 파일 롤백은 아니다. 설정 변경까지 되돌려야 한다면 NAS 스냅샷에서 `/config`를 복원한 뒤 컨테이너를 다시 시작한다. 복원 전 현재 폴더를 별도 이름으로 보관하면 원인 분석에 도움이 된다.

외부 접속은 업데이트 확인이 끝난 뒤에 열어야 한다. NAS의 8123 포트를 공유기에 직접 포워딩하지 말고, 기존 역방향 프록시나 Tailscale VPN을 사용한다. 관리자 계정의 2단계 인증과 정기 백업도 함께 확인한다.

짧게 정리하면 Home Assistant 2026.8.0 NAS Docker 업데이트의 기준은 세 가지다. `config` 백업을 실제로 복구할 수 있는지 확인하고, 이미지 태그를 고정하며, 자동화와 Zigbee 장치를 직접 한 번 실행해 본다. 공식 설치 문서 기준으로도 Container는 수동 업데이트 방식이므로, `stable`을 계속 따라가기보다 운영 환경에서는 버전 고정이 덜 불안하다.

- [Home Assistant 2026.8 공식 릴리스](https://www.home-assistant.io/blog/2026/08/05/release-20268/)
- [Home Assistant Container 설치 문서](https://www.home-assistant.io/installation/)

---
layout: post
title: "Home Assistant 업데이트 체크리스트 - NAS Docker에서 2026.6.4 안전하게 올리기"
description: "Home Assistant 2026.6.4를 NAS Docker 환경에서 올리기 전 백업, compose 고정, Matter와 Zigbee 점검, 롤백 순서를 실제 명령어로 정리했다."
date: 2026-07-01
tags: [HomeAssistant, Docker, 스마트홈, NAS설정, 홈오토메이션]
comments: true
share: true
---
![Home Assistant NAS Docker 업데이트 체크리스트](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)

Home Assistant를 NAS Docker로 돌린다면 업데이트 전에 `/config` 백업과 이미지 태그 고정을 먼저 해야 한다. 2026년 7월 1일 기준 공식 사이트의 최신 표기는 Home Assistant 2026.6.4이고, 이번 2026.6 계열은 대시보드와 Matter, Bluetooth 쪽 변경이 많아서 바로 올리기 전에 확인할 게 꽤 있다.

## 환경 정보

| 항목 | 값 |
|------|-----|
| NAS | Synology DS923+ |
| DSM | 7.2.2 |
| 실행 방식 | Docker Compose / Container Manager |
| Home Assistant | 2026.6.4 기준 |
| 컨테이너 이미지 | `ghcr.io/home-assistant/home-assistant` |
| 네트워크 | `host` 모드 |

Home Assistant 2026.6 기능 자체는 이전에 [대시보드 카드 피커 글]({% post_url 2026-06-26-16-00-00-308625-home-assistant-2026-6-new-features-card-picker %})에서 따로 정리했다. 이번 글은 기능 설명보다 "업데이트 버튼 누르기 전에 뭘 백업하고, 실패하면 어떻게 되돌릴지"에 초점을 둔다.

## 왜 NAS Docker 업데이트가 더 조심스러운가

Home Assistant OS로 설치한 경우에는 백업, 애드온, 업데이트 UI가 한 묶음으로 움직인다. 반면 NAS Docker는 내가 컨테이너 생명주기를 직접 관리한다. 이게 자유롭긴 한데, 업데이트 실패도 내 책임이다.

처음에는 `latest` 태그를 쓰고 Watchtower로 자동 업데이트까지 걸어놨다. 며칠은 편했다. 그러다 Zigbee2MQTT 컨테이너가 먼저 올라오고 Home Assistant가 늦게 뜨면서 자동화가 반쯤 죽은 적이 있다. 로그에는 별거 없어 보였다.

```bash
Config entry 'zha' for zha integration not ready yet: Failed to connect
```

이 메시지만 보고 "재시작하면 되겠지" 했다가 한 시간 날렸다. 원인은 업데이트 자체보다 컨테이너 시작 순서와 USB 장치 권한이었다. 그 뒤로 Home Assistant는 자동 업데이트에서 빼고, 한 달에 한 번 직접 올린다.

## 1단계: 현재 버전과 변경점 확인

업데이트 전에는 현재 컨테이너 버전부터 적어둔다. 나중에 롤백할 때 "아까 뭐였지"가 제일 허무하다.

이 명령으로 현재 실행 중인 Home Assistant 버전과 이미지 ID를 확인한다.

```bash
docker exec homeassistant python -m homeassistant --version
docker inspect homeassistant --format '{{.Config.Image}} {{.Image}}'
```

공식 릴리즈 노트도 같이 본다. 2026.6은 카드 피커, 자동화 편집기, Z-Wave 잠금장치 자격 증명 관리, Matter 기기 추가 흐름, Bluetooth 프록시 스캔 방식 변경이 들어갔다. 특히 Bluetooth 프록시나 Matter, Thread를 쓰면 "UI만 바뀌었겠지" 하고 넘기면 안 된다.

## 2단계: `/config`를 통째로 백업한다

Docker 설치에서 진짜 데이터는 컨테이너가 아니라 `/config` 볼륨이다. compose 파일, 데이터베이스, 자동화 YAML, 통합 설정이 다 여기에 있다.

Synology 기준으로 `/volume1/docker/homeassistant/config`를 쓴다면 이렇게 백업한다.

```bash
cd /volume1/docker/homeassistant

tar -czf "homeassistant-config-$(date +%Y%m%d-%H%M%S).tar.gz" config
ls -lh homeassistant-config-*.tar.gz | tail -3
```

SQLite 데이터베이스까지 포함하려면 Home Assistant를 잠깐 멈추고 백업하는 게 깔끔하다.

```bash
cd /volume1/docker/homeassistant

docker compose stop homeassistant
tar -czf "homeassistant-config-cold-$(date +%Y%m%d-%H%M%S).tar.gz" config
docker compose start homeassistant
```

나는 cold backup을 선호한다. 2분 정도 자동화가 멈추는 건 괜찮은데, 깨진 `home-assistant_v2.db`를 복구하느라 저녁을 버리는 건 싫다.

![NAS Docker 컨테이너 백업과 스마트홈 설정 점검](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?w=800&q=80)

## 3단계: compose에서 태그를 고정한다

공식 Docker 문서는 `stable` 태그를 예시로 쓴다. 설치할 때는 편하지만, 운영 중에는 특정 버전으로 고정하는 편이 낫다. 그래야 롤백이 쉽다.

운영용 compose는 이렇게 버전 태그를 명시한다.

```yaml
services:
  homeassistant:
    container_name: homeassistant
    image: ghcr.io/home-assistant/home-assistant:2026.6.4
    volumes:
      - /volume1/docker/homeassistant/config:/config
      - /etc/localtime:/etc/localtime:ro
      - /run/dbus:/run/dbus:ro
    restart: unless-stopped
    privileged: true
    network_mode: host
```

여기서 `network_mode: host`는 거의 필수에 가깝다. 스마트홈 기기 발견은 mDNS, SSDP, Bluetooth 프록시처럼 브로드캐스트성 네트워크를 많이 쓴다. bridge 네트워크로도 억지로 되는 경우가 있지만, 나중에 Matter나 Chromecast 발견이 안 될 때 원인 찾기가 피곤하다.

## 4단계: Matter와 Zigbee를 먼저 점검한다

2026.6 계열에서 Matter 쪽 변화가 컸다. [Matter 서버 전환 글]({% post_url 2026-06-27-10-00-00-320970-home-assistant-matter-upgrade-matterjs-2026 %})에서도 적었지만, matter.js 기반으로 옮겨가면서 안정성은 좋아졌고 기기 추가 흐름도 바뀌었다. 다만 Docker 환경에서는 Home Assistant Core만 올린다고 주변 컨테이너가 같이 맞춰지는 게 아니다.

Matter Server, Zigbee2MQTT, Mosquitto를 따로 Docker로 돌린다면 업데이트 전 상태를 남겨둔다.

```bash
docker ps --format 'table {{.Names}}\t{{.Image}}\t{{.Status}}' \
  | grep -E 'homeassistant|matter|zigbee|mosquitto'
```

Zigbee USB 동글 경로도 확인한다. Synology 재부팅 뒤 `/dev/ttyUSB0`가 `/dev/ttyUSB1`로 바뀌면 Zigbee2MQTT가 갑자기 못 뜬다.

```bash
ls -l /dev/serial/by-id/
```

compose에는 `/dev/ttyUSB0`보다 `/dev/serial/by-id/...` 경로를 쓰는 게 덜 흔들린다. 예전에 [Zigbee2MQTT 설정 글]({% post_url 2026-06-11-10-00-00-086415-zigbee2mqtt-setup-no-hub %})에서도 이걸 빼먹었다가 컨테이너 재시작 때마다 장치 경로를 다시 맞췄다.

## 5단계: 실제 업데이트 명령

백업과 태그 고정이 끝났으면 업데이트는 단순하다.

compose 파일을 수정한 뒤 새 이미지를 받고 컨테이너를 다시 띄운다.

```bash
cd /volume1/docker/homeassistant

docker compose pull homeassistant
docker compose up -d homeassistant
docker compose logs -f --tail=120 homeassistant
```

로그에서 아래 메시지가 지나가고 웹 UI가 열리면 1차 성공이다.

```bash
Home Assistant initialized
```

웹 접속은 기존처럼 `http://NAS_IP:8123`으로 확인한다. 접속은 되는데 대시보드가 비어 보이면 브라우저 캐시 문제일 때가 있다. 이때 컨테이너를 다시 내리기 전에 시크릿 창으로 먼저 확인한다.

## 6단계: 업데이트 후 바로 눌러볼 것

업데이트가 끝났다고 바로 끝내지 않는다. 나는 아래 다섯 개를 꼭 본다.

| 확인 항목 | 경로 |
|----------|------|
| 복구 모드 여부 | Settings -> System -> Repairs |
| 통합 오류 | Settings -> Devices & services |
| 자동화 실행 | Settings -> Automations & scenes |
| Matter 기기 상태 | Settings -> Devices & services -> Matter |
| Zigbee/MQTT 상태 | Zigbee2MQTT UI, Mosquitto 로그 |

특히 자동화는 하나를 직접 실행해본다. 대시보드가 정상이어도 자동화 조건이나 액션에서 deprecated 옵션이 걸리면 실제 생활 루틴이 망가진다.

이 명령으로 최근 에러만 빠르게 볼 수 있다.

```bash
docker logs homeassistant --since=20m 2>&1 \
  | grep -Ei 'error|failed|traceback|not ready|deprecated'
```

`deprecated`는 당장 장애는 아니지만 다음 버전에서 터질 가능성이 있다. 미루면 나중에 더 귀찮다.

## 롤백은 어떻게 하나

업데이트 후 Matter 기기가 전부 unavailable이 되거나, Zigbee2MQTT 연동이 깨지면 바로 되돌린다. 이때 필요한 게 아까 고정해둔 이미지 태그다.

compose에서 이미지 태그를 이전 버전으로 되돌린다.

```yaml
image: ghcr.io/home-assistant/home-assistant:2026.6.3
```

그 다음 컨테이너만 다시 만든다.

```bash
cd /volume1/docker/homeassistant

docker compose pull homeassistant
docker compose up -d homeassistant
```

그래도 안 되면 `/config` 자체를 백업본으로 되돌린다.

```bash
cd /volume1/docker/homeassistant

docker compose stop homeassistant
mv config "config-broken-$(date +%Y%m%d-%H%M%S)"
tar -xzf homeassistant-config-cold-20260701-100000.tar.gz
docker compose start homeassistant
```

여기서 백업 파일명은 실제 생성한 파일명으로 바꿔야 한다. 급할 때 오타 내기 쉬우니 `ls -lh homeassistant-config*.tar.gz`로 먼저 확인한다.

## 삽질했던 부분

**`stable` 태그는 롤백 기준점이 아니다.** `stable`은 오늘과 내일이 다를 수 있다. 문제 생긴 뒤 `stable`로 다시 받으면 같은 버전이 내려올 수도 있고, 이미 더 새 버전이 내려올 수도 있다. 운영용은 숫자 태그가 마음 편하다.

**Synology Container Manager의 "재생성" 버튼만 믿으면 compose 변경을 놓친다.** UI에서 컨테이너만 재생성하면 내가 기대한 compose 파일이 반영됐는지 헷갈린다. 업데이트할 때는 SSH에서 `docker compose config`로 최종 설정을 확인하는 편이 낫다.

```bash
cd /volume1/docker/homeassistant
docker compose config | sed -n '/homeassistant:/,/^[^ ]/p'
```

**Matter는 Home Assistant만의 문제가 아니다.** Thread 보더 라우터, Matter Server, iOS/Android Companion App, 공유기 mDNS까지 같이 엮인다. Home Assistant Core만 올렸는데 기기 추가가 안 되면 Core 로그만 볼 게 아니라 Matter Server 로그도 같이 봐야 한다.

## 한 줄 정리

Home Assistant 2026.6.4 업데이트는 NAS Docker에서도 어렵지 않다. 대신 `/config` cold backup, 이미지 숫자 태그 고정, Matter와 Zigbee 상태 확인, 롤백 명령까지 준비한 뒤 올려야 한다. 스마트홈 서버는 "켜지는 것"보다 "업데이트 후에도 자동화가 그대로 도는 것"이 더 중요하다.

---

다음 편은 Home Assistant 자동화 편집기에서 조건 테스트와 라벨을 써서 야간 조명 자동화를 정리하는 내용이다.

**참고 링크**
- [Home Assistant 2026.6 릴리즈 노트](https://www.home-assistant.io/blog/2026/06/03/release-20266/)
- [Home Assistant Container 설치 문서](https://www.home-assistant.io/installation/linux/)
- [Docker Compose 공식 문서](https://docs.docker.com/compose/)

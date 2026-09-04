---
layout: post
title: "Home Assistant 2026.9 설치 방법 비교 - Synology NAS Docker로 안전하게 시작하기"
description: "Home Assistant 2026.9을 Synology DS923+ Docker에 설치하고 백업·업데이트·Matter Server 연결까지 실제 운영 순서로 정리한다."
date: 2026-09-04
tags: [HomeAssistant, Synology, Docker, DSM, 홈서버구축, 스마트홈]
comments: true
share: true
---

![Synology NAS에서 Home Assistant Docker를 운영하는 홈서버](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

Home Assistant 2026.9은 2026년 9월 2일 공개됐고, Modbus 장치가 단일 연결을 공유하도록 바뀐 것이 눈에 띈다. Synology NAS 사용자라면 별도 라즈베리파이를 추가하기보다 Docker로 Home Assistant를 올리는 편이 관리하기 쉽다. 이 글은 DS923+, DSM 7.2.x, Docker Manager 기준으로 설치하고 업데이트 전 백업까지 끝내는 과정이다.

## 설치 방법을 고르는 기준

Home Assistant OS(HAOS)는 애드온과 Supervisor를 함께 쓰기 편하지만 NAS에 직접 설치할 수 없다. Synology에서는 Docker 방식이 현실적이다. 다만 HAOS 전용 애드온 화면은 없으므로 MQTT나 Matter Server를 별도 컨테이너로 운영해야 한다.

| 방식 | 장점 | 아쉬운 점 | 추천 상황 |
|---|---|---|---|
| HAOS 전용 미니PC | 애드온·백업 관리가 단순함 | 장비를 하나 더 둬야 함 | 스마트홈이 핵심 서버일 때 |
| Synology Docker | NAS 한 대로 운영, 스냅샷·백업 활용 | 네트워크 모드 설정이 필요함 | 이미 NAS를 24시간 켜두는 경우 |
| Home Assistant Container | 가볍고 업데이트가 빠름 | Supervisor·애드온 없음 | Docker 운영에 익숙한 경우 |

이번 구성은 `host` 네트워크를 사용한다. Home Assistant의 자동 발견(mDNS·SSDP)이 브리지 네트워크에서 누락되는 삽질을 줄이기 위해서다.

## Synology Docker에 컨테이너 만들기

DSM의 **File Station**에서 `/docker/homeassistant/config` 폴더를 만든다. 이 폴더가 설정과 자동화가 저장되는 위치라서 컨테이너를 지워도 데이터가 남는다. Docker Manager의 프로젝트에서 아래 Compose를 붙여 넣는다.

```yaml
services:
  homeassistant:
    container_name: homeassistant
    image: ghcr.io/home-assistant/home-assistant:stable
    restart: unless-stopped
    network_mode: host
    privileged: true
    environment:
      TZ: Asia/Seoul
    volumes:
      - /volume1/docker/homeassistant/config:/config
      - /etc/localtime:/etc/localtime:ro
```

`privileged: true`는 무조건 필요한 옵션은 아니지만 USB Zigbee 동글이나 GPIO 계열 장치를 붙일 때 권한 문제를 줄인다. 외부에서 Compose를 내려받을 때는 이미지 태그를 `stable`로 두기보다 실제 검증한 버전으로 고정하는 편이 안전하다. 2026.9 적용 후 문제가 없으면 그 시점의 이미지 digest를 기록해 둔다.

프로젝트를 배포한 뒤 브라우저에서 `http://NAS_IP:8123`으로 접속한다. 사용자 계정과 집 위치를 만들고, **설정 → 시스템 → 일반**에서 시간대가 `Asia/Seoul`인지 확인한다. 시간이 어긋나면 에너지 통계와 자동화 실행 시각이 모두 틀어진다.

## 업데이트 전 백업과 복구 확인

Home Assistant는 설정 폴더만 백업하면 된다. DSM **Hyper Backup**에서 `/docker/homeassistant/config`를 다른 공유 폴더나 USB 디스크로 주기 백업 대상으로 지정한다. 최소한 `automations.yaml`, `configuration.yaml`, `secrets.yaml`이 실제 대상에 포함되는지 확인한다.

컨테이너 업데이트는 설정 백업 뒤에 진행한다.

```bash
cd /volume1/docker/homeassistant
docker compose pull
docker compose up -d
docker logs --tail 100 homeassistant
```

로그에 `Home Assistant initialized`가 보이고 웹 화면이 열리면 성공이다. 업데이트 직후 자동화만 믿지 말고 조명 켜기, 외출 모드, 전력 센서 갱신을 하나씩 수동 확인한다. 복구 테스트 없이 백업 완료라고 판단한 것이 가장 큰 실수였다. 별도 임시 폴더에 config를 복원해 컨테이너를 띄워보면 백업의 쓸모를 바로 검증할 수 있다.

## Matter와 Thread를 붙일 때 막히는 지점

Matter는 Wi-Fi 또는 Thread 위에서 동작하는 표준이고, Home Assistant Container에서는 Matter Server를 별도 운영해야 한다. 공식 문서도 HAOS의 공식 Matter Server 앱을 권장하며, 32비트 플랫폼은 지원하지 않는다고 안내한다. Synology에서는 Matter Server 컨테이너의 네트워크와 WebSocket 주소를 별도로 관리해야 하므로, 처음부터 Matter 기기를 대량 도입하지 않는 편이 낫다.

Thread 기기를 쓸 때는 Thread Border Router가 필요하고, Home Assistant 호스트까지 IPv6 멀티캐스트가 통과해야 한다. 공유기에서 IoT VLAN을 분리했다면 mDNS·IPv6 차단 때문에 기기 검색이 실패할 수 있다. Zigbee 동글이 이미 안정적으로 동작한다면 Matter/Thread는 한 기기만 시험한 뒤 확장하는 순서가 덜 고생스럽다.

## 짧은 체크리스트

- DSM 방화벽에서 NAS 내부망의 TCP 8123만 허용했는가
- `/config`를 NAS의 다른 위치에도 백업했는가
- 컨테이너 업데이트 전 현재 이미지 버전과 복구 방법을 기록했는가
- Matter를 쓴다면 Matter Server와 Thread Border Router를 구분했는가
- 외부 접속은 8123 포트 직접 공개 대신 VPN이나 인증된 역방향 프록시를 사용했는가

Home Assistant 2026.9을 Synology NAS에서 시작한다면 Docker Container가 장비 추가 비용과 관리 부담 사이에서 균형이 좋다. 다만 HAOS와 같은 애드온 경험을 기대하면 곧 막힌다. `/config` 백업, host 네트워크, 기기 한 대씩 검증하는 세 가지를 지키면 업데이트와 확장 과정에서 되돌아갈 길이 생긴다.

참고한 공식 문서:

- [Home Assistant 2026.9 릴리스 노트](https://www.home-assistant.io/blog/2026/09/02/release-20269/)
- [Home Assistant Matter 통합 문서](https://www.home-assistant.io/integrations/matter)
- [Home Assistant Thread 통합 문서](https://www.home-assistant.io/integrations/thread)

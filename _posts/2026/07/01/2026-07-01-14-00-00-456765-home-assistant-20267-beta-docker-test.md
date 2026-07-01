---
layout: post
title: "Home Assistant 2026.7 베타 테스트 - NAS Docker에 운영 서버 복제해 사전 검증하기"
description: "Home Assistant 2026.7 전환 전 NAS Docker에서 운영 설정을 복제한 테스트 컨테이너로 자동화, Matter, Zigbee 연동을 안전하게 확인하는 방법."
date: 2026-07-01
tags: [HomeAssistant, Docker, 스마트홈, 홈서버, 홈오토메이션]
comments: true
share: true
---
![NAS Docker에서 Home Assistant 테스트 컨테이너를 검증하는 작업 환경](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?w=800&q=80)

Home Assistant 2026.7을 바로 운영 서버에 올리지 말고, NAS Docker에 테스트 컨테이너를 하나 더 띄워 자동화와 Matter, Zigbee 연동부터 확인하면 된다. 2026년 7월 1일 기준 공식 안정판 글은 2026.6이고, 2026.7은 베타 주간과 월간 릴리스 일정상 곧 확인해야 할 버전이다. 스마트홈은 업데이트 실패보다 "조명이 밤새 안 꺼지는" 자동화 꼬임이 더 피곤하다.

## 왜 테스트 컨테이너가 필요한가

Home Assistant OS는 백업과 복원이 UI에 잘 묶여 있다. Synology나 QNAP Docker로 Core를 돌리면 `/config` 폴더, 이미지 태그, 주변 컨테이너를 직접 관리한다. 특히 2026.6 이후 자동화 편집기, 대시보드 카드, Matter 흐름이 계속 바뀌고 있어 운영 서버 하나로 실험하면 원인 분리가 어렵다.

내가 삽질한 부분은 포트였다. 운영 컨테이너와 같은 `8123` 포트를 쓰면 당연히 충돌한다. 테스트용은 `8124`로 열고, 실제 기기 제어는 최대한 하지 않는 방식으로 확인한다.

## 운영 설정을 복제한다

Synology 기준 운영 설정이 `/volume1/docker/homeassistant/config`에 있다면 테스트 폴더를 따로 만든다.

```bash
mkdir -p /volume1/docker/homeassistant-test
rsync -a --delete /volume1/docker/homeassistant/config/ /volume1/docker/homeassistant-test/config/
```

SQLite DB까지 깨끗하게 복제하려면 운영 Home Assistant를 잠깐 멈춘 뒤 복사하는 게 낫다. Recorder DB가 쓰이는 중에 복사하면 테스트 서버 첫 실행에서 복구 로그가 길게 쌓인다.

```bash
docker stop homeassistant
rsync -a --delete /volume1/docker/homeassistant/config/ /volume1/docker/homeassistant-test/config/
docker start homeassistant
```

## 테스트용 compose 파일

운영 서버와 이름, 포트, 설정 경로를 반드시 다르게 둔다. 아래 예시는 `2026.7.0b1`처럼 베타 태그를 직접 넣어 확인하는 방식이다. 실제 태그는 Docker Hub에서 보이는 값으로 바꾼다.

```yaml
services:
  homeassistant-test:
    image: ghcr.io/home-assistant/home-assistant:2026.7.0b1
    container_name: homeassistant-test
    restart: unless-stopped
    ports:
      - "8124:8123"
    volumes:
      - /volume1/docker/homeassistant-test/config:/config
      - /etc/localtime:/etc/localtime:ro
    environment:
      - TZ=Asia/Seoul
```

테스트 컨테이너는 일부 자동 발견이 덜 될 수 있다. `network_mode: host`를 쓰면 실제 운영 서버와 같은 네트워크 발견 조건을 만들 수 있지만, 두 Home Assistant가 동시에 같은 기기를 제어할 위험이 있다. 나는 베타 검증 때는 포트 매핑 방식으로 UI, 로그, 설정 마이그레이션을 우선 보고, Matter나 mDNS 문제를 확인할 때만 운영 서버를 멈춘 뒤 host 모드로 잠깐 테스트한다.

## 자동화가 실제로 실행되지 않게 막는다

테스트 서버가 켜지자마자 조명이나 난방을 건드리면 곤란하다. 접속 후 `Settings -> Automations & scenes`에서 핵심 자동화를 비활성화한다. 더 확실하게 하려면 테스트 복제본의 `automations.yaml`을 잠시 비운다.

```bash
cp /volume1/docker/homeassistant-test/config/automations.yaml \
   /volume1/docker/homeassistant-test/config/automations.yaml.bak
printf "[]\n" > /volume1/docker/homeassistant-test/config/automations.yaml
```

이 상태에서 컨테이너를 올린다.

```bash
cd /volume1/docker/homeassistant-test
docker compose up -d
docker logs -f homeassistant-test
```

로그에서 봐야 할 건 세 가지다. 설정 마이그레이션 오류, 커스텀 통합 로딩 실패, Matter 또는 MQTT 연결 실패다. HACS 커스텀 카드가 많으면 2026.7 베타 UI에서 초반에 깨지는 경우가 있다.

## Matter와 Zigbee는 이렇게 본다

Matter Server, Zigbee2MQTT, Mosquitto를 별도 컨테이너로 운영한다면 테스트 Home Assistant가 같은 서버에 붙으려 할 수 있다. 이때는 읽기 확인만 한다. 기기를 새로 페어링하거나 제거하지 않는다.

확인 경로는 이 정도면 충분하다.

```text
Settings -> Devices & services -> Matter
Settings -> Devices & services -> MQTT
Settings -> System -> Logs
Developer tools -> States
```

센서 상태가 보이는지, 엔티티 이름이 바뀌지 않았는지, 자동화 조건에서 참조하는 entity_id가 남아 있는지 확인한다. 특히 Matter 기기는 재페어링을 함부로 누르면 운영 서버와 페어링 상태가 꼬일 수 있다.

## 운영 반영 기준

테스트 컨테이너에서 30분 이상 로그가 조용하고, 핵심 대시보드와 자동화 조건이 깨지지 않으면 운영 업데이트를 진행한다. 반대로 HACS 카드, 커스텀 통합, Matter 연동 중 하나라도 오류가 반복되면 운영 서버는 그대로 둔다. 베타는 기능 확인용이지 가족이 쓰는 스마트홈 운영판이 아니다.

정리하면 운영 `/config` 복제, 별도 포트의 테스트 컨테이너, 자동화 비활성화, 로그 확인 순서로 검증하면 된다. Home Assistant 2026.7처럼 자동화 편집기와 Matter 쪽 변화가 예상되는 업데이트는 운영 서버에서 바로 검증하지 않는 게 시간을 아낀다.

## 참고한 공식 자료

- [Home Assistant 2026.6 릴리스 노트](https://www.home-assistant.io/blog/2026/06/03/release-20266/)
- [Home Assistant 릴리스 일정 FAQ](https://www.home-assistant.io/faq/release/)
- [Home Assistant 2026.7 베타 주간 공지](https://community.home-assistant.io/t/2026-7-beta-week/1015047)

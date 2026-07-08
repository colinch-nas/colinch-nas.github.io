---
layout: post
title: "Home Assistant 2026.7 Activity - NAS Docker 자동화 오류를 타임라인으로 찾기"
description: "Home Assistant 2026.7 Activity 타임라인을 NAS Docker 환경에서 자동화 오류, 센서 누락, 사용자 조작 원인 추적에 쓰는 설정 순서."
date: 2026-07-08
tags: [HomeAssistant, Docker, 스마트홈, 홈오토메이션, NAS설정]
comments: true
share: true
---

![Home Assistant Activity 로그 확인 화면](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)
이 그림에서는 NAS에서 돌아가는 스마트홈 서버도 결국 로그를 시간순으로 읽어야 원인을 좁힐 수 있다는 점을 보면 된다.

Home Assistant 2026.7로 올렸다면 자동화가 안 됐을 때 YAML부터 열지 말고 Activity(기존 Logbook) 타임라인을 봐야 한다. 2026년 7월 1일 공식 릴리스 기준으로 Activity가 날짜별 타임라인으로 바뀌었고, 사람 조작, 자동화 실행, 통합 이벤트의 원인을 한 화면에서 보기 쉬워졌다. NAS Docker 환경에서는 컨테이너 로그와 Home Assistant 로그를 같이 봐야 하니 이 변화가 꽤 실용적이다.

내 기준 환경은 Synology DS923+, Container Manager, Home Assistant Container 2026.7.1, Zigbee2MQTT 1.39.x, Mosquitto 2.x다. 처음엔 조명이 안 켜지면 Zigbee2MQTT부터 의심했다. 그런데 실제로는 가족이 대시보드에서 수동으로 꺼둔 상태를 자동화 오류로 착각한 적이 있었다. Activity에서 사용자 아바타와 자동화 실행 원인을 나눠 보니 원인이 바로 갈렸다.

## Activity에서 볼 순서

| 증상 | Activity에서 볼 항목 | 같이 확인할 곳 |
|---|---|---|
| 조명이 안 켜짐 | 자동화가 실행됐는지 | Zigbee2MQTT 장치 이벤트 |
| 센서가 늦게 반응 | 센서 상태 변경 시각 | MQTT 브로커 로그 |
| 갑자기 꺼짐 | 사용자 조작인지 자동화인지 | 대시보드 사용자 기록 |
| 업데이트 뒤 오류 | 통합 이벤트와 unavailable 시각 | Docker 컨테이너 재시작 시각 |

문제 재현 시간을 정해두고 보는 게 핵심이다. 예를 들어 현관 센서가 21:34에 열렸는데 조명이 안 켜졌다면 Activity에서 `21:34` 주변만 본다. 전체 로그를 위에서부터 읽으면 금방 길을 잃는다. 여기서 자동화 실행 기록이 없으면 트리거 문제고, 실행 기록은 있는데 조명이 안 바뀌면 액션 또는 장치 통신 문제다.

NAS Docker에서는 아래처럼 컨테이너 시간을 같이 확인한다. Activity 시간과 Docker 로그 시간이 5분 이상 어긋나면 원인 추적이 꼬인다.

```bash
docker logs --since "2026-07-08T21:30:00" homeassistant
docker logs --since "2026-07-08T21:30:00" zigbee2mqtt
```

나는 여기서 한 번 삽질했다. NAS는 NTP가 맞았는데 Home Assistant 컨테이너의 시간대가 UTC로 보여서 Activity의 저녁 9시 로그와 Docker의 낮 12시 로그를 다른 사건으로 봤다. Compose에 시간대를 명시하고 다시 맞췄다.

```yaml
services:
  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:2026.7.1
    environment:
      - TZ=Asia/Seoul
    volumes:
      - /volume1/docker/homeassistant/config:/config
```

## 내가 남기는 점검 메모

- Activity에서 사람 조작, 자동화, 통합 이벤트를 구분한다.
- 같은 시각의 `homeassistant`, `zigbee2mqtt`, `mosquitto` 로그만 좁게 본다.
- `unavailable`과 자동화 실행 실패 중 어느 쪽이 앞인지 순서를 적는다.
- 업데이트 당일에는 HACS 카드, ESPHome 펌웨어, Core 이미지를 한꺼번에 올리지 않는다.

Activity는 백업 기능도, 장애 복구 버튼도 아니다. 대신 "누가, 무엇이, 몇 시에 상태를 바꿨는지"를 빠르게 좁혀준다. NAS에서 Home Assistant를 컨테이너로 운영한다면 이 화면을 사건 시간표로 쓰고, 실제 원인은 Docker 로그와 MQTT 로그에서 확인하는 방식이 덜 헤맨다.

확인 출처: [Home Assistant 2026.7 공식 릴리스 노트](https://www.home-assistant.io/blog/2026/07/01/release-20267/)

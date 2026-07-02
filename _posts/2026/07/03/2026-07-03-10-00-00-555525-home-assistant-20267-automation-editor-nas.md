---
layout: post
title: "Home Assistant 2026.7 자동화 편집기 - NAS Docker에서 새 자동화만 안전하게 적용하기"
description: "Home Assistant 2026.7 자동화 편집기 변경을 NAS Docker 운영 환경에서 기존 YAML을 깨지 않고 방 단위 자동화로 검증하는 절차."
date: 2026-07-03
tags: [HomeAssistant, 스마트홈, 홈오토메이션, Docker, NAS설정]
comments: true
share: true
---
![스마트홈 자동화와 NAS 서버를 함께 관리하는 홈서버 책상](https://images.unsplash.com/photo-1558002038-1055907df827?w=800&q=80)

Home Assistant 2026.7로 올렸다면 기존 자동화를 한꺼번에 고치지 말고, 새 자동화 편집기로 만든 항목만 따로 검증하면 된다. 2026년 7월 1일 공식 릴리스 노트 기준으로 목적별 트리거와 조건이 Labs에서 빠져나와 기본 편집 방식이 됐다. NAS Docker 운영자는 Core 업데이트보다 자동화가 실제 조명과 센서를 건드리는 순간을 더 조심해야 한다.

## 왜 NAS Docker에서 더 조심해야 하나

Home Assistant OS는 백업, 애드온, 복원이 한 화면에 묶여 있다. Synology나 QNAP Docker에서는 `/config` 폴더, compose 파일, Matter Server, Zigbee2MQTT를 따로 본다. UI에서 자동화 하나를 잘못 저장해도 컨테이너는 정상으로 보이니 원인을 늦게 찾는다.

내 경우 예전에는 `state` 트리거로 현관 센서의 `on` 값을 직접 봤다. 센서가 잠깐 `unavailable`이 되면 밤 조명이 안 켜졌고, 로그에는 별다른 오류가 없었다. 2026.7의 새 편집기는 "현관에 움직임 감지"처럼 목적 기준으로 시작할 수 있어 이런 삽질이 줄어든다.

## 적용 전 백업을 숫자 태그처럼 남긴다

자동화 편집기를 만지기 전에 `/config`를 통째로 복사한다. UI 백업만 믿지 말고 NAS 공유 폴더 밖에 한 벌 더 둔다.

```bash
cd /volume1/docker/homeassistant
docker compose stop homeassistant
tar -czf /volume1/backup/ha-config-2026-07-03-before-automation.tgz config
docker compose up -d homeassistant
```

Core 이미지는 `stable` 대신 실제 버전으로 고정해둔다.

```yaml
services:
  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:2026.7.0
    network_mode: host
    volumes:
      - ./config:/config
```

## 새 자동화는 방 단위로 하나만 만든다

Settings -> Automations & scenes -> Create automation으로 들어가서 기존 YAML 자동화를 수정하지 않는다. 테스트용으로 "서재 움직임이 감지되면 책상 조명을 5분 켜기" 같은 작은 자동화 하나를 만든다. 핵심은 개별 `binary_sensor.study_motion_01`이 아니라 `서재` 영역을 기준으로 잡는 것이다.

```text
Trigger: Motion detected in Study
Condition: After sunset
Action: Turn on Study desk light
Then: Wait 5 minutes
Action: Turn off Study desk light
```

저장 후 Activity(기존 Logbook)에서 원인을 확인한다. 2026.7에서는 활동 기록이 타임라인처럼 바뀌어서 자동화가 사람, 센서, 통합 중 무엇 때문에 실행됐는지 보기 쉽다. 여기서 한 번도 실행되지 않으면 자동화보다 영역 배치부터 본다.

## 기존 YAML은 남겨둔다

새 편집기가 편해졌다고 YAML을 바로 지우면 안 된다. 템플릿 조건, 전력 요금 계산, 여러 방을 묶는 자동화는 아직 YAML이 더 명확한 경우가 있다. 나는 새 UI 자동화 이름 앞에 `[2026.7]`을 붙이고 일주일 동안 같은 역할의 기존 자동화를 비활성화만 해둔다.

주의할 점은 세 가지다. 기존 자동화와 새 자동화가 같은 조명을 동시에 제어하지 않게 한다. HACS 커스텀 통합이 만든 센서는 목적별 트리거가 바로 보이지 않을 수 있다. Matter나 Zigbee 기기 이름을 바꾸면 영역 자동화가 따라오더라도 대시보드 카드 이름은 따로 남을 수 있다.

짧게 정리하면 이렇다. `/config` 백업부터 만든다. Core 이미지를 `2026.7.0`처럼 고정한다. 새 자동화 편집기는 방 단위 테스트 자동화 하나에만 쓴다. Activity에서 실행 원인을 확인한 뒤 기존 YAML을 천천히 정리한다.

참고:
- [Home Assistant 2026.7 공식 릴리스 노트](https://www.home-assistant.io/blog/2026/07/01/release-20267/)

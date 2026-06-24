---
layout: post
title: "Home Assistant 조명·전력 자동화 — 스마트 플러그와 모션 센서로 자동화 만들기"
description: "Home Assistant에서 스마트 플러그와 모션 센서를 활용해 조명 자동화와 대기전력 차단 자동화를 만드는 방법. YAML 자동화 예시 코드 포함."
date: 2026-06-12
tags: [HomeAssistant, 홈오토메이션, 스마트홈, 에너지모니터링]
comments: true
share: true
---

# Home Assistant 조명·전력 자동화 — 스마트 플러그와 모션 센서로 자동화 만들기

![Home Assistant 스마트홈 자동화](https://images.unsplash.com/photo-1558618047-3c8c76ca7d67?w=800&q=80)

Home Assistant의 진짜 재미는 자동화다. 기기를 앱으로 켜고 끄는 건 그냥 리모컨 하나 추가된 거고, 자동화를 만들면 집이 알아서 돌아간다. 모션 감지해서 조명 켜고, 외출하면 플러그 끄고, 자기 전에 에어컨 예약하는 식이다.

이전 글에서 [Zigbee2MQTT로 Zigbee 기기를 연결]({% post_url 2026-06-11-10-00-00-086415-zigbee2mqtt-setup-no-hub %})했다. 이번엔 그 기기들로 실제 자동화를 만든다.

## 환경 정보

| 항목 | 버전/모델 |
|------|----------|
| Home Assistant | 2026.5.x (HAOS) |
| 스마트 플러그 | SONOFF S31 Lite (Zigbee) |
| 모션 센서 | Aqara Motion Sensor P1 |
| 조명 | Philips Hue White (Zigbee) |

## 자동화 기본 구조

Home Assistant 자동화는 세 부분으로 이루어진다:

- **트리거(Trigger)**: 자동화가 시작되는 조건. 모션 감지, 특정 시간, 기기 상태 변화 등.
- **조건(Condition)**: 자동화가 실행되기 위해 추가로 충족해야 하는 조건. 없어도 됨.
- **액션(Action)**: 실제로 실행될 동작. 기기 켜기/끄기, 알림 보내기 등.

## 자동화 1: 모션 감지 시 조명 켜기

가장 기본적인 자동화. 사람이 감지되면 조명 켜고, 일정 시간 후 끈다.

**UI로 만들기**: 설정 → 자동화 및 장면 → 자동화 만들기

**YAML로 직접 작성**:

```yaml
alias: "거실 모션 감지 조명 자동화"
description: "사람 감지되면 조명 켜고, 5분 동안 움직임 없으면 끈다"
trigger:
  - platform: state
    entity_id: binary_sensor.motion_sensor_p1_occupancy
    to: "on"
action:
  - service: light.turn_on
    target:
      entity_id: light.living_room
    data:
      brightness_pct: 80
      color_temp: 4000    # 켈빈, 4000K = 중간 색온도
mode: single
```

5분 후 자동 끄기:

```yaml
alias: "거실 조명 5분 후 자동 끄기"
trigger:
  - platform: state
    entity_id: binary_sensor.motion_sensor_p1_occupancy
    to: "off"
    for:
      minutes: 5
action:
  - service: light.turn_off
    target:
      entity_id: light.living_room
mode: single
```

## 자동화 2: 시간대별 조명 밝기 조절

저녁엔 조명이 어두워야 잠들기 좋다. 시간에 따라 밝기와 색온도를 자동으로 조절한다.

```yaml
alias: "저녁 조명 어둡게"
trigger:
  - platform: time
    at: "21:00:00"
action:
  - service: light.turn_on
    target:
      area_id: living_room
    data:
      brightness_pct: 30
      color_temp: 2700    # 2700K = 따뜻한 색온도
mode: single
```

## 자동화 3: 스마트 플러그로 대기전력 차단

TV 끄면 셋톱박스는 계속 대기 상태다. 스마트 플러그의 에너지 모니터링 기능으로 소비 전력을 체크해서 5W 이하면 끈다.

```yaml
alias: "TV 꺼지면 셋톱박스 전원 차단"
trigger:
  - platform: numeric_state
    entity_id: sensor.tv_plug_power
    below: 5
    for:
      minutes: 3
action:
  - service: switch.turn_off
    target:
      entity_id: switch.settopbox_plug
mode: single
```

![Home Assistant 에너지 모니터링 대시보드](https://images.unsplash.com/photo-1544197150-880d4212a0c4?w=800&q=80)

## 자동화 4: 외출 시 전체 기기 끄기

스마트폰 GPS로 집을 벗어났을 때 자동으로 모든 기기 끄기.

먼저 **설정 → 사람**에서 위치 추적 디바이스를 등록한다. HA Companion 앱을 스마트폰에 설치하면 GPS 위치를 Home Assistant로 보낸다.

```yaml
alias: "외출 시 전기 절약"
trigger:
  - platform: state
    entity_id: person.my_name
    to: "not_home"
    for:
      minutes: 5    # 5분 후에 외출로 판단 (오탐 방지)
action:
  - service: homeassistant.turn_off
    target:
      area_id:
        - living_room
        - bedroom
  - service: climate.turn_off
    target:
      entity_id: all
mode: single
```

## 삽질했던 부분

**자동화 루프**: 조명 켜는 자동화와 끄는 자동화가 서로 트리거해서 무한 루프에 빠진 적 있다. `mode: single`을 설정하거나 적절한 딜레이를 넣어서 방지해야 한다.

**모션 센서 오탐**: Aqara P1 모션 센서가 커튼 흔들림이나 고양이한테 반응해서 한밤중에 조명이 켜진 경우가 있다. 조건에 시간대를 추가해서 오전 7시~오후 11시 사이에만 동작하게 했다.

```yaml
condition:
  - condition: time
    after: "07:00:00"
    before: "23:00:00"
```

**SONOFF S31 에너지 데이터 단위**: 에너지 센서 단위가 기기마다 다를 수 있다. `sensor.plug_power`가 W 단위인지 kW 단위인지 확인하고 자동화 조건의 숫자를 맞춰야 한다.

## 한 줄 정리

트리거-조건-액션 구조만 이해하면 어떤 자동화든 만들 수 있다. 처음엔 단순한 것부터 시작해서 점점 복잡하게 발전시켜 나가는 게 낫다.

---
다음엔 [Home Assistant 대시보드 구성 — Lovelace UI로 내 집 현황 한눈에 보기]({% post_url 2026-06-13-10-00-00-111105-home-assistant-lovelace-dashboard %})를 다룬다.

**참고 링크**
- [Home Assistant 자동화 공식 문서](https://www.home-assistant.io/docs/automation/)
- [Home Assistant 서비스 목록](https://www.home-assistant.io/docs/scripts/service-calls/)
- [Aqara Motion Sensor P1 공식 페이지](https://www.aqara.com/en/product/motion-sensor-p1/)

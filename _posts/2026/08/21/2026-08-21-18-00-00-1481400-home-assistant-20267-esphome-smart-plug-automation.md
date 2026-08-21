---
layout: post
title: "Home Assistant 2026.7 스마트 플러그 자동화 - NAS 주변 대기전력 끊는 설정"
description: "Home Assistant 2026.7과 ESPHome 지원 스마트 플러그로 NAS 주변 장비의 대기전력을 측정하고, 일정 시간 유휴 상태일 때 자동으로 끄는 방법을 정리한다."
date: 2026-08-21
tags: [HomeAssistant, 스마트홈, 홈오토메이션, 에너지모니터링, NAS설정]
comments: true
share: true
---

![Home Assistant와 스마트 플러그로 NAS 주변 전력 관리](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

NAS 주변에서 전기를 계속 먹는 장비는 NAS 본체보다 USB 허브, 모니터, 프린터, 스피커인 경우가 많다. Home Assistant 2026.7과 전력 측정 스마트 플러그를 연결하면 사용량을 보면서 일정 시간 유휴 상태인 장비만 자동으로 끌 수 있다. 단, NAS·공유기·스위치처럼 꺼지면 네트워크가 끊기는 장비는 이 자동화 대상에서 빼야 한다.

## 이번 구성

2026년 8월 21일 기준 Home Assistant의 최신 정식 월간 Core 릴리스는 2026.7이다. 7월 버전부터 목적형 트리거와 조건을 자동화 편집기의 기본 흐름으로 사용할 수 있다. IoTorero가 7월부터 Works with Home Assistant에 합류하며 ESPHome 지원 스마트 플러그를 선보인 것도 이런 로컬 전력 관리 구성을 시작하기 좋은 변화다.

| 항목 | 기준 |
|---|---|
| 서버 | Synology NAS Docker 또는 HAOS VM |
| Home Assistant | Core 2026.7.x |
| 플러그 | 전력 측정·ESPHome 또는 Zigbee 지원 모델 |
| 자동화 대상 | 모니터, 프린터, 충전기처럼 꺼져도 되는 장비 |

Energy 대시보드는 전력(W) 센서만으로는 누적 사용량을 계산하지 못한다. 스마트 플러그가 kWh 센서를 제공하면 `Settings → Dashboards → Energy`에서 바로 추가하고, W 센서만 있다면 Integration (Riemann sum integral) 통합으로 kWh 센서를 만든다. 공식 문서 기준으로 개별 기기 사용량도 Energy 대시보드에 등록할 수 있다.

## 1. 기기와 엔티티 확인

스마트 플러그를 Home Assistant에 추가한 뒤 `Settings → Devices & services`에서 아래 두 엔티티를 확인한다.

```text
switch.smart_plug_monitor
sensor.smart_plug_monitor_power
```

전력 센서 단위가 `W`이고 상태 클래스가 `measurement`인지 확인한다. 자동화에서 5W 기준을 쓸 것이므로 플러그 자체 대기전력이 0.5W 안팎인지도 봤다. 측정값이 0W로 고정되면 전원 차단 자동화부터 만들지 말고 기기 통합을 다시 설정해야 한다.

## 2. 15분 유휴 상태면 끄기

아래 YAML은 새 자동화의 목적형 편집기 대신 YAML 모드에서 붙여 넣을 수 있는 예시다. 매일 새벽 3시에 전력이 5W 이하인 상태가 15분 이어졌을 때만 플러그를 끈다.

```yaml
alias: NAS 주변 모니터 대기전력 차단
description: 15분 동안 5W 이하이면 모니터 플러그를 끈다
triggers:
  - trigger: time
    at: "03:00:00"
conditions:
  - condition: numeric_state
    entity_id: sensor.smart_plug_monitor_power
    below: 5
    for: "00:15:00"
actions:
  - action: switch.turn_off
    target:
      entity_id: switch.smart_plug_monitor
mode: single
```

이 조건을 낮 시간에 적용하면 사용 중인 모니터가 꺼질 수 있다. 실제로는 `binary_sensor.office_occupancy`가 `off`인지 조건으로 추가하거나, 프린터처럼 사용 패턴이 확실한 장비부터 적용하는 편이 안전하다. 자동화 실행 후에는 `Settings → Automations & scenes → Traces`에서 조건 통과 여부를 확인한다.

## 3. NAS 전력은 별도로 기록한다

NAS 전원을 스마트 플러그로 끊는 방식은 쓰지 않았다. DSM 업데이트 중이거나 디스크가 쓰기 작업 중일 때 강제 차단될 수 있기 때문이다. NAS는 전력 측정 플러그에 연결하더라도 차단 자동화는 만들지 않고, `sensor.nas_power`를 Energy 대시보드의 개별 기기로만 등록한다. 공유기와 스위치도 같은 원칙으로 기록만 한다.

Home Assistant 공식 Energy 문서에는 기기별 그래프와 상위·하위 장치 관계 설정이 정리돼 있다. 회로 전체 계측기와 개별 플러그를 같이 등록할 때는 upstream device를 지정해야 사용량이 이중 집계되지 않는다.

## 짧은 점검표

- 플러그가 전력(W)과 누적 에너지(kWh)를 모두 제공하는가
- 자동화 대상이 꺼져도 되는 장비인가
- 5W 기준이 장비의 실제 대기전력보다 높은가
- 실행 추적에서 15분 조건이 의도대로 통과하는가
- NAS·공유기·스위치는 차단 대상에서 제외했는가

이번 설정의 핵심은 전기를 아끼는 것보다 잘못 끄지 않는 데 있다. Home Assistant 2026.7의 자동화 편집기는 쉬워졌지만, 네트워크를 맡은 장비와 단순 주변기기를 같은 규칙에 넣으면 장애 원인을 찾기 어려워진다.

참고 문서: [Home Assistant 2026.7 릴리스 노트](https://www.home-assistant.io/blog/2026/07/01/release-20267/), [Energy 대시보드 공식 문서](https://www.home-assistant.io/docs/energy/), [IoTorero Works with Home Assistant 합류 소식](https://www.home-assistant.io/blog/2026/07/09/iotorero-joins-works-with-home-assistant/)

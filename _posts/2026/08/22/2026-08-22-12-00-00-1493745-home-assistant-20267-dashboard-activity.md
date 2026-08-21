---
layout: post
title: "Home Assistant 2026.7 대시보드 구성 - NAS 전력과 장애를 한 화면에서 확인하기"
description: "Synology VMM의 Home Assistant 2026.7에서 NAS 전력, 온도, 컨테이너 상태를 한 화면에 모으는 대시보드 구성법과 Activity 타임라인 활용법을 정리한다."
date: 2026-08-22
tags: [HomeAssistant, Synology, 스마트홈, 홈서버, 에너지모니터링]
comments: true
share: true
---

![Home Assistant 2026.7로 NAS 상태를 확인하는 스마트홈 대시보드](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

Home Assistant 2026.7에서는 자동화 편집기뿐 아니라 Activity 화면도 타임라인 형태로 바뀌었다. Synology VMM에 Home Assistant OS를 올려 쓰는 환경이라면 대시보드 첫 화면에 NAS 전력, 온도, 저장공간, Uptime Kuma 상태만 모아도 장애를 꽤 빨리 알아챌 수 있다. 예쁜 카드보다 “지금 NAS가 정상인가?”를 10초 안에 판단하는 구성이 더 오래 쓸 만하다.

## 내가 사용한 환경

| 항목 | 값 |
|---|---|
| NAS | Synology DS923+ |
| 가상머신 | Synology VMM, Home Assistant OS |
| Home Assistant | 2026.7 |
| 전력 측정 | ESPHome 스마트 플러그 |
| 장애 감시 | Uptime Kuma 연동 센서 |

전력 센서 엔티티는 `sensor.nas_power`, NAS 온도는 `sensor.synology_temperature`, Uptime Kuma는 `binary_sensor.nas_http`라는 이름으로 맞췄다. 엔티티 이름이 다르면 YAML에서 이 세 부분만 바꾸면 된다.

## 대시보드에 넣을 카드

대시보드 편집 화면에서 새 뷰를 만들고 수동 카드(Manual card)를 추가한다. 아래 YAML은 한 화면에서 상태를 읽는 데 필요한 최소 구성이다.

```yaml
type: grid
columns: 2
square: false
cards:
  - type: gauge
    entity: sensor.nas_power
    name: NAS 소비전력
    unit: W
    min: 0
    max: 80
    severity:
      green: 0
      yellow: 45
      red: 65
  - type: tile
    entity: sensor.synology_temperature
    name: NAS 온도
    icon: mdi:thermometer
  - type: tile
    entity: binary_sensor.nas_http
    name: Uptime Kuma
    icon: mdi:heart-pulse
  - type: entities
    title: 홈서버 핵심 상태
    entities:
      - entity: sensor.synology_volume_1_volume_used
        name: 저장공간 사용률
      - entity: sensor.synology_download_speed
        name: 다운로드
      - entity: sensor.synology_upload_speed
        name: 업로드
      - entity: sensor.nas_power_energy_today
        name: 오늘 사용량
```

![NAS 대시보드에서 확인할 전력·온도·장애 상태](https://images.unsplash.com/photo-1451187580459-43490279c9b7?w=1000&q=80)

이 구성에서 봐야 할 것은 게이지의 현재 숫자보다 평소 범위에서 벗어난 변화다. 내 NAS는 컨테이너가 유휴 상태일 때 18~25W였는데, 썸네일 재생이나 색인 작업이 겹치면 50W를 넘었다. 그래서 65W를 즉시 장애 기준으로 잡지 않고 “확인할 값”으로만 표시했다. 모델과 디스크 수에 따라 기준은 달라진다.

## 2026.7 Activity 화면을 장애 기록으로 쓰기

Home Assistant 2026.7의 Activity는 단순 로그 목록보다 원인과 대상이 가까이 보이는 타임라인에 가깝다. `설정 → 활동`에서 NAS 전력 센서와 Uptime Kuma 엔티티만 필터링하면, 전력 급증 직전에 컨테이너가 재시작됐는지 확인하기 쉽다.

자동화도 하나 추가했다. Uptime Kuma가 2분 이상 오프라인이고 NAS 전력이 10W 아래로 떨어지면 알림을 보낸다. 전원 차단인지 네트워크 문제인지 구분하려고 전력 조건을 함께 넣었다.

```yaml
alias: NAS 오프라인 알림
description: 전원 차단과 일시적인 HTTP 오류를 구분해 알림
triggers:
  - trigger: state
    entity_id: binary_sensor.nas_http
    to: "off"
    for: "00:02:00"
conditions: []
actions:
  - action: notify.mobile_app_my_phone
    data:
      title: NAS 접속 확인
      message: >-
        Uptime Kuma에서 NAS가 2분간 오프라인이다.
        현재 전력은 {{ states('sensor.nas_power') }}W다.
mode: single
```

`notify.mobile_app_my_phone`는 각자의 모바일 앱 알림 서비스명으로 바꿔야 한다. 이 자동화만 믿고 백업을 생략하면 안 된다. 알림 서버와 NAS가 같은 장비에 있다면 NAS가 완전히 꺼질 때 알림도 같이 멈춘다.

## 적용 전 체크리스트

| 확인 항목 | 기준 |
|---|---|
| 전력 센서 | W 단위인지 확인 |
| 온도 센서 | 섭씨 단위와 업데이트 주기 확인 |
| Uptime Kuma | HA에서 `binary_sensor`로 노출 |
| 알림 | 휴대폰 앱 서비스명으로 교체 |
| 백업 | 자동화·대시보드 YAML을 별도 백업 |

핵심은 대시보드를 정보 게시판처럼 채우지 않는 것이다. NAS 전력·온도·외부 접속 상태 세 가지를 위에 두고, 저장공간과 트래픽을 아래에 배치하면 문제가 생겼을 때 원인을 좁히는 속도가 빨라진다.

---
layout: post
title: "Home Assistant 2026.8 엔티티 ID 정리 - NAS 자동화 깨지지 않게 이름 바꾸기"
description: "Home Assistant 2026.8에서 엔티티 ID를 UI로 직접 변경하고, Synology NAS 자동화와 대시보드가 깨졌을 때 복구하는 실전 절차를 정리한다."
date: 2026-08-29
tags: [HomeAssistant, Synology, NAS설정, 홈서버, 홈오토메이션]
comments: true
share: true
---

![Synology NAS에서 Home Assistant 엔티티 ID 정리](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

Home Assistant 2026.8에서는 엔티티 ID(entity_id, 자동화와 대시보드가 장치를 가리키는 고유 주소)를 화면에서 직접 바꿀 수 있다. `sensor.tz3000_xxxx_power` 같은 이름을 `sensor.nas_rack_power`로 정리하기 좋다. 다만 백업과 참조 위치를 확인해야 자동화가 멈추지 않는다.

## 확인한 환경

| 항목 | 기준 |
|---|---|
| NAS | Synology DS923+ |
| NAS OS | DSM 7.2.2, Container Manager |
| Home Assistant | 2026.8.3 Container |
| 장치 | Zigbee2MQTT 전력 센서, NAS 유선망 |
| 변경 전 | `sensor.tz3000_ab12_power` |
| 변경 후 | `sensor.nas_rack_power` |

공식 릴리스 노트에 따르면 엔티티 ID의 이름과 순서를 사용자가 정할 수 있게 됐다. 장치 이름(Device name)만 바꾸는 것과 달리, ID를 바꾸면 YAML과 UI 자동화의 옛 주소도 확인해야 한다.

## 변경 전 백업과 참조 확인

NAS에서 Home Assistant 컨테이너를 운영 중이라면 **설정 → 시스템 → 백업**에서 전체 백업을 만든다. 백업 파일은 PC에도 내려받는다.

자동화 YAML에 옛 ID가 얼마나 남았는지는 설정 폴더에서 확인한다. 아래 명령은 컨테이너가 `/config`를 NAS의 `/volume1/docker/homeassistant/config`에 연결했다는 기준이다.

```bash
cd /volume1/docker/homeassistant/config
rg -n "tz3000_ab12_power" automations.yaml configuration.yaml scenes.yaml dashboards/ 2>/dev/null
```

대시보드를 UI에서만 만들었다면 이 검색에 잡히지 않을 수 있다. **설정 → 자동화와 장면**에서도 해당 센서를 조건·동작으로 사용하는지 확인한다.

## UI에서 엔티티 ID 변경

**설정 → 장치 및 서비스 → 엔티티**에서 센서를 검색하고 상세 화면을 연다. 엔티티 ID 옆의 편집 버튼을 누른 뒤 `sensor.nas_rack_power`처럼 입력한다. 공백과 한글은 쓰지 않고, 소문자와 숫자·밑줄만 사용한다.

| 점검 항목 | 예시 |
|---|---|
| 영역을 드러내는 이름 | `nas`, `livingroom`, `garage` |
| 역할을 드러내는 이름 | `power`, `temperature`, `disk_usage` |
| 피할 이름 | `sensor1`, `new_power`, 모델명만 사용 |

저장 후 기록 그래프가 이어지는지 확인한다. 장치를 다시 페어링하는 작업이 아니므로 Zigbee2MQTT 연결은 끊기지 않고, Home Assistant 내부 주소만 바뀐다.

## 자동화와 대시보드 검증

자동화 편집 화면에서 옛 엔티티가 `알 수 없음` 또는 빈 선택값으로 표시되면 새 ID를 다시 지정한다. YAML 자동화를 쓴다면 다음처럼 새 주소가 들어갔는지 확인한다.

```yaml
condition:
  - condition: numeric_state
    entity_id: sensor.nas_rack_power
    above: 150
```

대시보드 카드의 값과 개발자 도구의 **상태**에서 새 ID를 확인한다. NAS 전력 센서라면 값이 갱신되는지, 150W 조건의 테스트 자동화가 알림을 보내는지까지 본다.

## 이름 변경 후 문제가 생겼을 때

자동화가 실행되지 않으면 자동화의 **추적** 화면에서 멈춘 조건을 확인한다. 옛 ID가 남아 있으면 새 ID로 교체하고 구성 검사를 실행한다. 여러 개를 바꿀 때는 하루 2~3개씩 나누는 편이 원인을 찾기 쉽다.

체크리스트는 간단하다.

- 전체 백업 파일을 NAS 밖에도 보관했는가
- YAML 검색과 UI 자동화 참조를 모두 확인했는가
- 개발자 도구에서 새 ID의 상태가 갱신되는가
- 자동화 추적과 대시보드 카드를 실제로 시험했는가

Home Assistant 2026.8의 엔티티 ID 변경은 Zigbee 장치를 다시 연결하는 작업이 아니다. 백업 후 이름 규칙을 정하고, 하나씩 바꾸면서 자동화 추적까지 확인하면 NAS에 쌓인 난해한 모델명도 관리 가능한 주소로 정리할 수 있다.

출처: [Home Assistant 2026.8 공식 릴리스 노트](https://www.home-assistant.io/blog/2026/08/05/release-20268/)

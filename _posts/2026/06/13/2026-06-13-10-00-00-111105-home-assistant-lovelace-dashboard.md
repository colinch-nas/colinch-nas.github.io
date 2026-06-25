---
layout: post
title: "Home Assistant 대시보드 구성 — Lovelace UI로 내 집 현황 한눈에 보기"
description: "Home Assistant Lovelace 대시보드를 커스터마이즈해서 조명, 온습도, 에너지 소비, 보안 카메라를 한 화면에 표시하는 방법. HACS와 커스텀 카드 설치 포함."
date: 2026-06-13
tags: [HomeAssistant, 스마트홈, 홈오토메이션]
comments: true
share: true
---
![Home Assistant Lovelace 대시보드](https://images.unsplash.com/photo-1558618047-3c8c76ca7d67?w=800&q=80)

Home Assistant를 설치하고 기기를 연결했으면 이제 대시보드를 꾸밀 차례다. 기본 UI만 써도 되는데, HACS(Home Assistant Community Store)를 통해 커스텀 카드를 설치하면 훨씬 보기 좋게 꾸밀 수 있다.

이 글은 Home Assistant 스마트홈 시리즈 4편이다. [조명·전력 자동화]({% post_url 2026-06-12-10-00-00-098760-home-assistant-lighting-power-automation %})까지 만들었다면 이제 대시보드로 한눈에 볼 수 있게 정리하자.

## 환경 정보

| 항목 | 버전/모델 |
|------|----------|
| Home Assistant | 2026.5.x (HAOS) |
| HACS | 2.x |
| 주요 커스텀 카드 | Mushroom, Mini Graph Card, Button Card |

## 기본 대시보드 편집

**개요** 탭의 우상단 연필 아이콘을 클릭하면 편집 모드가 된다.

카드 추가 버튼을 누르면 기본 카드 목록이 나온다. 주요 기본 카드:

- **Entities**: 여러 기기 상태를 목록으로 표시
- **Gauge**: 온도/습도 같은 수치를 게이지로 표시
- **Light**: 조명 제어 카드
- **Thermostat**: 에어컨/히터 제어
- **History Graph**: 시간별 상태 변화 그래프

기본 카드만으로도 충분하지만 디자인이 평범하다.

## HACS 설치

커스텀 카드를 쓰려면 HACS(Home Assistant Community Store)를 먼저 설치해야 한다.

```bash
# SSH 또는 터미널 애드온에서 실행
wget -O - https://get.hacs.xyz | bash -
```

설치 후 Home Assistant 재시작 → **설정 → 기기 및 서비스 → 통합** 에서 HACS를 추가.

GitHub 계정으로 인증이 필요하다. 화면에 나오는 코드를 github.com/login/device에 입력한다.

## 추천 커스텀 카드

### Mushroom Cards

깔끔한 Material Design 스타일 카드 모음. 기본 카드를 거의 대체할 수 있다.

HACS → 프론트엔드 → Mushroom 검색 설치.

```yaml
# 조명 카드 예시
type: custom:mushroom-light-card
entity: light.living_room
name: 거실 조명
show_brightness_control: true
show_color_temp_control: true
```

### Mini Graph Card

에너지 소비나 온습도 변화를 미니 그래프로 표시한다.

```yaml
type: custom:mini-graph-card
entities:
  - entity: sensor.living_room_temperature
    name: 거실 온도
  - entity: sensor.living_room_humidity
    name: 거실 습도
hours_to_show: 24
points_per_hour: 2
```

![Home Assistant 커스텀 대시보드](https://images.unsplash.com/photo-1451187580094-2cfb76b7c9b7?w=800&q=80)

### Button Card

버튼 하나로 여러 기기를 동시에 제어하는 씬(scene) 실행에 적합하다.

```yaml
type: custom:button-card
name: 영화 모드
icon: mdi:movie
tap_action:
  action: call-service
  service: scene.turn_on
  service_data:
    entity_id: scene.movie_mode
styles:
  card:
    - background-color: "#1a1a2e"
    - color: white
```

## 대시보드 레이아웃 예시

방별로 뷰를 나누는 게 정리하기 좋다.

```yaml
# 거실 뷰 구성 예시
title: 거실
path: living_room
icon: mdi:sofa
cards:
  - type: custom:mushroom-light-card
    entity: light.living_room
  - type: custom:mini-graph-card
    entities:
      - sensor.living_room_temperature
  - type: entities
    entities:
      - switch.tv_plug
      - switch.settopbox_plug
```

## 에너지 대시보드 설정

Home Assistant 2021.8부터 내장된 에너지 대시보드가 있다. **설정 → 에너지**에서 전력계 기기(스마트 플러그)와 태양광 패널(있다면)을 등록하면 가정 내 에너지 소비를 시각화해서 보여준다.

스마트 플러그에서 에너지 소비 데이터(kWh)를 제공하면 자동으로 연동된다.

## 삽질했던 부분

**YAML 오류**: 대시보드를 YAML로 직접 편집할 때 들여쓰기 오류가 있으면 저장이 안 된다. 오류 메시지가 친절하지 않아서 어디가 틀렸는지 찾기가 힘들다. VS Code의 YAML 확장을 써서 로컬에서 먼저 검증하고 붙여넣는 게 낫다.

**HACS 업데이트 후 카드가 깨짐**: HACS로 설치한 카드가 Home Assistant 업데이트 후 동작하지 않을 때가 있다. HACS → 프론트엔드 탭에서 업데이트 가능한 카드가 있는지 확인하면 된다.

**모바일 레이아웃**: PC에서 보기 좋게 만들었는데 모바일에서 카드 배치가 다 틀어진 경우가 있다. `grid-template-columns`나 `view_layout`을 써서 반응형으로 만들 수 있지만 설정이 복잡하다.

## 한 줄 정리

Mushroom Cards + Mini Graph Card 조합이면 Home Assistant 대시보드를 방별로 깔끔하게 정리할 수 있다.

---
다음엔 [Nextcloud 설치 — 구글 드라이브 대신 내 서버에 클라우드 만들기]({% post_url 2026-06-14-10-00-00-123450-nextcloud-install-self-hosted-cloud %})를 다룬다.

**참고 링크**
- [Home Assistant 대시보드 공식 문서](https://www.home-assistant.io/dashboards/)
- [HACS 공식 사이트](https://hacs.xyz)
- [Mushroom Cards GitHub](https://github.com/piitaya/lovelace-mushroom)
- [Mini Graph Card GitHub](https://github.com/kalkih/mini-graph-card)

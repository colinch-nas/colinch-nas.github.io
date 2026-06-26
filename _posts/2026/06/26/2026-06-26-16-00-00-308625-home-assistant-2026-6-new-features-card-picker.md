---
layout: post
title: "Home Assistant 2026.6 — 대시보드 카드 피커 전면 개편과 달라진 것들"
description: "2026년 6월 출시된 Home Assistant 2026.6 주요 변경점 정리. 카드 추가 UI 전면 개편, 타일 카드 날씨·미디어 업그레이드, IR 양방향 수신, Thread 끊김 해결까지."
date: 2026-06-26
tags: [HomeAssistant, 스마트홈, 홈오토메이션, 대시보드, Matter]
comments: true
share: true
---

# Home Assistant 2026.6 — 대시보드 카드 피커 전면 개편과 달라진 것들

![Home Assistant 스마트홈 자동화 대시보드](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=800&q=80)

Home Assistant 2026.6이 6월 3일 출시됐다. 이번 릴리즈 테마는 "Pick a card, any card" — 이름 그대로 대시보드 카드 추가 UI가 뿌리부터 바뀌었다. 기존 사용자라면 업데이트 직후 카드 추가 버튼을 눌렀다가 "이게 뭐야" 할 수 있다.

## 환경 정보

| 항목 | 버전 |
|------|------|
| Home Assistant Core | 2026.6.0 |
| Home Assistant OS | 15.x |
| ESPHome | 2026.6.0 |

## 카드 피커 — 완전히 바뀐 진입점

기존 카드 추가 다이얼로그는 카드 타입 이름이 쭉 나열된 구조였다. Tile, Entities, Button, Gauge... 이걸 처음 보는 사람은 뭘 골라야 할지 전혀 모른다. 쓰던 사람도 "타일이었나 엔티티였나" 기억을 더듬어야 했다.

2026.6부터는 기본 화면이 **"By entity" 탭**으로 바뀌었다.

화면 왼쪽에 집 구조가 표시된다 — 층 → Area → 방 → 기기. 여기서 엔티티를 고르면 HA가 그 엔티티에 맞는 카드를 자동 추천해준다. 조명이면 타일 카드, 날씨면 날씨 카드, 미디어 플레이어면 미디어 타일.

기존처럼 카드 타입을 직접 고르는 탭도 그대로 있다. 이제 그게 기본이 아닐 뿐이다.

처음 며칠은 위치 찾느라 어색했는데, 신규 사용자한테는 이게 훨씬 낫겠다 싶다. [대시보드 카드 구성 기초는 이 글 참고]({% post_url 2026-06-13-10-00-00-111105-home-assistant-lovelace-dashboard %}).

## 타일 카드 — 날씨 엔티티 전용 기능 추가

이번 버전에서 날씨 엔티티를 타일 카드에 붙였을 때 새 기능 두 개가 생겼다.

**온도 예보 바 차트**: 일별 최고·최저 기온을 cyan → red 그라데이션 막대로 시각화. 시간별 뷰로 전환하면 곡선 형태로 렌더링된다.

**강수 예보**: 예상 강우량 또는 강수 확률을 함께 표시.

미디어 플레이어 타일도 컨트롤이 늘었다. 재생/일시정지만 있던 게 볼륨 조절, 다음 트랙, 셔플, 반복 재생까지 타일 하나에서 가능해졌다. 별도 팝업 없이 대시보드에서 바로 조작된다.

## IR 양방향 수신 — 드디어 됐다

2026.6 이전까지 HA의 IR 지원은 단방향이었다. HA가 IR 신호를 보내서 TV, 에어컨을 제어하는 건 됐는데, 사람이 리모컨으로 직접 조작하면 HA 상태가 싱크를 잃었다.

예를 들어 에어컨을 리모컨으로 껐는데 HA 대시보드에는 "켜짐"으로 남는 것.

**2026.6부터는 IR 신호 수신이 가능하다.**

ESP32 또는 ESP8266에 IR 수신기(TSOP38238 같은 것)를 달면, ESPHome을 통해 HA가 리모컨 버튼 입력을 감지한다.

```yaml
# ESPHome 2026.6.0 기준 — IR 수신기 설정
remote_receiver:
  pin: GPIO14
  dump: all  # 학습 단계에서 버튼 ID 확인용
```

이걸로 할 수 있는 것:
- 리모컨으로 끈 에어컨을 HA가 인식해 상태 자동 동기화
- 서랍 속 구형 리모컨을 HA 자동화 트리거 버튼으로 재활용
- 음악 재생 중인 스피커 물리 버튼 → HA 반응

이 기능 때문에 ESP32 IR 브릿지 설치를 미루고 있었는데, 이번에 다시 꺼내볼 생각이다.

![ESP32 IR 수신기 홈 자동화 연동](https://images.unsplash.com/photo-1518770660439-4636190af475?w=800&q=80)

## 자동화 에디터 — 타겟 엔티티 수 표시

Area(영역) 전체나 Label을 자동화 타겟으로 지정하면, 실제로 몇 개 엔티티가 걸리는지 숫자로 바로 보인다.

"거실 조명 전체" 조건을 걸었는데 엔티티 1개만 매핑돼 있다면 즉시 확인 가능하다. 설정이 의도대로 됐는지 검증하는 데 편해졌다.

## Thread 기기 끊김 해결

OpenThread Border Router가 1.4 안정 버전으로 올라갔다. 내장 mDNS 구현이 들어가면서 Thread 기기가 가끔 응답 없어지는 현상이 줄었다는 보고가 많다.

Matter+Thread 기기(Nanoleaf, Eve, 일부 필립스 Hue 등)를 쓰는데 대시보드에서 가끔 "사용 불가"로 뜨는 현상을 겪었다면 이번 업데이트 후 체감 차이가 있을 수 있다.

## 업데이트 방법

Home Assistant OS 사용 중이라면:

```
설정 → 시스템 → 업데이트 → Home Assistant Core 2026.6.x 설치
```

Docker Compose로 운영 중이라면:

```bash
docker pull ghcr.io/home-assistant/home-assistant:stable
docker compose pull && docker compose up -d
```

대시보드 카드 피커 변경은 업데이트 즉시 적용된다. 기존에 만들어둔 대시보드는 그대로 유지된다.

## 한 줄 정리

2026.6은 신규 사용자 진입장벽을 낮추는 방향의 업데이트다. 오래 쓴 사람에겐 카드 피커 변경이 처음엔 어색하지만, IR 양방향이나 Thread 1.4 안정화처럼 실질적으로 오래 기다리던 버그 픽스가 같이 들어왔다.

---

다음엔 ESP32 + ESPHome으로 IR 학습 후 에어컨 상태를 HA에 자동 동기화하는 방법을 다룬다.

**참고 링크**
- [Home Assistant 2026.6 공식 릴리즈 노트](https://www.home-assistant.io/blog/2026/06/03/release-20266/)
- [ESPHome 2026.6.0 변경 사항](https://esphome.io/changelog/2026.6.0/)
- [OpenThread Border Router 1.4](https://openthread.io/)

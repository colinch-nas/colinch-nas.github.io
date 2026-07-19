---
layout: post
title: "Home Assistant 2026.7 YAML 자동화 오류 - NAS Docker에서 바뀐 트리거 키 점검"
description: "Home Assistant Core 2026.7.2에서 이름이 바뀐 YAML 트리거·조건 키를 NAS Docker 운영자가 찾고 수정하는 방법. 기존 자동화가 멈추는 조건과 복구 순서를 정리한다."
date: 2026-07-20
tags: [HomeAssistant, Docker, NAS설정, 홈서버, 스마트홈]
comments: true
share: true
---

![Home Assistant NAS 자동화 YAML 점검](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

Home Assistant Core 2026.7.2로 올린 뒤 자동화가 갑자기 실행되지 않는다면 장치 연결보다 YAML 키 변경을 먼저 확인하는 게 빠르다. 2026.7에서 목적형 트리거와 조건이 정식 기본값이 되면서 일부 이름이 더 자연스러운 표현으로 바뀌었다. 기존 자동화 전체가 깨지는 것은 아니지만, 해당 키를 직접 사용한 YAML은 수정이 필요하다.

## 무엇이 바뀌었나

공식 변경사항에서 실제로 확인할 항목은 아래와 같다.

| 이전 키 | 새 키 | 주로 쓰는 자동화 |
|---|---|---|
| `battery.low` | `battery.became_low` | 배터리 부족 알림 |
| `battery.not_low` | `battery.no_longer_low` | 배터리 교체 후 알림 해제 |
| `update.update_became_available` | `update.became_available` | 업데이트 알림 |
| `vacuum.docked` | `vacuum.returned_to_dock` | 로봇청소기 복귀 알림 |

그림에서 볼 부분은 버전 숫자보다 기존 YAML의 키가 새 이름으로 바뀌었는지다.

## NAS Docker에서 먼저 백업하기

Synology Container Manager에서 Home Assistant 컨테이너를 바로 편집하지 말고, compose 볼륨의 설정 폴더를 복사한다. 아래 경로는 내 NAS의 예시다.

```bash
cd /volume1/docker/homeassistant
cp -a config config.backup-2026-07-20
```

컨테이너 안에서 설정 파일을 관리하는 구성이라면 호스트 경로가 다르다. `docker inspect homeassistant`의 `Mounts`에서 `/config`에 연결된 실제 경로를 확인해야 한다. 이걸 놓치고 컨테이너 내부 파일만 고치면 재생성 때 수정 내용이 사라진다.

## 바뀐 키를 찾아 한 번에 점검하기

설정 폴더에서 예전 키를 검색한다. 명령어 위의 목적은 자동화 파일을 수정하기 전에 영향 범위를 세는 것이다.

```bash
cd /volume1/docker/homeassistant/config
rg -n "battery\.low|battery\.not_low|update\.update_became_available|vacuum\.docked|timer\.time_remaining" \
  automations.yaml scripts.yaml scenes.yaml
```

예를 들어 자동화 YAML 안에 아래처럼 예전 키가 남아 있다면:

```yaml
trigger: battery.low
```

2026.7 기준으로는 키만 다음처럼 바꾼다.

```yaml
trigger: battery.became_low
```

다만 모든 자동화를 기계적으로 바꿀 필요는 없다. UI에서 만든 장치 트리거가 내부적으로 새 형식을 사용하고 있다면 이미 정상이다. 검색 결과를 보고 예전 문자열을 직접 작성한 항목만 수정한다. UI 편집기에서 저장한 자동화는 YAML 구조가 달라질 수 있으므로, 화면에서 해당 트리거를 다시 선택하고 저장하는 편이 안전하다.

## 검증과 실패 지점

수정 후에는 **설정 → 시스템 → YAML 구성 확인**에서 검증하고, 오류가 없을 때만 **자동화 다시 불러오기**를 실행한다. NAS Docker에서 자주 헷갈렸던 부분은 컨테이너 재시작이 필요하다고 생각하는 것이다. YAML 문법과 자동화 정의만 바꿨다면 전체 재시작보다 reload가 먼저다. reload 뒤에도 실행되지 않으면 **설정 → 자동화 및 장면 → 해당 자동화 → 추적**에서 트리거가 발생했는지 확인한다. 배터리 센서가 `unknown`인 상태에서 `became_low`를 기다리는 경우처럼, 키를 고쳐도 센서 값이 정상화되지 않으면 알림은 나오지 않는다.

## 짧은 점검표

- Home Assistant Core가 2026.7.2인지 확인한다.
- `/config` 호스트 경로를 백업한다.
- 예전 키를 `rg`로 검색한다.
- 설정 확인 후 자동화만 reload한다.
- 배터리·청소기처럼 실제 이벤트를 한 번 발생시켜 추적 화면을 본다.

Home Assistant 2026.7의 변경은 대규모 마이그레이션보다 이름 정리에 가깝다. NAS 운영자는 업데이트 직후 모든 자동화를 다시 만드는 대신, 예전 키를 검색하고 실제로 쓰는 항목만 고치는 방식이 가장 안전하다.

- [Home Assistant 2026.7 릴리스 노트](https://www.home-assistant.io/blog/2026/07/01/release-20267/)

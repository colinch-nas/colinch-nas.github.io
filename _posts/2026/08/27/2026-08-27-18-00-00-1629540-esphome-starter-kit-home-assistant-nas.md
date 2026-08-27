---
layout: post
title: "ESPHome Starter Kit과 Home Assistant 2026.8 연결 - NAS에서 온도 센서 자동화 만들기"
description: "Home Assistant 2026.8을 Synology NAS에서 운영하면서 ESPHome Starter Kit을 연결하는 방법을 정리한다. USB 없이 Wi-Fi 페어링하고 온도 변화 알림까지 설정한다."
date: 2026-08-27
tags: [HomeAssistant, ESPHome, Synology, Docker, 스마트홈]
comments: true
share: true
---

![ESPHome Starter Kit과 NAS 기반 Home Assistant 스마트홈](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

Home Assistant 2026.8을 Synology NAS에서 돌리고 있다면 ESPHome Starter Kit을 별도 허브 없이 붙일 수 있다. 2026년 8월 12일 공식 출시 안내가 나온 키트라서, 이번에는 Wi-Fi 센서 하나를 등록하고 온도가 바뀔 때 알림을 보내는 데까지 구성해봤다. 핵심은 ESPHome 장치를 NAS 컨테이너와 같은 네트워크에서 찾게 하는 것이다.

## 준비한 환경

| 항목 | 기준 |
|---|---|
| NAS | Synology DS923+ |
| NAS OS | DSM 7.2.2, Container Manager |
| Home Assistant | 2026.8.x Container 버전 |
| 장치 | ESPHome Starter Kit, 2.4GHz Wi-Fi |
| 네트워크 | NAS와 장치를 같은 공유기 내부망에 연결 |

ESPHome은 펌웨어를 올린 장치가 Home Assistant와 직접 통신하는 방식이다. Zigbee처럼 코디네이터 USB 동글을 NAS에 꽂지 않아도 되는 점이 편하다. 반대로 Wi-Fi가 끊기면 센서도 함께 사라지므로 공유기와 NAS의 위치가 더 중요해진다.

## 1. Home Assistant에 ESPHome 연동 추가

Home Assistant 화면에서 **설정 → 기기 및 서비스 → 통합 추가**를 열고 `ESPHome`을 검색한다. 설치된 통합을 찾지 못하면 컨테이너가 정상 실행 중인지, 브라우저에서 `http://NAS-IP:8123`으로 접속되는지부터 확인한다.

Starter Kit을 켜고 초기 설정에서 집 Wi-Fi의 2.4GHz SSID와 비밀번호를 입력한다. 5GHz 전용 SSID만 있는 환경에서는 검색되지 않는다. 장치가 부팅된 뒤 ESPHome 통합이 자동으로 발견하면 IP 주소를 확인하고 추가한다.

발견되지 않을 때는 장치 화면에 표시된 IP를 직접 입력한다. 이때 NAS의 방화벽에서 Home Assistant 컨테이너가 내부 네트워크에 접근할 수 있어야 한다. Container Manager에서 네트워크를 `host`로 바꾸는 방법도 있지만, 기존 포트와 충돌할 수 있어 우선 기본 브리지 네트워크를 유지하고 공유기의 DHCP 할당 목록에서 장치 IP를 확인하는 편이 안전했다.

## 2. 장치 이름과 영역 정리

연동 직후에는 `ESPHome Device`처럼 애매한 이름으로 보인다. 기기 페이지에서 이름을 `거실 환경 센서`로 바꾸고 영역을 `거실`로 지정한다. 이 작업을 미루면 자동화를 만들 때 같은 이름의 센서가 늘어나서 잘못된 엔티티를 고르기 쉽다.

`센서 → 온도` 엔티티의 단위가 섭씨(°C)인지도 확인한다. 지역 설정이 화씨로 잡혔다면 **설정 → 시스템 → 일반**의 단위 체계를 미터법으로 바꾼다.

## 3. 온도 상승 알림 자동화 만들기

아래 YAML은 거실 온도가 28°C를 넘고 5분간 유지될 때 모바일 알림을 보내는 예시다. 엔티티 ID는 기기 이름에 따라 달라지므로 화면에서 실제 값을 복사해야 한다.

```yaml
alias: 거실 온도 상승 알림
description: 28도 이상이 5분 유지되면 알림
triggers:
  - trigger: numeric_state
    entity_id: sensor.geosil_hwan_gyeong_sensor_temperature
    above: 28
    for: "00:05:00"
conditions: []
actions:
  - action: notify.mobile_app_my_phone
    data:
      title: 거실 온도 확인
      message: "현재 온도는 {{ states('sensor.geosil_hwan_gyeong_sensor_temperature') }}°C다."
mode: single
```

자동화 편집기의 YAML 모드에 붙여 넣은 뒤 `notify.mobile_app_my_phone`을 본인 휴대폰 서비스로 바꾼다. 처음부터 30°C처럼 높은 값을 쓰지 않고 28°C로 테스트한 이유는 실제 센서 값이 올라가는지 즉시 확인하기 위해서다.

## 막혔던 지점과 운영 체크

가장 많이 헷갈린 부분은 장치가 켜졌는데도 통합 목록에 나타나지 않는 경우였다. 공유기에서 게스트 Wi-Fi를 쓰고 있으면 NAS와 장치가 서로 다른 VLAN에 들어가 mDNS(로컬 네트워크 장치 검색)가 차단될 수 있다. 두 장치를 같은 일반 Wi-Fi에 잠시 연결해 등록한 뒤, 필요하면 방화벽 규칙을 별도로 나누는 편이 낫다.

- 2.4GHz SSID가 살아 있는가
- NAS와 ESPHome 장치가 같은 내부망인가
- 장치에 DHCP 예약을 걸었는가
- Home Assistant 백업에 자동화 YAML이 포함되는가
- 펌웨어 업데이트 전 현재 설정을 내보냈는가

ESPHome Starter Kit은 Home Assistant 2026.8에서 센서 확장용으로 시작하기 좋다. NAS는 장치 데이터를 저장하고 자동화를 실행하는 중심 서버가 되며, 장치 자체는 Wi-Fi만 안정적이면 된다. 다만 인터넷이 끊겨도 자동화를 유지하려면 Home Assistant와 장치 모두 같은 집 안 네트워크에서 통신하도록 구성해야 한다.

참고한 공식 문서:

- [Home Assistant 2026.8 릴리스 노트](https://www.home-assistant.io/blog/2026/08/05/release-20268/)
- [ESPHome Starter Kit 출시 안내](https://www.home-assistant.io/blog/2026/08/12/the-esphome-starter-kit-is-here/)
- [ESPHome 공식 문서](https://esphome.io/)

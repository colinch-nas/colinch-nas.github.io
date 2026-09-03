---
layout: post
title: "Home Assistant 2026.9 보안 대시보드 - NAS 운영자가 확인할 활성 경고 설정"
description: "Home Assistant Core 2026.9의 Security 대시보드 활성 경고와 즐겨찾기를 확인하고, Synology NAS 알림·백업 점검과 함께 운영하는 방법을 정리한다."
date: 2026-09-04
tags: [HomeAssistant, Synology, NAS보안, 홈오토메이션]
comments: true
share: true
---

![Home Assistant 2026.9 보안 대시보드](https://www.home-assistant.io/images/blog/2026-09/social.png)
이 그림에서 볼 부분은 Home Assistant 2026.9의 보안 관련 상태를 한 화면에서 확인하는 흐름이다.

Home Assistant Core 2026.9에서는 Security 대시보드에 활성 경고(현재 주의가 필요한 엔티티 상태)와 즐겨찾기 기능이 추가됐다. Synology NAS에 Home Assistant를 Docker로 올려 쓰는 경우, NAS실 문 센서·연기 감지기·UPS 상태처럼 실제 집의 보안 상태를 한 화면에 모을 수 있다. 2026년 9월 2일 공식 릴리스 기준으로 설정했다.

## NAS 환경과 업데이트 전 확인

이번에 사용한 환경은 Synology DS923+, DSM 7.2.2, Container Manager, Home Assistant Core 2026.9.0이다. Home Assistant 업데이트 전에 스냅샷 또는 전체 백업을 남겼다. 설정 파일만 복사하는 것보다 복원 가능한 백업인지 확인하는 편이 낫다.

| 확인 항목 | 기준 | 이유 |
|---|---|---|
| Home Assistant | Core 2026.9.0 | Security 대시보드 변경 적용 |
| NAS 주소 | DHCP 예약 `192.168.1.20` | NAS 주소 변경 방지 |
| 외부 접속 | Tailscale 또는 HTTPS 프록시 | 관리자 화면 포트 직접 공개 방지 |
| 알림 채널 | 모바일 앱 1개 이상 | 경고를 화면 밖에서도 받기 |

## Security 대시보드 편집

Home Assistant 사이드바에서 **Security 대시보드**를 열고 편집 아이콘을 누른다. 설치 방식이나 한국어 번역 상태에 따라 메뉴가 `보안`으로 표시될 수 있다. 편집기에서 **Favorites**에는 자주 확인할 엔티티를, **Active alerts**에는 문제가 생겼을 때 표시할 엔티티를 추가한다.

처음엔 NAS 보안 점검 결과를 자동으로 보여주는 화면이라고 생각했는데, 실제로는 문이 열렸거나 연기 감지기가 작동한 것처럼 엔티티 상태를 보여주는 용도다. NAS에 연결한 접점 센서가 `binary_sensor.nas_room_door`라면 이를 Active alerts에 넣고, NAS실 온도 센서나 UPS 배터리 상태는 Favorites에 고정하는 식으로 나누면 된다.

내 환경에서 Favorites에 고정한 항목은 아래 세 가지다.

- NAS실 문이 닫혀 있는가
- NAS실 온도가 평소 범위에 있는가
- UPS 배터리와 전원 상태가 정상인가

Active alerts에서는 엔티티별 심각도를 `Alert` 또는 `Warning`으로 정한다. NAS실 문이 오래 열려 있으면 Warning, 연기 감지기는 Alert로 두는 식이다. 경고가 없으면 이 영역 자체가 나타나지 않는 것도 이번 변경점이다.

## NAS 알림과 함께 쓰기

대시보드 경고는 화면에서 보는 용도이므로, 외부에서도 알아야 하는 장애는 모바일 알림 자동화로 보완한다. 아래 예시는 Home Assistant가 재시작될 때 관리자에게 확인 알림을 보내는 최소 구성이다.

```yaml
alias: Home Assistant 재시작 후 보안 점검 알림
description: 재시작 뒤 Security 대시보드를 확인하도록 알림
trigger:
  - platform: homeassistant
    event: start
action:
  - action: notify.mobile_app_내휴대폰
    data:
      title: "Home Assistant 재시작"
      message: "Security 대시보드의 NAS실 문·온도·UPS 상태를 확인한다."
mode: single
```

`notify.mobile_app_내휴대폰`은 실제 모바일 앱 알림 서비스 이름으로 바꿔야 한다. 자동화 저장 후 NAS를 재부팅할 필요는 없다. **개발자 도구 → 상태 → Home Assistant 재시작** 대신 실제 재시작 알림이 필요할 때만 테스트한다.

## 이 설정에서 삽질한 지점

Docker 컨테이너가 재생성된 뒤 모바일 알림 서비스 이름이 바뀌는 문제는 NAS보다 Home Assistant 모바일 앱 등록 상태에서 생긴다. 알림이 오지 않으면 `설정 → 기기 및 서비스 → 모바일 앱`에서 서비스 이름을 다시 확인한다.

Security 대시보드가 조용하다고 NAS 네트워크가 안전해진 것은 아니다. 포트포워딩을 잠깐 테스트한 뒤 그대로 두면 대시보드와 무관하게 공유기와 NAS는 계속 노출된다. 외부 접속은 Tailscale을 우선하고, 공개가 필요한 서비스만 역방향 프록시에 연결한다. Synology 관리자 포트와 SMB 포트는 인터넷에 직접 열지 않는다.

Home Assistant 2026.9의 Security 대시보드는 보안 도구 전체를 대신하지 않는다. `활성 경고 확인 → 즐겨찾기 3개 점검 → 모바일 알림 수신 확인 → NAS 백업 복원 가능 여부 확인`을 주간 루틴으로 묶는 정도가 현실적이다.

출처: [Home Assistant 2026.9 공식 릴리스 노트](https://www.home-assistant.io/blog/2026/09/02/release-20269/)

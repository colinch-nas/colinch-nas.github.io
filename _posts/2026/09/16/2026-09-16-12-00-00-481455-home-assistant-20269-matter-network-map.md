---
layout: post
title: "Home Assistant 2026.9 Matter 네트워크 맵 설정 - NAS에서 스마트홈 연결 상태 확인하기"
description: "Home Assistant 2026.9의 Matter 네트워크 맵을 Synology NAS 환경에서 확인하는 방법과 IPv6·mDNS·VLAN 문제를 실제 설정 기준으로 정리한다."
date: 2026-09-16
tags: [HomeAssistant, Matter, Synology, Docker, 스마트홈]
comments: true
share: true
---

![Home Assistant Matter 네트워크 맵을 확인하는 NAS 홈서버 구성](https://images.unsplash.com/photo-1518770660439-4636190af475?w=1200&q=80)

그림에서 볼 부분은 NAS에서 실행 중인 Home Assistant가 Matter 기기와 Thread Border Router(스레드 기기와 IP 네트워크를 연결하는 장치)의 연결 관계를 한 화면에서 확인하는 구조다.

Home Assistant 2026.9에서 Matter 네트워크 맵이 추가됐다. 기기가 등록됐는데 간헐적으로 오프라인이 되거나, Thread 기기만 유독 응답이 늦을 때 무작정 컨테이너를 재시작하지 않아도 된다. 맵에서 연결 품질과 경로를 확인한 뒤 공유기 설정을 고칠 수 있다. 여기서는 Synology DS923+, DSM 7.2.2, Container Manager, Home Assistant 2026.9, Matter Server Docker 구성을 기준으로 했다.

## 업데이트 전에 확인할 환경

Home Assistant 공식 문서 기준으로 Matter는 로컬 IPv6와 mDNS(같은 네트워크의 기기를 자동 발견하는 멀티캐스트 방식)에 의존한다. 그래서 NAS와 스마트홈 기기를 서로 다른 VLAN(논리적으로 나눈 네트워크)에 두고 멀티캐스트를 막아두면 맵보다 페어링부터 실패한다.

| 항목 | 이 글의 기준 | 확인할 점 |
|---|---|---|
| NAS | Synology DS923+ | NAS 고정 IP, Docker 네트워크 모드 |
| Home Assistant | Core 2026.9 | 2026.9.1 이상 패치 권장 |
| Matter Server | 공식 Docker 이미지 | WebSocket 포트 5580 |
| 네트워크 | 같은 LAN 또는 mDNS 릴레이 구성 | IPv6 자동 활성화 |

처음에는 IoT VLAN을 분리하면 보안이 좋아진다고 생각하기 쉽다. 실제로는 Matter를 붙인 직후부터 검색이 끊겼다. 공유기에서 `mDNS reflector`, `Bonjour gateway`, `멀티캐스트 전달` 중 하나를 지원하지 않는다면 NAS와 Matter 기기를 같은 VLAN에 두는 편이 빠르다.

## Matter Server와 Home Assistant 연결

이미 Matter Server를 운영 중이면 이 단계는 건너뛰고 Home Assistant의 연결 주소만 확인하면 된다. 새로 만드는 경우 Container Manager 프로젝트에 아래 Compose를 넣는다.

```yaml
services:
  matter-server:
    image: ghcr.io/matter-js/matter.js/matter-server:latest
    container_name: matter-server
    restart: unless-stopped
    network_mode: host
    volumes:
      - /volume1/docker/matter-server:/data
    environment:
      - TZ=Asia/Seoul
```

`network_mode: host`는 Matter의 mDNS 검색을 단순하게 만들기 위한 선택이다. NAS의 5580 포트를 외부에 공개하는 설정은 아니다. 프로젝트를 시작한 뒤 Home Assistant에서 `설정 → 기기 및 서비스 → 통합 추가 → Matter`를 열고, 외부 Matter Server 연결을 선택해 `ws://NAS_IP:5580/ws`를 입력한다.

Home Assistant가 이미 Docker에서 실행 중이라면 두 컨테이너 모두 host 네트워크를 쓰거나, 최소한 NAS의 실제 LAN 인터페이스로 멀티캐스트가 전달돼야 한다. 여기서 `matter-server:5580`처럼 서비스 이름만 넣고 연결되지 않아 한참 헤맸다. 다른 Docker 네트워크에 있는 컨테이너 이름은 자동으로 보이지 않는다.

## 2026.9 Matter 네트워크 맵 열기

Matter 통합이 정상 연결된 뒤 기기가 한 개 이상 등록돼 있어야 맵에 의미 있는 링크가 표시된다.

1. `설정 → 기기 및 서비스`에서 Matter 통합을 연다.
2. Matter 통합 카드의 메뉴에서 네트워크 맵 또는 연결 시각화 항목을 연다.
3. 기기 노드를 클릭해 연결 품질, 마지막 응답, 연결된 Border Router를 확인한다.
4. Thread 기기만 끊기면 Thread Border Router와 NAS의 IPv6 경로를 함께 점검한다.

| 화면에서 보이는 현상 | 의심할 부분 | 조치 |
|---|---|---|
| 기기가 맵에 없음 | Matter Server 연결·페어링 실패 | WebSocket 주소와 기기 재등록 확인 |
| Wi-Fi 기기만 끊김 | AP의 멀티캐스트 차단 | IoT VLAN의 mDNS 허용 |
| Thread 기기만 끊김 | Border Router 또는 IPv6 | Thread 네트워크와 IPv6 자동 설정 확인 |
| 맵은 보이지만 응답 지연 | 무선 품질·AP 격리 | AP isolation 해제, 공유기 로그 확인 |

Matter 통합 문서도 Thread 기기에는 Thread Border Router가 필요하고, `설정 → 시스템 → 네트워크`에서 IPv6를 자동 또는 고정으로 활성화하라고 안내한다. 단순히 Matter 기기를 NAS 가까이에 옮기는 것보다 이 두 항목을 확인하는 편이 효과가 컸다.

## 외부 공개와 백업 주의사항

Matter Server의 5580 포트를 공유기에서 포트포워딩하지 않는다. 외부에서 Home Assistant를 써야 한다면 Tailscale VPN이나 Home Assistant Cloud를 사용하고, 관리자 화면을 Cloudflare Tunnel로 공개하는 경우에도 인증 계층을 하나 더 둔다.

설정 변경 전에는 Home Assistant 백업과 Matter Server의 `/data` 폴더를 함께 복사한다. Matter 기기를 다시 페어링하면 장치 식별자가 달라질 수 있어, 컨테이너만 새로 만들고 데이터 폴더를 버리면 자동화가 전부 끊길 수 있다.

## 짧은 점검표

- Home Assistant 2026.9와 Matter Server가 연결됐는가
- NAS와 Matter 기기가 같은 VLAN에 있거나 mDNS 릴레이가 켜졌는가
- IPv6가 비활성화되지 않았는가
- Thread 기기에 Border Router가 있는가
- 5580 포트를 인터넷에 공개하지 않았는가
- Home Assistant 백업과 Matter Server `/data`를 함께 보관했는가

공식 변경 내용은 [Home Assistant 2026.9 릴리스 노트](https://www.home-assistant.io/blog/2026/09/02/release-20269/)와 [Matter 통합 문서](https://www.home-assistant.io/integrations/matter/)에서 확인할 수 있다.

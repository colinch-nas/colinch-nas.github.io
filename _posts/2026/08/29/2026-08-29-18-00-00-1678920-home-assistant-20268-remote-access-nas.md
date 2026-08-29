---
layout: post
title: "Home Assistant 2026.8 원격접속 - Synology NAS에서 Cloud·Tailscale 선택 기준"
description: "Home Assistant 2026.8의 새 Cloud 화면을 기준으로 Synology NAS 원격접속을 Nabu Casa Cloud, Tailscale, Cloudflare Tunnel로 나눠 설정하고 선택 기준을 정리한다."
date: 2026-08-29
tags: [HomeAssistant, Synology, Tailscale, CloudflareTunnel, NAS보안]
comments: true
share: true
---

![Synology NAS와 Home Assistant 원격접속 구성](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

Home Assistant 2026.8을 Synology NAS에서 운영한다면 외부 접속은 포트포워딩부터 열 필요가 없다. 이번 버전은 Home Assistant Cloud 화면을 기능별로 나누고 안내형 설정을 추가했다. 비용을 내고 빨리 끝내려면 Nabu Casa Cloud, 무료로 가족 기기까지 묶으려면 Tailscale, 공개 도메인이 필요하면 Cloudflare Tunnel이 맞다.

## 세 가지 방식 비교

| 방식 | 비용 | 포트포워딩 | 설정 난이도 | 어울리는 경우 |
|---|---:|---|---|---|
| Home Assistant Cloud | 유료 | 없음 | 낮음 | 처음 구축하거나 음성·백업까지 쓸 때 |
| Tailscale | 개인 용도 무료 구간 | 없음 | 낮음 | 내 휴대폰과 노트북에서만 접속할 때 |
| Cloudflare Tunnel | 무료 구간 있음 | 없음 | 중간 | `ha.example.com` 같은 공개 주소가 필요할 때 |

이 그림에서 봐야 할 부분은 세 방식 모두 공유기에서 Home Assistant 포트 `8123`을 인터넷에 직접 공개하지 않는다는 점이다.

## Home Assistant Cloud로 끝내기

Synology DS923+, DSM 7.2.2의 Container Manager에서 Home Assistant 2026.8.3 컨테이너를 띄운 상태라면 Home Assistant 화면에서 `설정 → Home Assistant Cloud`로 들어간다. 2026.8부터 이 화면이 원격접속, 백업, 음성, Companion 앱 영역으로 나뉘어 원하는 항목을 찾기 쉬워졌다.

로그인 후 원격접속을 켜고 Companion 앱에서 계정을 연결하면 된다. 이 방식은 인증서, DDNS(유동 IP를 도메인처럼 연결하는 서비스), 역방향 프록시를 직접 관리하지 않는 대신 월 구독료가 든다. 가족이 외부에서 자주 접속하거나 고장 대응 시간을 줄이고 싶다면 이 비용이 가장 덜 피곤했다.

공식 릴리스 노트: [Home Assistant 2026.8](https://www.home-assistant.io/blog/2026/08/05/release-20268/)

## Tailscale은 개인용 관리 화면에 적합하다

Tailscale(기기 간 암호화 VPN)을 NAS와 휴대폰에 설치하고 같은 계정으로 로그인한다. NAS의 Tailscale IP를 확인한 뒤 휴대폰 브라우저에서 아래처럼 접속한다.

```text
http://100.x.y.z:8123
```

Home Assistant의 `설정 → 시스템 → 네트워크`에서 내부 URL을 Tailscale 주소로 바꾸면 Companion 앱도 같은 경로를 사용한다. 공유기 포트를 열지 않는 점은 좋지만, Tailscale 앱이 꺼진 기기나 초대하지 않은 가족의 휴대폰에서는 접속할 수 없다. 외부 공개 서비스라기보다 내 관리망을 안전하게 연장하는 방식이다.

## Cloudflare Tunnel을 쓸 때 확인할 것

공개 도메인이 필요하다면 Cloudflare Tunnel 컨테이너를 별도로 운영하고 `ha.example.com`을 Home Assistant의 `8123`으로 연결한다. Home Assistant에는 프록시 주소를 신뢰하도록 `configuration.yaml`에 다음을 추가한다.

```yaml
http:
  use_x_forwarded_for: true
  trusted_proxies:
    - 172.20.0.0/16
```

여기서 `172.20.0.0/16`은 Tunnel 컨테이너 네트워크에 맞춰 바꿔야 한다. 무작정 `0.0.0.0/0`을 넣으면 모든 프록시를 신뢰하게 되므로 피한다. 설정 후 컨테이너를 재시작하고 `도구 → 구성 확인`에서 YAML 오류가 없는지 확인한다.

## 실제 선택 체크리스트

- 내 기기만 접속: Tailscale
- 가족에게 링크를 공유: Home Assistant Cloud
- 별도 도메인과 여러 자체 호스팅 서비스를 함께 운영: Cloudflare Tunnel
- 관리자 계정에 MFA 적용
- NAS와 Home Assistant 백업을 같은 디스크에만 두지 않기
- `8123` 포트 직접 포워딩은 사용하지 않기

Home Assistant 2026.8의 새 Cloud 화면은 기능을 쉽게 찾게 해주지만, 원격접속 보안까지 자동으로 해결해 주는 것은 아니다. NAS 초보라면 Cloud 또는 Tailscale로 시작하고, 공개 도메인이 꼭 필요해졌을 때 Tunnel로 확장하는 순서가 시행착오가 적다.

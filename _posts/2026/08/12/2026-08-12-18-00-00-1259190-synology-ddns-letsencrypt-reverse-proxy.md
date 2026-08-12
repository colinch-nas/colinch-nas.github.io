---
layout: post
title: "시놀로지 외부접속 설정 - DSM 7.4 DDNS와 Let's Encrypt 인증서"
description: "Synology NAS를 DSM 7.4 기준으로 외부에서 안전하게 접속하는 방법이다. DDNS, 공유기 포트포워딩, Let's Encrypt 인증서, 역방향 프록시까지 실제 설정 순서로 정리했다."
date: 2026-08-12
tags: [Synology, DSM, NAS설정, DDNS, NAS보안, NginxProxyManager]
comments: true
share: true
---

![시놀로지 NAS DDNS와 HTTPS 외부접속 구성](/assets/images/2026/07/synology-ddns-reverse-proxy-home-server.png)

시놀로지 NAS 외부접속은 `QuickConnect`만으로 끝내기보다 DDNS(유동 IP를 도메인으로 연결하는 서비스)와 HTTPS 인증서를 같이 구성하는 편이 낫다. DSM 7.4의 `제어판 → 외부 액세스 → DDNS`에서 주소를 만들고, 공유기에는 80·443 포트만 NAS로 전달하면 된다. 이후 `drive.example.com`이나 `photos.example.com`처럼 서비스별 주소도 붙일 수 있다.

이 글은 Synology NAS를 내부 IP `192.168.1.20`으로 고정하고, 공유기가 공인 IPv4를 직접 받는 환경을 기준으로 했다. 통신사가 CGNAT를 쓰는 환경이면 포트포워딩이 동작하지 않으므로 Tailscale이나 Cloudflare Tunnel을 선택해야 한다.

## QuickConnect와 DDNS 중 무엇을 쓸까

Synology 공식 비교표 기준으로 QuickConnect는 라우터 설정이 거의 필요 없고 초보자에게 맞는다. 반면 DDNS는 포트포워딩이 필요하지만 전송 속도가 유리하고, 서드파티 앱과 사용자 도메인을 연결할 수 있다.

| 방식 | 장점 | 한계 | 맞는 용도 |
|---|---|---|---|
| QuickConnect | 설정이 빠르고 포트 개방이 적다 | 일부 서드파티 앱 연결 불가 | DSM, Drive 모바일 접속 |
| DDNS + HTTPS | Jellyfin·Immich·Vaultwarden 주소 연결 가능 | 공유기·인증서 설정 필요 | 자체호스팅 서비스 |
| Tailscale | 포트포워딩 없이 사설망처럼 접속 | 외부 사용자 공유에는 불편 | 관리자·가족 전용 접속 |

그림에서 볼 것은 NAS를 인터넷에 바로 노출하는 게 아니라, 공유기의 443 요청을 HTTPS로 받은 뒤 내부 서비스로 나누는 흐름이다.

## 1. NAS 내부 IP를 고정한다

공유기 DHCP 예약에서 NAS MAC 주소에 `192.168.1.20`을 지정했다. NAS에서 수동 IP를 따로 지정해도 되지만, 공유기와 중복되면 접속이 끊기므로 한 곳에서만 관리하는 편이 안전하다.

DSM에서 `제어판 → 네트워크 → 네트워크 인터페이스`로 현재 주소를 확인하고, NAS가 재부팅돼도 같은 IP를 받는지 확인한다.

## 2. DSM 7.4에서 DDNS를 만든다

`제어판 → 외부 액세스 → DDNS → 추가`를 열고 다음처럼 입력한다.

| 항목 | 예시 |
|---|---|
| 서비스 제공업체 | Synology |
| 호스트 이름 | `myhome-nas.synology.me` |
| 외부 주소 확인 | 자동 |
| 인증서 요청 | Let's Encrypt 선택 |
| 이메일 | 갱신 알림을 받을 주소 |

생성 후 `myhome-nas.synology.me`가 현재 공인 IP를 가리키는지 확인한다. DDNS는 주소를 IP에 매핑할 뿐이라서, 이것만 설정해도 외부 접속이 열리는 것은 아니다.

## 3. 공유기 포트는 443만 공개한다

공유기 포트포워딩에 아래 규칙을 만든다.

```text
외부 TCP 443 → 192.168.1.20:443
```

Let's Encrypt HTTP-01 인증이 실패하면 발급 순간에만 TCP 80을 `192.168.1.20:80`으로 열고, 인증서가 발급된 뒤에는 닫는다. DSM 관리 화면을 `5000·5001` 포트로 직접 공개하는 구성은 피했다. 관리자 로그인 화면을 인터넷에 그대로 노출하는 면적이 커지기 때문이다.

## 4. 인증서를 기본값으로 지정한다

`제어판 → 보안 → 인증서`에서 발급된 인증서를 선택하고 `구성`을 눌러 DSM, Web Station, Synology Drive처럼 실제 사용할 서비스에 배정한다. 브라우저 주소도 반드시 IP가 아니라 `https://myhome-nas.synology.me`로 접속해야 경고가 사라진다.

인증서가 발급되지 않으면 DNS가 아직 갱신되지 않았거나, 80 포트가 다른 장비로 향하는 경우가 많았다. 휴대폰을 Wi-Fi에서 끈 뒤 LTE로 접속해 확인하는 것이 정확하다.

## 5. 서비스별 주소는 역방향 프록시로 나눈다

DSM의 `로그인 포털 → 고급 → 역방향 프록시`에서 규칙을 추가한다. Reverse Proxy(외부 요청을 내부 서비스로 전달하는 중계 서버)를 쓰면 하나의 443 포트로 여러 컨테이너를 운영할 수 있다.

```text
소스: https / photos.example.com / 443
대상: http / 192.168.1.20 / 2283
```

DNS 제공업체에는 `photos.example.com`을 집 공인 IP로 연결한다. Immich의 실제 포트가 2283이 아니라면 Container Manager의 포트 매핑값을 기준으로 바꾼다. WebSocket을 쓰는 서비스는 프록시 설정에서 WebSocket 지원도 켜야 로그인 후 화면이 멈추지 않는다.

## 외부 공개 전 체크리스트

- 관리자 계정에 2단계 인증을 켰는가
- DSM 자동 차단과 방화벽 국가·포트 규칙을 확인했는가
- 공유기 UPnP를 끄고 포트포워딩을 직접 관리하는가
- 80 포트가 계속 열려 있지 않은가
- 모바일 LTE에서 DDNS와 HTTPS가 모두 동작하는가
- 인증서 만료일과 갱신 실패 알림을 확인했는가

내 환경에서는 DSM 자체는 QuickConnect로 남기고, 외부 공개가 필요한 Jellyfin·Immich만 DDNS와 역방향 프록시로 분리하는 구성이 가장 관리하기 쉬웠다. 외부 접속은 주소를 만드는 작업보다 공개 범위를 줄이는 작업이 핵심이다.

참고한 공식 문서:

- [Synology NAS External Access Quick Start Guide](https://kb.synology.com/en-sg/DSM/tutorial/Quick_Start_External_Access)
- [DSM 7.4 External Access 안내](https://kb.synology.com/en-global/DSM/help/DSM/AdminCenter/connection_public_access_desc?version=7)
- [QuickConnect와 DDNS 차이](https://kb.synology.com/en-ca/DSM/tutorial/What_are_the_differences_between_QuickConnect_and_DDNS)

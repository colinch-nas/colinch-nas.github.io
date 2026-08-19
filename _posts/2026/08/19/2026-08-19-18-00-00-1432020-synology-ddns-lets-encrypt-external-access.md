---
layout: post
title: "Synology 외부접속 설정 - DSM 7.2.2 DDNS와 Let's Encrypt 인증서"
description: "Synology NAS를 DSM 7.2.2 기준으로 외부에서 안전하게 접속하는 방법을 정리한다. DDNS, 공유기 포트포워딩, Let's Encrypt HTTPS 인증서와 역방향 프록시 설정까지 직접 따라 할 수 있다."
date: 2026-08-19
tags: [Synology, DSM, DDNS, NAS보안, CloudflareTunnel]
comments: true
share: true
---

![Synology NAS 외부 접속과 HTTPS 구성](/assets/images/2026/08/synology-ddns-https-external-access.png)

Synology NAS 외부접속은 DSM 7.2.2에서 DDNS(유동 IP를 도메인처럼 연결하는 서비스)를 만들고, HTTPS 인증서를 붙인 뒤 필요한 포트만 여는 방식이 가장 이해하기 쉽다. 여기서는 `nasname.synology.me`로 DSM과 Docker 서비스를 접속하는 구성을 기준으로 한다.

## 준비할 환경

| 항목 | 예시 |
|---|---|
| NAS | Synology DS923+ |
| 운영체제 | DSM 7.2.2 |
| NAS 내부 IP | `192.168.0.20` |
| 외부 주소 | `nasname.synology.me` |
| 공유기 | 공인 IPv4를 받는 일반 공유기 |

가장 먼저 확인할 것은 통신사나 공유기의 CGNAT 여부다. 공유기 WAN 주소와 `whatismyip` 같은 사이트의 주소가 다르면 포트포워딩을 해도 접근되지 않는다. 이 경우엔 포트 개방 대신 Tailscale이나 Cloudflare Tunnel을 쓰는 편이 낫다.

## 1. DSM에서 DDNS 만들기

DSM에서 `제어판 > 외부 액세스 > DDNS`로 들어가 `추가`를 누른다. 서비스 제공업체는 `Synology`를 선택하고 호스트 이름에 원하는 이름을 입력한다. 예를 들어 `nasname.synology.me`가 사용 가능하면 등록한다.

`연결 테스트`가 성공해야 한다. 이 화면의 `Let's Encrypt 인증서를 가져와 기본 인증서로 설정`을 함께 선택하면 DDNS 등록과 인증서 발급을 한 번에 처리할 수 있다. 인증서에 사용할 이메일은 만료 알림을 받을 수 있는 주소를 넣는다.

처음엔 이 옵션을 찾지 못해 인증서를 별도로 발급했는데, Synology DDNS를 쓰는 경우에는 DDNS 화면에서 처리하는 편이 덜 헷갈린다. 단, 인증서 발급 검증을 위해 외부에서 NAS의 80번 포트로 접근할 수 있어야 하는 환경이 있다.

## 2. 공유기에서 포트포워딩 설정

공유기 관리자 화면에서 아래 두 규칙만 만든다.

| 외부 포트 | 내부 IP | 내부 포트 | 용도 |
|---:|---|---:|---|
| 80 | 192.168.0.20 | 80 | 인증서 발급·갱신 검증 |
| 443 | 192.168.0.20 | 443 | HTTPS 외부 접속 |

DSM 기본 관리 포트인 5000, 5001을 외부에 직접 노출하는 구성은 피한다. 인증서 발급이 끝난 뒤에도 80번 포트가 계속 필요할 수 있으므로, 공유기에서 80을 닫기 전 자동 갱신이 가능한지 먼저 확인한다. 외부에서 `https://nasname.synology.me`를 열었을 때 DSM 로그인 화면이 나오면 443 연결은 된 상태다.

## 3. 인증서와 DSM 연결 확인

`제어판 > 보안 > 인증서`에서 `nasname.synology.me` 인증서를 선택하고 `구성`을 누른다. DSM과 File Station처럼 외부에서 쓸 서비스에 이 인증서를 지정한다. 브라우저 주소창에 경고가 사라지고 자물쇠가 표시되면 인증서 연결은 끝난다.

여기서 자주 한 삽질은 DDNS 이름과 실제 접속 주소를 다르게 쓰는 것이다. 인증서가 `nasname.synology.me`용이면 IP 주소나 다른 별칭으로 접속할 때 경고가 다시 뜬다. 외부 주소를 하나로 고정해 북마크와 모바일 앱에도 같은 주소를 사용한다.

## 4. Docker 서비스는 역방향 프록시로 분리

Jellyfin이나 Uptime Kuma를 5001 포트에 억지로 붙이지 말고 `제어판 > 로그인 포털 > 고급 > 역방향 프록시`에서 도메인별 규칙을 만든다. 역방향 프록시(외부 요청을 내부 서비스로 전달하는 중계 서버)의 예시는 다음과 같다.

| 구분 | 값 |
|---|---|
| 소스 프로토콜/호스트/포트 | HTTPS / `jellyfin.nasname.synology.me` / 443 |
| 대상 프로토콜/호스트/포트 | HTTP / `127.0.0.1` / 8096 |

소스에 HTTPS와 443을 쓰고, 인증서 화면에서 `jellyfin.nasname.synology.me`를 인증서의 주체 대체 이름(SAN)으로 추가한다. Jellyfin처럼 실시간 스트리밍이나 알림이 있는 서비스는 규칙 생성 화면의 `WebSocket` 헤더도 추가해야 재생 화면에서 연결이 끊기지 않는다.

## 외부 공개 전 체크리스트

- DSM 관리자 계정에 2단계 인증을 켠다.
- 관리자 계정 이름을 기본 `admin`으로 두지 않는다.
- DSM 방화벽에서 허용 국가와 포트를 제한한다.
- 공유기 UPnP를 끄고, 포트포워딩 목록을 직접 관리한다.
- Hyper Backup이나 Snapshot Replication으로 설정과 데이터를 별도 백업한다.
- 외부 접속은 모바일 데이터에서 실제로 테스트한다.

핵심은 `DDNS → HTTPS 인증서 → 필요한 서비스만 역방향 프록시` 순서다. 5001 포트를 인터넷에 그대로 열어 두는 것보다 주소와 인증서를 서비스별로 나누기 쉽고, 나중에 Docker 서비스를 추가할 때도 관리가 편해진다. Synology 공식 문서도 DSM의 DDNS 화면에서 Let’s Encrypt 인증서를 발급할 수 있고, Application Portal에서 역방향 프록시 규칙을 만들 수 있다고 안내한다.

참고 문서: [Synology DDNS](https://kb.synology.com/en-global/DSM/help/DSM/AdminCenter/connection_ddns), [Synology 인증서](https://kb.synology.com/en-us/DSM/help/DSM/AdminCenter/connection_certificate), [Application Portal 역방향 프록시](https://kb.synology.com/en-us/DSM/help/DSM/AdminCenter/application_appportalias)

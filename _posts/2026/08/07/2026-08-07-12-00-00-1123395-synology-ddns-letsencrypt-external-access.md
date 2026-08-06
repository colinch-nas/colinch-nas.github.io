---
layout: post
title: "Synology NAS 외부접속 - DSM 7.2 DDNS와 Let's Encrypt 설정"
description: "Synology DSM 7.2.2에서 DDNS를 연결하고 Let's Encrypt 인증서를 발급해 NAS를 HTTPS로 외부 접속하는 방법을 정리한다. 포트 80 검증과 갱신 실패 원인까지 확인한다."
date: 2026-08-07
tags: [Synology, DSM, NAS설정, DDNS, NAS보안]
comments: true
share: true
---

![Synology NAS DDNS와 Let's Encrypt 외부접속 구성](/assets/images/synology-ddns-letsencrypt-remote-access.png)

NAS 외부접속은 QuickConnect만 켜는 것보다 **DDNS(유동 IP를 도메인처럼 연결하는 기능) + HTTPS 인증서**를 같이 구성하는 편이 관리하기 쉽다. 내가 확인한 DSM 7.2.2 기준으로는 `nas이름.synology.me` 주소를 만들고, Let's Encrypt 인증서를 발급한 뒤 DSM 서비스에 연결하면 된다. 단, 인증서 발급과 갱신 때 인터넷에서 NAS의 80번 포트로 도메인 검증이 들어온다는 점을 놓치면 바로 막힌다.

이 그림에서 NAS와 외부 접속 장치 사이의 HTTPS 연결 구성을 확인하면 된다.

## 준비한 환경

| 항목 | 설정값 |
|---|---|
| NAS | Synology DS923+ |
| 운영체제 | DSM 7.2.2 |
| 공유기 | 공인 IPv4 사용, 80·443 포트 전달 가능 |
| 주소 | `myhome.synology.me` 예시 |
| 목표 | DSM을 `https://myhome.synology.me:5001`로 접속 |

CGNAT(통신사 공유망) 환경이면 공유기에서 포트를 열어도 NAS까지 요청이 도달하지 않는다. 이 경우에는 [DSM 역방향 프록시로 Docker 앱을 연결한 구성]({% post_url 2026-07-23-12-00-00-1049325-synology-dsm-reverse-proxy-docker-subdomain %})과 별개로 먼저 공인 IP 또는 VPN 사용 가능 여부를 확인해야 한다.

## 1. Synology DDNS 등록

DSM에서 **제어판 → 외부 액세스 → DDNS → 추가**를 연다. 서비스 제공업체는 `Synology`로 두고 다음처럼 입력한다.

```text
호스트 이름: myhome.synology.me
사용자 이름/이메일: Synology 계정
외부 주소: 자동 감지
Let's Encrypt 인증서 발급: 체크
```

`연결 테스트`가 성공하면 DDNS 주소가 현재 공인 IP를 가리킨다. 공유기에서 공인 IP를 확인했는데 DSM의 외부 주소와 다르면 이 단계에서 멈추는 게 맞다. 이 상태에서 인증서를 반복 발급하면 실패 횟수만 쌓인다.

## 2. 공유기 포트 전달

공유기 관리 화면에서 NAS의 고정 내부 IP를 목적지로 지정한다.

| 외부 포트 | 내부 포트 | 용도 |
|---:|---:|---|
| 80 | 80 | Let's Encrypt 도메인 검증·갱신 |
| 443 | 443 | HTTPS 기본 포트 선택 시 |
| 5001 | 5001 | DSM HTTPS 접속 |

DSM 기본 HTTPS 포트가 5001이므로 주소에 포트를 붙이는 구성이 가장 단순하다. 인증서 발급만 필요할 때는 80번 전달이 핵심이다. 발급이 끝난 뒤 80번을 닫고 싶어도, DSM이 HTTP 방식으로 갱신하는 환경이라면 90일 뒤 갱신이 실패할 수 있다. Synology 공식 문서도 인증서 유효기간을 90일로 안내하고 갱신 검증을 위해 80번 포트를 열어 두라고 설명한다.

## 3. 인증서와 DSM 서비스 연결

DDNS 화면에서 인증서 발급을 체크하지 않았다면 **제어판 → 보안 → 인증서 → 추가**로 들어간다. `Let's Encrypt에서 인증서 가져오기`를 선택하고 아래처럼 입력한다.

```text
도메인 이름: myhome.synology.me
이메일: 실제 알림을 받을 주소
주체 대체 이름(SAN): nas.myhome.synology.me (선택)
```

발급 후 **인증서 → 설정**에서 `DSM`과 `웹 서비스`에 방금 만든 인증서를 지정한다. 인증서가 있어도 기본 인증서가 Synology 기본 인증서로 남아 있으면 브라우저에 “안전하지 않음”이 계속 표시된다. 반드시 도메인으로 접속해 확인한다. `192.168.1.20:5001` 같은 내부 IP 접속에는 이 도메인 인증서가 맞지 않는다.

## 막혔을 때 확인할 순서

| 증상 | 먼저 확인할 것 |
|---|---|
| 인증서 발급 실패 | 도메인이 현재 공인 IP를 가리키는지, 80번 포트가 NAS까지 오는지 |
| DDNS 연결 테스트 실패 | 공유기 이중 NAT, CGNAT, 외부 주소 수동 입력 여부 |
| HTTPS인데 경고 표시 | 인증서 설정에서 DSM 서비스에 올바른 인증서가 연결됐는지 |
| 3개월 뒤 접속 불가 | 80번 포트 변경·차단, 공유기 규칙 삭제, NAS IP 변경 여부 |

외부에 DSM 로그인 화면을 공개하는 만큼 관리자 계정에는 2단계 인증을 켜고, 기본 `admin` 계정은 비활성화하는 게 좋다. 포트 번호를 바꾸는 것만으로는 보안 대책이 되지 않는다. 가능하면 DSM 외부 접속은 VPN으로 제한하고, 공개해야 하는 서비스만 역방향 프록시에서 별도 도메인으로 분리한다.

## 짧은 요점 정리

- DDNS가 공인 IP를 제대로 가리키는지 먼저 확인한다.
- Let's Encrypt 발급과 갱신에는 80번 포트 도메인 검증이 필요하다.
- 인증서 발급 후 DSM 서비스에 해당 인증서를 직접 지정한다.
- 2단계 인증과 관리자 계정 정리를 외부 공개 전에 끝낸다.

참고한 공식 문서: [Synology DDNS 설정](https://kb.synology.com/en-global/DSM/help/DSM/AdminCenter/connection_ddns), [Synology 인증서 설정](https://kb.synology.com/en-us/DSM/help/DSM/AdminCenter/connection_certificate), [안전하지 않음 경고 해결](https://kb.synology.com/en-us/DSM/tutorial/Why_did_I_see_a_not_secure_warning_in_the_browser_when_connecting_to_my_Synology_product)

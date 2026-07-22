---
layout: post
title: "Synology DSM 7.2.2 역방향 프록시 - Docker 앱을 서브도메인으로 공개하는 설정"
description: "Synology DSM 7.2.2에서 DDNS와 Let's Encrypt 인증서를 연결하고 Docker 앱을 서비스별 서브도메인으로 공개하는 역방향 프록시 설정법을 정리한다."
date: 2026-07-23
tags: [Synology, DSM, NAS설정, Docker, NAS보안, DDNS]
comments: true
share: true
---

![Synology NAS DDNS와 역방향 프록시 구성](/assets/images/2026/07/synology-ddns-reverse-proxy-home-server.png)

그림에서 봐야 할 부분은 NAS의 443 포트 하나로 여러 내부 Docker 서비스를 나눠 연결하는 구조다.

Synology DS923+에 DSM 7.2.2를 설치한 뒤 `home.example.com:8096`처럼 포트를 붙여 Jellyfin에 접속하는 방식은 오래 쓰기 불편하다. DSM의 역방향 프록시(외부 요청을 내부 서비스로 전달하는 중계 기능)를 쓰면 `jellyfin.example.com`으로 정리할 수 있다. 외부에는 443만 열고, 내부 포트는 공개하지 않는 구성이 핵심이다.

## 이번에 구성한 환경

| 항목 | 값 |
|---|---|
| NAS | Synology DS923+ |
| 운영체제 | DSM 7.2.2 |
| 대상 서비스 | Docker Jellyfin, 내부 포트 8096 |
| 도메인 | `example.com`과 `jellyfin.example.com` |
| 공유기 전달 | TCP 80, 443 → NAS |

도메인이 없다면 Synology DDNS의 `이름.synology.me`로도 시작할 수 있다. DSM 공식 문서 기준으로 DDNS는 유동 외부 IP를 호스트명에 연결하고, Synology 제공 호스트명은 인증서 발급 안내까지 이어진다.

## DDNS와 인증서 연결

DSM에서 `제어판 → 외부 액세스 → DDNS → 추가`를 연다. 서비스 제공업체를 Synology로 선택하고 호스트명을 정한 뒤 **연결 테스트**를 실행한다. 공유기가 별도 공인 IP를 받지 않는 CGNAT 환경이면 여기서 성공해도 외부 접속이 되지 않는다.

`제어판 → 보안 → 인증서 → 추가`에서 `Let's Encrypt에서 인증서 가져오기`를 선택한다. 도메인에는 `example.com`, 이메일에는 실제 알림을 받을 주소를 입력한다. 발급 검증과 90일 인증서 갱신을 위해 TCP 80이 외부에서 NAS까지 도달해야 한다. 발급 뒤에는 `설정 → 구성`에서 DSM과 Web Station이 새 인증서를 사용하도록 지정한다.

여기서 한 번 삽질했다. 인증서는 발급됐는데 브라우저가 계속 경고를 냈다. 인증서 목록에 존재하는 것과 서비스에 배정된 것은 별개라서, 기본 인증서만 바꾸지 말고 해당 서비스별 매핑까지 확인해야 한다.

## 역방향 프록시 규칙 만들기

`제어판 → 로그인 포털 → 고급 → 역방향 프록시 → 생성`을 선택한다. Jellyfin 규칙은 아래처럼 입력한다.

| 구분 | 프로토콜 | 호스트 이름 | 포트 |
|---|---|---|---:|
| 소스 | HTTPS | `jellyfin.example.com` | 443 |
| 대상 | HTTP | `127.0.0.1` 또는 Docker 브리지 주소 | 8096 |

Docker 컨테이너가 NAS의 포트를 `8096:8096`으로 바인딩된 상태라면 대상은 `127.0.0.1:8096`으로 충분하다. 컨테이너 전용 네트워크에만 붙어 있다면 NAS에서 접근 가능한 컨테이너 IP나 같은 Docker 네트워크의 프록시 구성을 사용해야 한다. 저장한 뒤 인증서 설정에서 `jellyfin.example.com` 인증서를 이 규칙에 연결한다.

WebSocket을 쓰는 서비스는 `사용자 지정 헤더 → WebSocket`을 활성화한다. Jellyfin은 일반 재생만 테스트하면 놓치기 쉬운데, 모바일 앱의 실시간 상태와 일부 원격 재생에서 연결 문제가 드러났다.

## 공유기와 보안 점검

공유기 포트포워딩은 아래 두 개만 남긴다.

```text
TCP 80  → NAS 내부 IP:80   # Let's Encrypt 검증·갱신
TCP 443 → NAS 내부 IP:443  # HTTPS 역방향 프록시
```

DSM 관리 포트 5000, 5001과 Jellyfin 8096을 외부에 직접 열면 안 된다. 관리자 계정에는 2단계 인증을 적용하고, `제어판 → 보안 → 방화벽`에서 관리 포트 접근 대역도 제한한다. 내부 Wi-Fi에서 도메인 접속이 안 되면 공유기의 NAT loopback(내부에서 공인 주소로 되돌아오는 기능) 미지원일 수 있으니 휴대폰 LTE로도 확인한다.

## 짧은 점검표

- [ ] DDNS 상태가 정상이고 외부 IP가 현재 회선과 일치한다.
- [ ] TCP 80, 443만 포워딩했다.
- [ ] Let's Encrypt 인증서를 서비스에 배정했다.
- [ ] 소스 호스트명과 대상 포트를 각각 확인했다.
- [ ] 관리자 포트와 Docker 앱 원본 포트는 외부에서 닫혀 있다.
- [ ] LTE 환경에서 HTTPS 접속과 앱 로그인을 시험했다.

이 구성은 포트 번호를 숨기는 것만으로 보안이 완성되는 방식은 아니다. NAS와 Docker 앱을 계속 업데이트하고, 공개할 서비스와 내부에서만 쓸 서비스를 역방향 프록시 규칙 단계에서 나눠야 한다. 내 환경에서는 443 하나로 앱을 정리한 뒤 북마크와 인증서 관리가 훨씬 단순해졌다.

참고: [Synology DDNS 공식 문서](https://kb.synology.com/en-global/DSM/help/DSM/AdminCenter/connection_ddns?version=7), [Synology 인증서 공식 문서](https://kb.synology.com/en-us/DSM/help/DSM/AdminCenter/connection_certificate)

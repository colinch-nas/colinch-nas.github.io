---
layout: post
title: "Home Assistant Companion App 2026.8 - 구형 iPhone에서 NAS 스마트홈 접속 유지하는 법"
description: "Home Assistant Companion App 2026.8부터 iOS 15, watchOS 8, macOS 11 지원이 종료된다. NAS에서 운영 중인 Home Assistant 사용자가 현재 버전 확인과 브라우저 대체 접속을 준비하는 방법을 정리한다."
date: 2026-07-21
tags: [HomeAssistant, 스마트홈, NAS설정, 홈오토메이션, 자체호스팅]
comments: true
share: true
---

![Home Assistant NAS 서버와 iPhone 접속 점검](https://images.unsplash.com/photo-1558655146-d09347e92766?w=1200&q=80)

구형 iPhone이나 Mac으로 NAS의 Home Assistant를 쓰고 있다면 2026년 8월 Companion App 업데이트 전에 호환성을 확인해야 한다. 공식 발표 기준 Companion App 2026.8.0부터 iOS 15, watchOS 8, macOS 11이 지원 대상에서 빠지고, 마지막 호환 버전은 2026.7.1이다. Home Assistant 서버가 멀쩡해도 휴대폰 앱만 자동 업데이트되면 알림과 위치 센서가 끊길 수 있다.

## 내 환경에서 확인할 것

이번 변경은 Home Assistant Core 버전이 아니라 Apple용 Companion App의 지원 범위 변경이다. NAS Docker에서 Home Assistant 2026.7.x를 운영 중이어도 iPhone OS가 오래됐으면 별도로 영향을 받는다.

| 기기 | 2026.8부터 필요한 최소 버전 | 대응 |
|---|---:|---|
| iPhone·iPad | iOS 16.4 | 업데이트 가능 여부 확인 |
| Apple Watch | watchOS 9 | 연결된 iPhone도 같이 점검 |
| Mac | macOS 12 | 앱 대신 웹 브라우저도 가능 |

iPhone에서는 `설정 → 일반 → 정보`에서 iOS 버전을 본다. Mac은 ` → 이 Mac에 관하여`에서 확인하면 된다. iOS 15에 남아 있다면 App Store에서 앱을 지우거나 로그아웃하기 전에 현재 Companion App 버전을 기록해둔다. 공식 안내상 2026.7.1은 계속 사용할 수 있지만, 이후 보안 수정과 기능 추가는 기대하기 어렵다.

## NAS에서 웹 접속 주소를 먼저 준비한다

앱이 안 맞아도 Home Assistant 웹 화면은 계속 쓸 수 있다. 다만 외부에서 `http://공인IP:8123`으로 열어두는 방식은 피하고, 기존 HTTPS 주소를 확인한다. Nginx Proxy Manager 같은 Reverse Proxy(외부 도메인 요청을 내부 서버로 전달하는 중계 서버)를 쓰는 환경이라면 주소는 대략 아래처럼 된다.

```text
https://ha.example.com
```

NAS 내부에서만 접속한다면 `http://192.168.0.20:8123`으로 먼저 확인할 수 있다. 외부 접속은 Tailscale VPN을 붙인 뒤 NAS의 내부 주소로 연결하는 편이 포트 포워딩보다 관리하기 쉽다. 특히 Home Assistant의 위치 센서는 앱 권한과 백그라운드 동작에 의존하므로, 브라우저 접속으로 바꾸면 자동화 트리거가 일부 사라질 수 있다.

## 구형 iPhone에서 마지막 버전 유지하기

앱 자동 업데이트가 켜져 있으면 2026.8이 배포된 뒤 실수로 교체될 수 있다.

1. App Store에서 Home Assistant 앱의 현재 버전과 로그인 서버 주소를 확인한다.
2. `설정 → App Store → 앱 업데이트`를 끈다.
3. Home Assistant에서 `설정 → 기기 및 서비스 → 모바일 앱`을 열고 해당 iPhone의 마지막 센서 시간이 갱신되는지 확인한다.
4. 앱이 동작하는 동안 같은 서버 주소를 Safari 즐겨찾기에 저장한다.

테스트는 집 Wi-Fi를 끈 상태에서 한다. Safari에서 HTTPS 주소가 열리고, 대시보드 조작과 알림 수신이 되는지 각각 확인해야 한다. 브라우저는 앱의 위치 센서·푸시 동작을 완전히 대체하지 못하므로, 외출 자동화가 핵심이면 지원 OS 기기나 Android Companion App을 별도 준비하는 게 낫다.

## NAS 운영자가 놓치기 쉬운 부분

Home Assistant 컨테이너를 재시작한다고 Apple 앱 호환 문제가 해결되지는 않는다. 반대로 앱만 유지하고 NAS의 `/config` 백업을 생략하면 나중에 새 기기로 옮길 때 자동화와 모바일 앱 등록을 다시 해야 한다.

| 점검 항목 | 확인 방법 |
|---|---|
| 서버 주소 | HTTPS 도메인 또는 Tailscale 내부 주소 접속 |
| 앱 버전 | Companion App 정보 화면 |
| 모바일 등록 | 설정 → 기기 및 서비스 → 모바일 앱 |
| 백업 | NAS의 `/config`와 Home Assistant 전체 백업 |
| 위치 자동화 | 집 밖 LTE에서 위치 업데이트 테스트 |

이번 변경의 핵심은 NAS 업데이트가 아니라 클라이언트 호환성이다. iOS 16.4 이상이면 평소처럼 업데이트하면 되고, 구형 기기라면 2026.7.1을 유지하면서 웹 접속과 자동화 대체 수단을 함께 확인해야 한다.

자료: [Home Assistant Companion App Apple 플랫폼 지원 변경 공지](https://www.home-assistant.io/blog/2026/07/07/companion-app-changing-support-for-apple-platforms/), [Home Assistant Companion App 문제 해결 문서](https://companion.home-assistant.io/docs/troubleshooting/faqs/)

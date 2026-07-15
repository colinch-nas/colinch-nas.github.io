---
layout: post
title: "Synology NAS 외부접속 - DSM 7.2.2에서 Tailscale VPN 설정"
description: "Synology DS923+와 DSM 7.2.2에서 포트포워딩 없이 Tailscale VPN으로 NAS에 외부 접속하는 방법을 정리했다. 방화벽 예외와 업데이트 후 점검까지 포함한다."
date: 2026-07-16
tags: [Synology, DSM, Tailscale, VPN, NAS보안, NAS설정]
comments: true
share: true
---

![Synology NAS 외부접속과 Tailscale VPN 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)
_포트포워딩 없이 Tailscale 주소로 NAS 관리 화면에 접속하는 구성을 만들었다._

NAS 외부접속은 공유기에서 5000번, 5001번 포트를 여는 것보다 Tailscale VPN으로 시작하는 편이 안전하다. Tailscale(기기 사이에 암호화된 사설망을 만드는 VPN)을 Synology에 설치하면 카페나 LTE 환경에서도 `100.x.x.x` 주소로 DSM에 들어갈 수 있다. 2026년 7월 기준 Tailscale 공식 문서에서도 Synology Package Center 설치와 DSM 7의 제한 사항을 별도로 안내하고 있다.

## 테스트 환경

| 항목 | 설정값 |
|---|---|
| NAS | Synology DS923+ |
| DSM | 7.2.2 |
| Tailscale | Package Center 최신 패키지 |
| 공유기 | 일반 가정용 공유기, 포트포워딩 없음 |
| 클라이언트 | macOS 또는 Android Tailscale 앱 |

## Synology에 Tailscale 설치

DSM에서 **패키지 센터 → 유틸리티**로 들어가 Tailscale을 설치한다. 설치가 끝나면 Tailscale을 열고 로그인한 뒤, 같은 계정의 tailnet(내 Tailscale 기기 목록)에 NAS를 등록한다. 관리자 콘솔의 Machines 목록에 NAS가 보이면 기본 연결은 끝난다.

NAS의 Tailscale IP는 Tailscale 화면에서 확인한다. 예를 들어 `100.86.12.34`가 표시되면 외부에서 브라우저에 아래 주소를 입력한다.

```text
https://100.86.12.34:5001
```

인증서 경고가 뜨는 것은 공인 도메인 인증서가 아니라 NAS의 기본 인증서로 접속했기 때문이다. 로그인 계정은 DSM 관리자 계정과 별도로 2단계 인증을 켜두는 것이 좋다.

## DSM 방화벽을 켰다면 꼭 추가할 규칙

DSM 방화벽을 사용 중이면 Tailscale 인터페이스가 차단될 수 있다. **제어판 → 보안 → 방화벽**에서 기본 프로필에 다음 허용 규칙을 추가한다.

| 순서 | 값 |
|---|---|
| 동작 | 허용 |
| 포트 | 모두 또는 DSM 관리 포트 5001 |
| 소스 IP | `100.64.0.0/10` |
| 프로토콜 | TCP |

규칙 순서는 거부 규칙보다 위에 둔다. 저장한 뒤 휴대폰을 Wi-Fi에서 끄고 Tailscale을 켠 상태로 `https://100.x.x.x:5001`에 접속해 확인한다. 내부 Wi-Fi에서만 테스트하면 공유기 라우팅 덕분에 성공한 것처럼 보여 실수를 놓치기 쉽다.

## DSM 7에서 한 번 더 손볼 부분

Tailscale 공식 문서에는 DSM 7에서 TUN 설정을 부팅 후 유지하려면 사용자 정의 스크립트를 등록하라고 되어 있다. **제어판 → 작업 스케줄러 → 생성 → 예약된 작업 → 부팅 시 실행**으로 만들고, 사용자 정의 스크립트에 아래 내용을 넣는다. 실행 사용자는 `root`로 지정한다.

```bash
/var/packages/Tailscale/target/bin/tailscale configure-host
synosystemctl restart pkgctl-Tailscale.service
```

이 작업을 등록하지 않고 NAS를 재부팅했을 때 Tailscale은 켜져 있는데 DSM 접속만 안 되는 경우가 있었다. 패키지를 업데이트한 뒤에도 NAS를 재부팅하거나 같은 스크립트를 다시 실행해야 한다.

## 포트포워딩과 비교하면

| 방식 | 장점 | 주의점 |
|---|---|---|
| 공유기 포트포워딩 | 어떤 기기에서도 접속 가능 | 공격 표면이 커지고 스캔 대상이 됨 |
| Tailscale 직접 설치 | 외부 포트 0개, 기기별 접근 제어 | 접속 기기에도 앱 설치 필요 |
| Tailscale 서브넷 라우터 | Tailscale을 설치할 수 없는 내부 기기도 접근 | 네트워크 전체 권한을 넓히기 쉬움 |

처음에는 NAS와 내 휴대폰 두 대에만 Tailscale을 설치하는 구성이 적당하다. 프린터나 Home Assistant처럼 앱을 설치할 수 없는 장비까지 연결할 때만 서브넷 라우터를 검토한다. NAS 관리 포트를 인터넷에 직접 공개하는 것과 Tailscale Funnel로 공개하는 것은 목적이 다르므로, 관리자 화면에는 Funnel을 쓰지 않는 편이 낫다.

## 접속 전 체크리스트

- [ ] NAS에 관리자 계정 대신 별도 운영 계정을 만들었는가
- [ ] DSM 2단계 인증을 켰는가
- [ ] 공유기 포트포워딩에서 5000·5001 공개 규칙을 제거했는가
- [ ] DSM 방화벽에 `100.64.0.0/10` 규칙을 추가했는가
- [ ] LTE 환경에서 실제 접속을 확인했는가
- [ ] Tailscale 패키지 업데이트 후 재부팅 점검을 했는가

핵심은 Tailscale 설치 자체보다 외부 포트를 닫고, 접속할 기기만 tailnet에 넣는 데 있다. DS923+와 DSM 7.2.2 조합에서는 이 방식으로 DDNS와 Let's Encrypt 설정 없이 DSM 관리 화면에 접근할 수 있다. 다만 Tailscale 계정이 탈취되면 NAS 접근 권한도 함께 위험해지므로 계정 2단계 인증과 기기 목록 정리는 주기적으로 확인해야 한다.

- [Tailscale의 Synology 외부접속 공식 문서](https://tailscale.com/docs/integrations/synology)
- [Tailscale 서브넷 라우터 공식 문서](https://tailscale.com/docs/features/subnet-routers)

---
layout: post
title: "Synology 외부 접속 설정 - DSM 7.4 DDNS와 Let's Encrypt 인증서"
description: "Synology DSM 7.4에서 QuickConnect와 DDNS를 비교하고, 공유기 포트포워딩과 Let's Encrypt 인증서로 NAS 외부 접속을 구성하는 실제 설정 순서와 보안 체크리스트를 정리한다."
date: 2026-09-11
tags: [Synology, DSM, NAS설정, DDNS, NAS보안]
comments: true
share: true
---

![Synology NAS 외부 접속 네트워크 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

*외부 접속은 NAS를 인터넷에 바로 노출하는 작업이라 도메인보다 포트와 계정 보안을 함께 봐야 한다.*

Synology NAS를 휴대폰이나 회사 PC에서 쓰려면 QuickConnect 또는 DDNS(Dynamic Domain Name System, 유동 IP를 도메인에 연결하는 방식)를 선택하면 된다. 내가 확인한 DSM 7.4 기준으로 QuickConnect는 공유기 설정이 거의 필요 없지만 서드파티 Docker 서비스에는 맞지 않는다. Jellyfin이나 Vaultwarden까지 외부에서 열 생각이면 DDNS와 역방향 프록시(도메인 요청을 내부 서비스로 전달하는 중계 서버)가 더 현실적이다.

## QuickConnect와 DDNS 중 고르기

| 방식 | 설정 난이도 | 속도·호환성 | 어울리는 경우 |
|---|---:|---|---|
| QuickConnect | 낮음 | 릴레이 연결은 느릴 수 있음 | File Station, Synology Photos, Drive |
| DDNS + 포트포워딩 | 중간 | 직접 연결, 외부 서비스 확장 가능 | Docker, WebDAV, 미디어 서버 |

Synology 공식 비교 문서도 QuickConnect는 간단한 설정, DDNS는 더 높은 속도와 고급 사용자용으로 구분한다. 그래서 파일 몇 개를 확인하는 용도면 QuickConnect를 유지하고, NAS 안의 여러 웹 서비스를 공개할 때만 DDNS를 추가하는 구성이 낭비가 적다.

## DSM 7.4에서 DDNS 만들기

공유기에서 NAS의 내부 IP가 바뀌면 포트포워딩이 끊긴다. 공유기에서 NAS에 DHCP 예약을 걸어 `192.168.0.20`처럼 고정한 뒤 DSM 설정을 시작한다.

`제어판 → 외부 액세스 → DDNS → 추가`에서 서비스 제공자를 `Synology`로 고르고 호스트 이름을 정한다. 예를 들어 `myhome.synology.me`를 만들고 연결 테스트가 성공하는지 확인한다. 통신사가 CGNAT(공인 IP를 여러 가입자가 공유하는 구조)를 쓰면 DDNS가 만들어져도 외부에서 들어오지 않는다. 이때는 공유기의 WAN IP와 포털에서 확인한 공인 IP가 같은지 확인하고, 다르면 QuickConnect나 Tailscale VPN을 선택해야 한다.

## HTTPS 포트만 열고 인증서 발급하기

공유기에는 아래처럼 최소 규칙만 추가한다. 외부 포트는 기본값을 그대로 쓰지 않고 예시처럼 바꿨다.

```text
외부 TCP 443  →  NAS 192.168.0.20:5001
```

DSM에서 `제어판 → 로그인 포털 → DSM → HTTPS 포트`를 `5001`로 확인한다. `제어판 → 보안 → 인증서 → 추가 → 새 인증서 추가 → Let's Encrypt에서 인증서 얻기`로 이동해 도메인에 `myhome.synology.me`, 이메일을 입력한다. 발급이 되면 인증서를 기본값으로 지정하고 `HTTP 연결을 HTTPS로 자동 리디렉션`을 켠다. 인증서 갱신은 90일 주기라서 443 포트가 계속 NAS로 연결되는지 확인해야 한다.

DSM의 외부 주소 설정은 접속을 자동으로 만들어주는 메뉴가 아니다. 공유 링크에 어떤 호스트와 포트를 넣을지 정하는 값이라서, 포트포워딩을 별도로 하지 않으면 링크가 열리지 않는다. 이 부분을 놓고 “인증서는 발급됐는데 외부 공유가 안 된다”고 삽질하기 쉽다.

## 공개 직후 보안 체크

- 기본 `admin` 계정은 비활성화하고 별도 관리자 계정을 만든다.
- 모든 관리자 계정에 2단계 인증을 적용한다.
- `제어판 → 보안 → 보호`에서 자동 차단과 계정 보호를 켠다.
- DSM 관리 화면을 443 하나로 공개하지 말고, Docker 서비스는 별도 서브도메인과 역방향 프록시로 분리한다.
- SMB 445, SSH 22, DSM HTTP 5000은 인터넷에 포트포워딩하지 않는다.
- 외부 접속이 꼭 필요하지 않은 기간에는 포트 규칙을 끄고 QuickConnect 또는 VPN만 사용한다.

내 기준의 선택은 간단하다. Synology 앱만 쓸 때는 QuickConnect, 자체 호스팅 서비스를 붙일 때는 DDNS+HTTPS, 공유기나 통신사 제약이 있을 때는 Tailscale이다. 외부 접속은 연결 성공보다 “필요한 서비스만 열려 있는가”를 확인해야 끝난다.

참고: [Synology DSM 7.4 네트워크·외부 액세스 사양](https://www.synology.com/en-global/dsm/7.4/software_spec/network_external_access), [QuickConnect와 DDNS 비교](https://kb.synology.com/en-uk/DSM/tutorial/What_are_the_differences_between_QuickConnect_and_DDNS), [외부 액세스 보안 가이드](https://kb.synology.com/en-au/DSM/tutorial/Quick_Start_External_Access)

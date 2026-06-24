---
layout: post
title: "Synology NAS 외부 접속 설정 — DDNS와 Let's Encrypt 인증서로 HTTPS 연결"
description: "Synology DSM 7.2에서 DDNS를 설정하고 Let's Encrypt 인증서를 발급받아 외부에서 안전하게 HTTPS로 NAS에 접속하는 방법. 공유기 포트 포워딩 설정까지 포함."
date: 2026-06-08
tags: [Synology, DDNS, NAS설정, 보안]
comments: true
share: true
---

# Synology NAS 외부 접속 설정 — DDNS와 Let's Encrypt 인증서로 HTTPS 연결

![NAS DDNS HTTPS 외부 접속 설정](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=800&q=80)

집 밖에서 NAS에 접속하려면 두 가지가 필요하다. 변하는 공인 IP를 도메인으로 고정하는 DDNS, 그리고 HTTP가 아닌 HTTPS로 암호화하는 SSL 인증서. 둘 다 Synology DSM에서 공짜로 해결된다.

이 글은 [Synology 완전 정복 시리즈] 4편이다. 앞서 [Jellyfin 미디어 서버 설치]({% post_url 2026-06-07-10-00-00-037035-synology-jellyfin-media-server-setup %})까지 마쳤다면 이제 외부에서 접속할 차례다.

## 환경 정보

| 항목 | 버전/모델 |
|------|----------|
| NAS 모델 | Synology DS923+ |
| DSM 버전 | DSM 7.2.2 |
| 공유기 | ASUS RT-AX88U |
| ISP | KT (가정용) |

주의: KT, SKT, LGU+ 모두 가정용 회선에서는 80포트와 443포트를 기본 차단한다. 이 경우 표준 포트를 쓸 수 없고 비표준 포트(예: 4443)로 HTTPS를 운영해야 한다.

## DDNS 설정

공인 IP는 ISP에서 주기적으로 바꾼다. 매번 IP 확인하고 접속하는 건 불편하니 DDNS로 고정된 도메인명을 쓴다.

**제어판 → 외부 액세스 → DDNS** 탭에서 추가 버튼 클릭.

서비스 제공자로 **Synology**를 선택하면 `*.synology.me` 형식의 무료 도메인을 받을 수 있다. 예: `myhome.synology.me`

```
호스트 이름: myhome
서비스 제공자: Synology
이메일: your@email.com
```

등록 후 상태가 **정상**으로 나오면 도메인이 현재 공인 IP에 연결된 것이다. IP가 바뀌면 DSM이 자동으로 DDNS를 갱신한다.

커스텀 도메인을 가지고 있다면 서비스 제공자로 Cloudflare를 선택하고 API 토큰을 입력하면 된다.

## Let's Encrypt 인증서 발급

**제어판 → 보안 → 인증서** 탭에서 **추가** 버튼 클릭.

**Let's Encrypt에서 인증서 받기**를 선택하고 도메인명을 입력한다.

```
도메인 이름: myhome.synology.me
이메일 주소: your@email.com
```

여기서 주의할 점이 있다. Let's Encrypt 인증서를 발급받으려면 **포트 80이 외부에서 열려 있어야** 한다. 도메인 소유권 확인(HTTP-01 challenge)을 80포트로 하기 때문이다.

ISP에서 80포트를 막은 경우 인증서 발급이 실패한다. 이 경우 DNS-01 challenge를 지원하는 DDNS 제공자를 써야 하는데, Cloudflare가 지원한다.

## 공유기 포트 포워딩

NAS가 공유기 안쪽(사설 IP)에 있으면 외부에서 오는 연결을 NAS로 전달하는 포트 포워딩이 필요하다.

공유기 관리 페이지 접속 → NAT/포트 포워딩 설정:

| 외부 포트 | 내부 IP | 내부 포트 | 프로토콜 |
|---------|---------|---------|--------|
| 80 | 192.168.1.100 | 80 | TCP |
| 443 | 192.168.1.100 | 443 | TCP |
| 49001 | 192.168.1.100 | 49001 | TCP |

192.168.1.100은 NAS의 사설 IP. **제어판 → 네트워크 → 네트워크 인터페이스**에서 확인할 수 있다.

NAS의 사설 IP가 공유기 재시작 때마다 바뀌는 경우 IP를 고정해야 한다. 공유기에서 MAC 주소 기반으로 고정 IP를 할당하거나, NAS에서 직접 고정 IP를 설정하면 된다.

## HTTPS 리다이렉트 설정

**제어판 → 로그인 포털 → DSM** 탭에서 **HTTP 연결을 HTTPS로 자동 리다이렉트**를 켠다.

이렇게 하면 `http://myhome.synology.me:49000` 으로 접속해도 자동으로 `https://myhome.synology.me:49001` 로 넘어간다.

![Synology HTTPS 리다이렉트 설정](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)

## 리버스 프록시 설정 (선택)

Docker로 여러 서비스를 올렸다면 포트 번호 대신 서브도메인으로 접속하고 싶을 것이다. 리버스 프록시(Reverse Proxy: 외부 도메인 요청을 내부 서버로 전달하는 중계 서버)를 설정하면 된다.

**제어판 → 로그인 포털 → 고급 → 리버스 프록시** 탭:

```
소스:
  프로토콜: HTTPS
  호스트 이름: jellyfin.myhome.synology.me
  포트: 443

대상:
  프로토콜: HTTP
  호스트 이름: localhost
  포트: 8096
```

이렇게 하면 `https://jellyfin.myhome.synology.me` 로 접속하면 내부의 Jellyfin(8096포트)으로 연결된다.

## 삽질했던 부분

**ISP 포트 차단**: KT 가정용을 쓰는데 80포트가 막혀 있어서 Let's Encrypt 발급이 계속 실패했다. 방법이 없어 Cloudflare를 통해 DNS-01 방식으로 인증서를 받았다.

**포트 포워딩 후에도 외부 접속 안 됨**: 공유기 포트 포워딩 설정을 저장하고 바로 테스트하면 안 되는 경우가 있다. ISP에서 변경된 포트를 캐시하고 있어서 수 분 기다려야 할 때도 있다. 또는 공유기를 재시작해봤더니 해결된 경우도 있었다.

**NAS IP 고정 안 함**: 공유기 재시작했더니 NAS 사설 IP가 바뀌어 포트 포워딩이 엉뚱한 기기를 가리키게 됐다. NAS IP는 반드시 공유기에서 고정 할당해야 한다.

## 한 줄 정리

DDNS로 도메인 고정, Let's Encrypt로 인증서 발급, 포트 포워딩으로 연결 — 세 가지만 하면 외부에서도 HTTPS로 안전하게 NAS에 접속된다.

---
다음엔 [Synology NAS 랜섬웨어 대비 백업 전략 — 3-2-1 규칙 실제 구현]({% post_url 2026-06-09-10-00-00-061725-synology-ransomware-backup-321-strategy %})을 다룬다.

**참고 링크**
- [Synology DDNS 설정 공식 가이드](https://kb.synology.com/ko-kr/DSM/help/DSM/AdminCenter/connection_ddns)
- [Let's Encrypt 공식 사이트](https://letsencrypt.org)
- [Synology 리버스 프록시 설정 가이드](https://kb.synology.com/ko-kr/DSM/help/DSM/AdminCenter/connection_portal_advanced)

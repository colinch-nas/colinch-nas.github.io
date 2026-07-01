---
layout: post
title: "Synology Let's Encrypt 갱신 실패 - DSM 7.2.2에서 포트 80과 DDNS부터 확인하기"
description: "Synology DSM 7.2.2에서 Let's Encrypt 인증서 갱신이 실패할 때 DDNS, 포트 80, 공유기 포워딩, CGNAT를 순서대로 점검하는 방법."
date: 2026-07-01
tags: [Synology, DSM, DDNS, NAS보안]
comments: true
share: true
---
![Synology NAS 인증서 갱신 점검](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)

Synology NAS에서 HTTPS 접속이 갑자기 인증서 오류로 바뀌면 DSM 인증서 화면보다 공유기와 회선 상태부터 봐야 한다. 2026년 7월 1일 기준 Synology 공식 문서의 DSM 경로는 **제어판 → 보안 → 인증서**이고, Let's Encrypt 검증은 외부에서 NAS에 닿을 수 있어야 성공한다. 나는 DSM 7.2.2에서 인증서 재발급 버튼만 여러 번 눌렀다가 Rate Limit만 걱정하게 됐다.

문제 상황은 보통 비슷하다. `xxx.synology.me` 주소로는 어제까지 접속됐는데 오늘은 "인증서가 만료됨"이 뜬다. DSM에서는 갱신 실패 알림만 남고 원인은 자세히 안 보인다. 이때 NAS 내부의 시계, DDNS 주소, 80번 포트, 공유기 포워딩을 나눠서 확인하면 삽질이 줄어든다.

## DSM에서 확인할 항목

DSM 7.2.2 기준으로 **제어판 → 외부 액세스 → DDNS**에 들어가 상태가 `정상`인지 확인한다. 상태가 실패라면 인증서 문제가 아니라 도메인이 현재 공인 IP를 못 가리키는 문제다. 수정 후 바로 인증서를 누르지 말고 5~10분 기다렸다가 외부 LTE망에서 도메인을 열어본다.

NAS 시간이 틀어져도 인증서 검증이 꼬인다. **제어판 → 지역 옵션 → 시간**에서 NTP 서버 동기화를 켠다. 나는 공유기 DNS를 바꾼 뒤 NAS가 NTP를 못 잡아서 인증서가 실패한 적이 있었다.

```text
NTP 서버: time.google.com
대체 서버: pool.ntp.org
시간대: Asia/Seoul
```

## 포트 80은 5000으로 넘기면 안 된다

Let's Encrypt HTTP 검증은 외부의 80번 요청을 확인한다. 공유기에서 외부 80번을 NAS의 5000번 DSM 포트로 넘기면 될 것 같지만, 이 설정은 자주 실패한다. 안전하게는 외부 80 → NAS 80, 외부 443 → NAS 443으로 맞춘 뒤 DSM의 인증서 발급을 시도한다.

공유기 포트 포워딩 예시는 이렇게 잡는다.

```text
TCP 80  -> NAS 내부 IP 192.168.0.20:80
TCP 443 -> NAS 내부 IP 192.168.0.20:443
```

DSM 로그인 포트 5000, 5001을 그대로 외부에 여는 건 추천하지 않는다. 인증서 발급과 실제 서비스 공개를 분리해서 생각해야 한다. DSM 관리 화면은 Tailscale VPN이나 내부망에서만 쓰고, 외부 공개는 역방향 프록시(도메인 요청을 내부 서비스로 넘기는 중계 설정)로 최소화하는 편이 낫다.

## CGNAT와 ISP 차단 확인

공유기 WAN IP가 `100.64.x.x`, `10.x.x.x`, `172.16~31.x.x`, `192.168.x.x`라면 CGNAT 가능성이 높다. 이 상태에서는 DDNS가 공유기까지 오지 못하고, 포트 포워딩도 동작하지 않는다. 통신사에 공인 IP 옵션을 요청하거나 포트 포워딩 없는 Tailscale, Cloudflare Tunnel 쪽으로 바꾸는 게 현실적이다.

외부에서 80번이 열렸는지는 휴대폰 LTE에서 확인한다. 집 Wi-Fi로 도메인을 열면 NAT 루프백 지원 여부 때문에 결과가 헷갈린다.

```bash
curl -I http://내도메인.synology.me
curl -I https://내도메인.synology.me
```

## 갱신 전 체크 순서

인증서 만료가 임박했을 때는 재발급 버튼을 연타하지 않는다. DDNS 정상, NAS 시간 동기화, 공유기 80/443 포워딩, 외부망 접속 확인 순서로 본 뒤 **제어판 → 보안 → 인증서 → 추가 → Let's Encrypt에서 인증서 얻기**로 진행한다.

짧게 정리하면 DSM 인증서 갱신 실패는 DSM 자체 오류보다 외부 검증 경로 문제인 경우가 많다. `DDNS 정상`, `외부 80번 도달`, `CGNAT 아님`, `NAS 시간 정상` 네 가지만 통과하면 대부분 다시 발급된다.

출처: [Synology HTTPS 및 인증서 공식 문서](https://kb.synology.com/DSM/tutorial/How_to_enable_HTTPS_and_create_a_certificate_signing_request_on_your_Synology_NAS), [Let's Encrypt 커뮤니티 Synology 포트 80 안내](https://community.letsencrypt.org/t/certificate-request-keeps-failing-on-synology-nas/239032)

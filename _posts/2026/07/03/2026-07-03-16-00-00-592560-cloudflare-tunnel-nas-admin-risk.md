---
layout: post
title: "Cloudflare Tunnel NAS 공개 - 관리자 화면까지 열기 전에 막아야 할 것"
description: "Cloudflare Tunnel로 NAS 서비스를 외부 공개할 때 관리자 페이지, 인증, 내부 포트 분리와 접근 정책을 점검하는 방법이다."
date: 2026-07-03
tags: [CloudflareTunnel, NAS보안, Docker, HomeLab, NAS]
comments: true
share: true
---
![Cloudflare Tunnel NAS 보안 점검](https://images.unsplash.com/photo-1550751827-4bd374c3f58b?w=800&q=80)

이 그림에서는 외부에서 접속이 된다는 사실보다 어디까지 열렸는지가 더 중요하다는 점을 봐야 한다.

Cloudflare Tunnel은 NAS 서비스를 밖에서 쓰기 편하게 만들지만, 관리자 화면까지 같은 규칙으로 열면 위험해진다. Nextcloud, Immich, Vaultwarden 같은 사용자 서비스와 DSM, Proxmox, 라우터 관리 페이지는 공개 기준이 달라야 한다. 나는 처음에 테스트용 터널을 만들고 내부 포트를 넓게 열었다가, 접속 로그를 보고 바로 규칙을 쪼갰다.

내 기준 환경은 Synology NAS, Docker 컨테이너 6개, Cloudflare Tunnel, 내부망 192.168.0.0/24다. 포트포워딩은 쓰지 않았지만, 터널이 사실상 외부 입구가 되므로 보안 점검은 필요하다.

| 대상 | 공개 여부 | 권장 처리 |
|---|---|---|
| Nextcloud | 조건부 공개 | Cloudflare Access 또는 앱 로그인 |
| Immich | 조건부 공개 | 가족 계정만 허용 |
| Vaultwarden | 매우 신중 | 2FA, 관리자 경로 제한 |
| NAS 관리자 | 비공개 권장 | VPN 또는 내부망 전용 |
| Proxmox | 비공개 권장 | Tailscale, WireGuard |

터널 설정에서 흔한 실수는 내부 호스트를 넓게 잡는 것이다.

```yaml
ingress:
  - hostname: photos.example.com
    service: http://192.168.0.20:2283
  - hostname: nas.example.com
    service: https://192.168.0.20:5001
  - service: http_status:404
```

위 설정에서 `nas.example.com`이 DSM 관리자라면 다시 생각해야 한다. 공개가 꼭 필요하다면 최소한 Access 정책, 2단계 인증, IP 제한, 관리자 계정 이름 변경을 같이 봐야 한다. 그래도 나는 관리자 화면은 터널보다 VPN 쪽이 낫다고 본다.

점검 순서는 이렇게 잡았다.

| 단계 | 확인 |
|---|---|
| 1 | 공개할 서비스 목록을 사용자 앱과 관리자 앱으로 분리 |
| 2 | 각 hostname이 정확한 내부 포트 하나만 가리키는지 확인 |
| 3 | 기본 catch-all을 `404`로 둠 |
| 4 | 관리자 서비스는 Tunnel 대신 VPN으로 접근 |
| 5 | 접속 로그와 실패 로그를 주 1회 확인 |

또 하나의 삽질은 테스트용 hostname을 지우지 않은 것이다. `test-nas.example.com`을 만들어놓고 잊어버리면 문서에도 안 남은 입구가 생긴다. 터널이 편한 만큼 오래된 규칙 청소가 중요하다.

짧게 정리하면 이렇다. Cloudflare Tunnel은 포트포워딩보다 깔끔하지만 자동으로 안전해지는 건 아니다. NAS 관리자 화면, Proxmox, 라우터 설정 페이지는 사용자 서비스와 분리하고, 외부 공개보다 VPN 접근을 기본값으로 두는 편이 안정적이다.

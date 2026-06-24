---
layout: post
title: "Cloudflare Tunnel — 포트 포워딩 없이 집 서버 서비스 외부 공개하기"
description: "Cloudflare Tunnel(cloudflared)을 사용해 포트 포워딩과 공인 IP 없이 홈서버 서비스를 외부에 안전하게 공개하는 방법. Docker 설치와 도메인 연결 설정 포함."
date: 2026-06-20
tags: [CloudflareTunnel, NAS보안, 홈서버, 자체호스팅]
comments: true
share: true
---

# Cloudflare Tunnel — 포트 포워딩 없이 집 서버 서비스 외부 공개하기

![Cloudflare Tunnel 홈서버 외부 공개](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=800&q=80)

포트 포워딩을 열면 보안 위험이 생기고, CGNAT 환경이면 아예 불가능하다. Cloudflare Tunnel은 집 서버에서 Cloudflare로 아웃바운드 연결을 맺어두고, 외부 요청을 Cloudflare가 이 터널로 전달하는 방식이다. 집 서버에서 어떤 인바운드 포트도 열 필요가 없다.

## Cloudflare Tunnel의 동작 원리

```
사용자 브라우저
  ↓ HTTPS
Cloudflare 엣지 서버
  ↓ 암호화된 터널
cloudflared (집 서버에 설치)
  ↓ 로컬
Nextcloud / Vaultwarden / Immich (내부 서비스)
```

집 서버에 설치된 `cloudflared`가 Cloudflare로 터널을 맺고 있다. 사용자가 도메인에 접속하면 Cloudflare가 이 터널을 통해 내부 서비스로 요청을 전달한다.

## 사전 조건

- Cloudflare 계정 (무료)
- Cloudflare에 등록된 도메인 (DNS를 Cloudflare로 관리)

도메인은 가비아, Namecheap 같은 곳에서 사도 되고, Cloudflare에서 직접 살 수도 있다. 도메인 구매 후 DNS 네임서버를 Cloudflare 것으로 변경해야 한다.

## 환경 정보

| 항목 | 버전/모델 |
|------|----------|
| 홈서버 | Synology DS923+ |
| cloudflared 버전 | 2024.x |
| 도메인 | example.com (Cloudflare에서 관리) |

## Cloudflare Tunnel 생성

[Cloudflare Zero Trust 대시보드](https://one.dash.cloudflare.com) → Networks → Tunnels → Create a tunnel.

```
터널 이름: homelab
커넥터: Docker
```

Docker 설치 명령이 생성된다:

```bash
sudo docker run -d \
  --name cloudflared \
  --restart unless-stopped \
  cloudflare/cloudflared:latest \
  tunnel --no-autoupdate run \
  --token YOUR_TUNNEL_TOKEN
```

docker-compose로 관리하려면:

```yaml
# /volume1/docker/cloudflared/docker-compose.yml
version: "3.9"
services:
  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared
    restart: unless-stopped
    command: tunnel --no-autoupdate run
    environment:
      - TUNNEL_TOKEN=YOUR_TUNNEL_TOKEN
```

```bash
cd /volume1/docker/cloudflared
sudo docker compose up -d
```

대시보드에서 터널이 **Healthy** 상태가 되면 연결된 것이다.

## 서비스 공개 설정 (Public Hostname)

Cloudflare Zero Trust 대시보드 → Tunnels → 생성한 터널 → Configure → Public Hostname.

```
서브도메인: cloud
도메인: example.com
경로: (비워두면 루트)
서비스 유형: HTTP
URL: 192.168.1.100:8080    # Nextcloud 내부 주소
```

이렇게 설정하면 `https://cloud.example.com`으로 접속하면 Nextcloud로 연결된다.

여러 서비스를 각각 다른 서브도메인으로 추가할 수 있다:

```
cloud.example.com → 192.168.1.100:8080 (Nextcloud)
vault.example.com → 192.168.1.100:8070 (Vaultwarden)
photos.example.com → 192.168.1.100:2283 (Immich)
```

![Cloudflare Tunnel Public Hostname 설정](https://images.unsplash.com/photo-1451187580094-2cfb76b7c9b7?w=800&q=80)

## Access 정책으로 접근 제한 (선택)

외부에 서비스를 공개하면서 특정 사람만 접근하게 하려면 Cloudflare Access를 쓸 수 있다.

Networks → Tunnels → Access 설정에서:
```
이메일 허용 목록: your@email.com
인증 방법: One-time PIN (이메일로 인증 코드 발송)
```

이렇게 하면 서비스에 접속하려는 사람은 먼저 이메일 인증을 통과해야 한다. Vaultwarden처럼 외부에 열고 싶지만 이중 보호가 필요한 서비스에 적합하다.

## DDNS + 포트 포워딩 vs Cloudflare Tunnel

| 항목 | DDNS + 포트 포워딩 | Cloudflare Tunnel |
|------|-----------------|-----------------|
| 포트 오픈 필요 | 있음 | 없음 |
| CGNAT 가능 | 불가 | 가능 |
| SSL 인증서 | Let's Encrypt 필요 | 자동 |
| DDoS 방어 | 없음 | Cloudflare 제공 |
| 레이턴시 | 낮음 | 약간 높음 |
| 비용 | 무료 | 무료 (기본) |

## 삽질했던 부분

**WebSocket 지원 문제**: Immich나 Vaultwarden처럼 WebSocket을 쓰는 서비스는 Cloudflare 설정에서 HTTP/2나 WebSocket이 활성화돼 있어야 한다. 기본으로 켜져 있긴 한데, 안 된다면 Network → WebSocket 설정 확인.

**Nextcloud의 신뢰 도메인**: Cloudflare Tunnel로 도메인을 바꾸면 Nextcloud에서 새 도메인을 신뢰 도메인에 추가해야 한다.

```bash
sudo docker exec -it nextcloud php /var/www/html/occ config:system:set trusted_domains 2 --value="cloud.example.com"
```

**대역폭 제한**: Cloudflare 무료 플랜에서 대용량 파일 전송(GB 단위)을 많이 하면 속도 제한이 생길 수 있다. Nextcloud 파일 동기화 클라이언트는 영향을 받을 수 있다.

## 한 줄 정리

Cloudflare Tunnel은 포트 포워딩 없이 홈서버 서비스를 외부에 안전하게 공개하는 가장 현대적인 방법이다. CGNAT 환경이거나 공유기 설정이 복잡한 경우 특히 유용하다.

---
다음엔 [QNAP NAS 백업 전략 — 3-2-1 규칙으로 데이터 손실 없이 관리하는 법]({% post_url 2026-06-21-10-00-00-209865-qnap-nas-backup-321-rule-implementation %})을 다룬다.

**참고 링크**
- [Cloudflare Tunnel 공식 문서](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/)
- [Cloudflare Zero Trust 대시보드](https://one.dash.cloudflare.com)
- [cloudflared Docker Hub](https://hub.docker.com/r/cloudflare/cloudflared)

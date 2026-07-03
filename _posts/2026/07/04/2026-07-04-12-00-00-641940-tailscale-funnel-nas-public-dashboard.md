---
layout: post
title: "Tailscale Funnel - NAS 상태 대시보드만 HTTPS로 공개하기"
description: "Tailscale Funnel과 Docker로 NAS의 작은 읽기 전용 대시보드만 외부에 공개하고, 관리자 화면 노출을 피하는 설정 순서다."
date: 2026-07-04
tags: [Tailscale, Docker, NAS보안, 홈서버]
comments: true
share: true
---

![NAS에서 작은 상태 대시보드를 공개하기 전 네트워크 구성을 점검하는 화면](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)
이 그림에서는 외부에 열 서비스와 내부에만 둘 관리 화면을 분리해서 봐야 한다.

Tailscale Funnel은 NAS 전체를 공개하는 도구가 아니라, 공개해도 되는 작은 웹 서비스 하나만 HTTPS로 내보낼 때 맞다. 2026년 7월 4일 기준 Tailscale 공식 문서는 Funnel이 `*.ts.net` 주소, HTTPS 인증서, 공개 DNS를 자동으로 처리한다고 설명한다. 나는 DSM 관리 화면이나 Vaultwarden은 절대 올리지 않고, `Uptime Kuma` 요약이나 정적 HTML 상태판처럼 읽기 전용 페이지만 대상으로 잡는다.

내 기준 환경은 Synology DS923+, DSM 7.2.2, Container Manager, Tailscale CLI 1.98 계열이다. QNAP이나 Ubuntu 홈서버도 흐름은 같다. 핵심은 NAS 앱 포트를 바로 열지 않고, 별도 컨테이너가 만든 3000번 대시보드만 Funnel로 연결하는 것이다.

| 항목 | 공개 여부 | 이유 |
|---|---:|---|
| DSM 5000/5001 | 아니오 | 관리자 로그인 화면이다 |
| Portainer 9443 | 아니오 | 컨테이너 제어권이 있다 |
| Vaultwarden | 신중 | 2FA와 추가 접근 제어가 필요하다 |
| 정적 상태 대시보드 3000 | 가능 | 읽기 전용이면 피해 범위가 작다 |

## 작은 대시보드 컨테이너를 만든다

예시는 nginx로 정적 HTML만 띄운다. 실제로는 Uptime Kuma의 public status page를 프록시해도 되지만, 처음 테스트는 더 단순해야 삽질이 줄어든다.

`/volume1/docker/public-status/docker-compose.yml` 파일을 만든다.

```yaml
services:
  status-page:
    image: nginx:1.27-alpine
    container_name: public-status
    restart: unless-stopped
    ports:
      - "127.0.0.1:3000:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro
```

여기서 중요한 건 `127.0.0.1:3000:80`이다. 처음엔 `3000:80`으로 열었다가 같은 LAN의 다른 기기에서도 바로 접근되는 걸 보고 바꿨다. Funnel만 통로로 쓸 거면 NAS 전체 인터페이스에 포트를 열 필요가 없다.

테스트용 페이지를 만든 뒤 컨테이너를 올린다.

```bash
mkdir -p /volume1/docker/public-status/html
printf '<h1>NAS OK</h1><p>public read-only status</p>' > /volume1/docker/public-status/html/index.html
cd /volume1/docker/public-status
docker compose up -d
curl http://127.0.0.1:3000
```

## Funnel은 한 포트만 연결한다

NAS에 Tailscale이 이미 로그인돼 있어야 한다. Tailnet 관리자 화면에서 Funnel 권한을 허용하지 않았다면 명령 실행 중 승인 화면이 뜬다.

```bash
sudo tailscale funnel 3000
```

정상이라면 `https://nas-name.tailnet-name.ts.net` 형태의 주소가 나온다. 휴대폰 LTE로 열어 보고, 같은 주소 뒤에 `/admin`, `:5001`, `:9443` 같은 내부 관리 경로가 열리지 않는지 확인한다. Funnel은 포트 하나를 공개하는 방식이지만, 내가 만든 웹 서버가 내부 프록시를 잘못 물고 있으면 의도치 않은 화면이 보일 수 있다.

## 공개 전 체크리스트

- 페이지에 내부 IP, 장비명, 사용자 이름을 그대로 쓰지 않는다.
- `docker ps`로 3000번 외 새 포트가 열렸는지 확인한다.
- 공유기 포트포워딩은 추가하지 않는다.
- 공개 주소는 인증서 투명성 로그와 DNS에 남는다고 보고 다룬다.
- 필요가 끝나면 `sudo tailscale funnel off`로 끈다.

짧게 정리하면 이렇다. Tailscale Funnel은 Cloudflare Tunnel이나 공유기 포트포워딩을 대체할 수 있지만, 공개 대상은 더 좁게 잡아야 한다. NAS 운영에서는 "외부에서 보여도 되는 읽기 전용 화면"과 "로그인하면 제어권이 생기는 관리자 화면"을 분리해야 한다. DSM, Portainer, 비밀번호 서버는 내부망이나 Tailscale Serve에 두고, Funnel에는 작은 상태판 하나만 올리는 쪽이 실수했을 때 피해가 작다.

출처: [Tailscale Funnel 공식 문서](https://tailscale.com/docs/features/tailscale-funnel), [Tailscale Serve 공식 문서](https://tailscale.com/docs/features/tailscale-serve), [Tailscale Docker 컨테이너 문서](https://tailscale.com/docs/features/containers/docker)

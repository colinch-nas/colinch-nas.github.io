---
layout: post
title: "Nginx Proxy Manager로 도메인·HTTPS 한 번에 관리하기"
description: "Nginx Proxy Manager(NPM)를 Docker로 설치하고 홈서버에서 운영 중인 여러 서비스에 각각 도메인과 Let's Encrypt HTTPS 인증서를 쉽게 연결하는 방법."
date: 2026-06-23
tags: [NginxProxyManager, NAS보안, 자체호스팅, DDNS]
comments: true
share: true
---
![Nginx Proxy Manager 리버스 프록시 설정](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=800&q=80)

홈서버에 서비스가 여러 개 늘어나면 포트 번호로 관리하는 게 한계에 온다. `http://192.168.1.100:8096`(Jellyfin), `http://192.168.1.100:8080`(Nextcloud) 이런 식으로 외우기도 힘들고, 각각 HTTPS 설정하는 것도 번거롭다. Nginx Proxy Manager는 웹 UI로 리버스 프록시와 SSL 인증서를 한 번에 관리한다.

Synology에도 리버스 프록시 기능이 내장돼 있는데, 여러 서버에 걸쳐 서비스를 운영하거나 Synology가 아닌 환경(TrueNAS, Proxmox VM 등)에서는 NPM이 더 유연하다.

## 환경 정보

| 항목 | 버전/모델 |
|------|----------|
| 설치 위치 | Proxmox Ubuntu VM |
| Nginx Proxy Manager 버전 | 2.11.x |
| 도메인 | example.com (Cloudflare DNS) |

## 설치

```yaml
# /opt/nginx-proxy-manager/docker-compose.yml
version: "3.9"
services:
  app:
    image: jc21/nginx-proxy-manager:latest
    container_name: nginx-proxy-manager
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
      - "81:81"    # 관리 UI 포트
    volumes:
      - /opt/nginx-proxy-manager/data:/data
      - /opt/nginx-proxy-manager/letsencrypt:/etc/letsencrypt
```

```bash
cd /opt/nginx-proxy-manager
sudo docker compose up -d
```

관리 UI 접속: `http://서버-IP:81`

기본 로그인:
```
이메일: admin@example.com
비밀번호: changeme
```

첫 로그인 후 이메일과 비밀번호를 반드시 변경한다.

## Proxy Host 추가

**Hosts → Proxy Hosts → Add Proxy Host**

Jellyfin 예시:

```
Details 탭:
  Domain Names: jellyfin.example.com
  Scheme: http
  Forward Hostname: 192.168.1.100
  Forward Port: 8096
  Websockets Support: 활성화

SSL 탭:
  SSL Certificate: 새 인증서 요청
  Force SSL: 활성화
  HTTP/2 Support: 활성화
```

저장하면 NPM이 자동으로 Let's Encrypt에서 인증서를 발급받는다. 발급에 1~2분 걸린다.

같은 방식으로 다른 서비스도 추가:

```
Nextcloud:
  Domain: cloud.example.com → 192.168.1.100:8080

Vaultwarden:
  Domain: vault.example.com → 192.168.1.100:8070

Immich:
  Domain: photos.example.com → 192.168.1.100:2283
  Websockets: 활성화

Home Assistant:
  Domain: ha.example.com → 192.168.1.101:8123
```

## 와일드카드 인증서 설정 (선택)

서브도메인이 많으면 각각 인증서 발급 대신 와일드카드 인증서(`*.example.com`) 하나로 모두 커버할 수 있다. 단, 와일드카드 인증서는 DNS-01 Challenge가 필요해서 Cloudflare API를 사용해야 한다.

**SSL Certificates → Add SSL Certificate → Let's Encrypt**:

```
도메인: *.example.com, example.com
Use DNS Challenge: 활성화
DNS Provider: Cloudflare
API Token: (Cloudflare에서 생성한 API 토큰)
```

Cloudflare API 토큰 생성:
- Cloudflare 대시보드 → My Profile → API Tokens
- Create Token → Edit zone DNS 템플릿 사용
- Zone: 해당 도메인 선택

![Nginx Proxy Manager 관리 UI](https://images.unsplash.com/photo-1451187580094-2cfb76b7c9b7?w=800&q=80)

## 접근 제한 설정

**Access Lists** 기능으로 특정 IP만 허용하거나 HTTP Basic Auth를 추가할 수 있다.

내부 네트워크에서만 접속 가능하게 하는 예시:

```
Allow: 192.168.1.0/24
Deny: all
```

## 인증서 자동 갱신

Let's Encrypt 인증서는 90일 유효기간이다. NPM이 자동으로 갱신한다. 갱신 실패 시 NPM 로그에서 확인:

```bash
sudo docker compose logs app | grep -i "cert\|ssl\|error"
```

## 삽질했던 부분

**포트 80/443 충돌**: NPM이 80과 443 포트를 점유하기 때문에 같은 서버에서 다른 서비스가 이 포트를 쓰면 충돌한다. NPM을 제일 먼저 설치하고 다른 서비스는 다른 포트를 쓰게 해야 한다.

**Let's Encrypt Rate Limit**: 같은 도메인에 너무 자주 인증서를 발급받으면 Rate Limit에 걸린다. 일주일에 5개 한도. 테스트할 때는 Staging 인증서를 쓰고 최종 확인 후에 실제 인증서를 발급받는 게 낫다.

**Nextcloud HTTPS 리다이렉트 루프**: Nextcloud 뒤에 리버스 프록시를 두면 무한 리다이렉트 루프가 생길 수 있다. NPM에서 Nextcloud로 전달할 때 `X-Forwarded-Proto: https` 헤더를 추가해야 한다.

NPM 고급 설정에서:
```nginx
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Host $host;
```

Nextcloud `config.php`에서:
```php
'overwriteprotocol' => 'https',
'trusted_proxies' => ['172.16.0.0/12'],
```

## 한 줄 정리

Nginx Proxy Manager는 홈서버에서 여러 서비스를 도메인과 HTTPS로 관리하는 가장 쉬운 방법이다. 와일드카드 인증서 하나면 서브도메인을 몇 개든 추가할 수 있다.

---
다음엔 [NAS 랜섬웨어 대비 보안 설정 체크리스트 2026 — Synology 기준]({% post_url 2026-06-24-10-00-00-246900-nas-ransomware-security-checklist-2026 %})을 다룬다.

**참고 링크**
- [Nginx Proxy Manager 공식 사이트](https://nginxproxymanager.com)
- [Nginx Proxy Manager GitHub](https://github.com/NginxProxyManager/nginx-proxy-manager)
- [Let's Encrypt Rate Limits](https://letsencrypt.org/docs/rate-limits/)

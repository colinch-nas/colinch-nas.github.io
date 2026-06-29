---
layout: post
title: "WireGuard VPN — Synology NAS에 Docker로 완전 자립형 원격접속 구축하기"
description: "Tailscale 계정 없이 WG-Easy Docker 컨테이너로 Synology NAS에 WireGuard VPN 서버를 직접 설치하는 방법. DSM 7.2 기준, wg-easy v15 설정부터 스마트폰 연결까지."
date: 2026-06-29
tags: [Synology, VPN, WireGuard, Docker, NAS보안]
comments: true
share: true
---

![WireGuard VPN Synology NAS 원격접속 보안 설정](https://images.unsplash.com/photo-1614064641938-3bbee52942c7?w=800&q=80)

WireGuard를 NAS에 직접 설치하면 Tailscale 계정도, ZeroTier 서버도 필요 없다. 내 집 NAS가 VPN 서버 자체가 된다.

## 환경 정보

| 항목 | 버전/모델 |
|------|----------|
| NAS 모델 | Synology DS923+ |
| DSM 버전 | DSM 7.2.2-72806 |
| Docker | Container Manager 21.10 (DSM 패키지) |
| WG-Easy 버전 | wg-easy v15 |

---

## Tailscale 쓰면 되는데 왜 직접 설치하나

[Tailscale](https://tailscale.com)은 정말 편하다. 설치하면 5분 안에 된다. 근데 Tailscale 계정이 있어야 하고, 무료 플랜은 기기 3대 제한이 생겼다(2025년 기준). 더 결정적인 건 Tailscale 서버가 죽으면 내 VPN도 안 된다는 것. 코디네이터(coordinator) 서버에 의존하는 구조라서.

WireGuard를 직접 설치하면:
- 서드파티 계정 없음
- 기기 수 제한 없음
- Tailscale 서버 장애 무관
- 단, 공유기 포트 포워딩 필요

포트 포워딩이 막혀 있거나 이미 Tailscale로 만족한다면 굳이 바꿀 필요는 없다. 포트 포워딩 없이 외부 서비스를 열고 싶으면 예전에 정리한 {% post_url 2026-06-20-10-00-00-197520-cloudflare-tunnel-no-port-forwarding %}이 더 맞다.

---

## 왜 DSM에 WireGuard를 직접 못 까는가

Synology DSM 7.2.x는 Linux 커널 4.4.302+를 쓴다. WireGuard는 Linux 5.6부터 커널에 내장됐다. 커널 버전이 낮아서 Synology 공식 패키지로는 WireGuard를 쓸 수 없다. Synology도 공식 지원 계획이 없다고 포럼에서 답변한 적 있다.

해결책은 **userspace WireGuard 구현체를 Docker로 돌리는 것**이다. WG-Easy가 바로 그 역할이다. 커널 모듈 없이 순수 소프트웨어로 WireGuard 프로토콜을 구현해서 컨테이너 안에서 실행된다.

---

## 설치 전 준비

### 공유기 포트 포워딩

WireGuard는 UDP 51820 포트를 쓴다. 공유기에서 아래를 설정한다.

| 항목 | 값 |
|------|-----|
| 프로토콜 | UDP |
| 외부 포트 | 51820 |
| 내부 IP | NAS IP (예: 192.168.1.100) |
| 내부 포트 | 51820 |

WG-Easy 웹 UI는 51821 포트인데, 이건 외부에 열 필요 없다. 로컬에서만 접근하면 된다.

### NAS IP 확인

```bash
# DSM 터미널(SSH)에서 확인
ip addr show | grep "inet "
```

NAS IP를 고정 IP로 설정해두는 게 좋다. DSM → 네트워크 → 네트워크 인터페이스 → 수동 설정.

---

## WG-Easy 설치 (Docker Compose)

DSM Container Manager에서 SSH로 접속해 작업하거나, Container Manager 프로젝트 기능을 써도 된다. SSH가 편하다.

### 1. 폴더 구조 생성

```bash
mkdir -p /volume1/docker/wg-easy
```

### 2. docker-compose.yml 작성

```yaml
version: "3.8"

services:
  wg-easy:
    image: ghcr.io/wg-easy/wg-easy:15
    container_name: wg-easy
    environment:
      - LANG=ko
      - WG_HOST=내_공인IP_또는_DDNS주소
      - PASSWORD_HASH=$$2a$$12$$해시값  # bcrypt 해시
      - WG_PORT=51820
      - WG_DEFAULT_DNS=1.1.1.1
      - UI_TRAFFIC_STATS=true
      - UI_CHART_TYPE=1
    volumes:
      - /volume1/docker/wg-easy:/etc/wireguard
    ports:
      - "51820:51820/udp"
      - "51821:51821/tcp"
    cap_add:
      - NET_ADMIN
      - SYS_MODULE
    sysctls:
      - net.ipv4.ip_forward=1
      - net.ipv4.conf.all.src_valid_mark=1
    restart: unless-stopped
```

`WG_HOST`에는 외부에서 접속할 공인 IP 또는 DDNS 주소를 넣는다. 유동 IP라면 Synology DDNS(myds.me)나 DuckDNS 주소를 쓰면 된다.

### 3. 비밀번호 해시 생성

v15부터 평문 비밀번호(`PASSWORD=`)를 더 이상 쓸 수 없다. bcrypt 해시를 생성해야 한다.

```bash
# Docker로 해시 생성
docker run --rm -it ghcr.io/wg-easy/wg-easy:15 wgpw '내비밀번호'
```

출력 결과가 `$2a$12$xxx...` 형태다. `docker-compose.yml`의 `PASSWORD_HASH` 값에 넣을 때 `$` 앞에 `$`를 하나 더 붙인다(YAML에서 `$` 이스케이프). 즉 `$$2a$$12$$...` 형태가 된다.

### 4. 컨테이너 실행

```bash
cd /volume1/docker/wg-easy
docker compose up -d
```

### 5. 웹 UI 접속 확인

브라우저에서 `http://NAS-IP:51821`로 접속. 비밀번호 입력 후 대시보드가 나오면 성공이다.

![WireGuard WG-Easy 웹 대시보드 클라이언트 관리 화면](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=800&q=80)

---

## 클라이언트 추가 (스마트폰·PC)

### 웹 UI에서 추가

1. 웹 UI 우측 상단 **+ New** 클릭
2. 클라이언트 이름 입력 (예: `iPhone-KS`, `MacBook`)
3. QR 코드 생성됨

### 스마트폰 연결

1. App Store / Play Store에서 **WireGuard** 앱 설치
2. 앱 → `+` → QR 코드 스캔
3. 토글 켜기 → 연결 확인

스마트폰에서 모바일 데이터로 전환한 뒤 NAS 웹 UI(`http://NAS-IP:5001`)가 열리면 VPN이 제대로 터널링되는 것이다.

---

## 삽질했던 부분

### v14에서 v15로 올리면 설정 깨짐

이미 wg-easy v14를 쓰고 있다면 v15는 설정 포맷이 달라서 그냥 올리면 안 된다. `docker-compose.yml`을 v15 방식으로 완전히 새로 작성해야 한다. 특히 `PASSWORD_HASH` 환경변수가 v14의 `PASSWORD`를 대체한다. 기존 클라이언트 설정(`/etc/wireguard/wg0.json`)은 그대로 마이그레이션된다.

### DSM 방화벽 막히는 문제

DSM 제어판 → 보안 → 방화벽을 켜놓으면 51820(UDP), 51821(TCP)을 별도로 허용해줘야 한다. 컨테이너는 켜졌는데 접속이 안 되면 방화벽부터 확인하자.

### `net.ipv4.ip_forward` 권한 오류

Synology는 Docker 컨테이너의 sysctls 설정이 기본적으로 제한된다. `docker compose up` 시 아래 오류가 나면:

```
failed to set net.ipv4.ip_forward: permission denied
```

DSM SSH에서 먼저 수동으로 켜준다.

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

그 다음 컨테이너를 다시 실행하면 된다.

### 유동 IP + DDNS 조합에서 연결 끊김

공인 IP가 바뀌면 WireGuard 터널이 끊긴다. `WG_HOST`에 DDNS 주소를 쓰는 게 맞는데, WireGuard 클라이언트는 최초 설정한 IP를 그대로 캐싱한다. IP 바뀐 뒤엔 클라이언트 설정 파일에서 Endpoint를 수동으로 업데이트하거나, 클라이언트를 지웠다 다시 추가해야 한다.

완전히 자동화하려면 DuckDNS + 자동갱신 스크립트를 조합하는 게 낫다. 이 부분은 다음에 따로 정리한다.

---

## Tailscale vs WireGuard(WG-Easy) 비교

| 항목 | Tailscale | WireGuard (WG-Easy) |
|------|-----------|---------------------|
| 설정 난이도 | 매우 쉬움 | 중간 (포트 포워딩 필요) |
| 서드파티 의존 | Tailscale 계정·서버 | 없음 (완전 자립) |
| 무료 기기 수 | 3대 (2025년 이후) | 무제한 |
| 속도 | 빠름 | 빠름 (WireGuard 프로토콜 동일) |
| 서버 장애 영향 | 받음 | 없음 |
| 포트 포워딩 | 불필요 | 필수 |

포트 포워딩이 안 되는 환경(회사 네트워크, CGNAT)이라면 Tailscale이나 {% post_url 2026-06-18-10-00-00-172830-tailscale-vpn-home-server-remote-access %}을 참고하자.

---

WG-Easy 웹 UI에서 클라이언트별 트래픽 통계를 실시간으로 볼 수 있다. 어떤 기기가 VPN을 쓰고 있는지, 얼마나 데이터를 주고받는지 한눈에 보인다는 게 생각보다 유용하다.

다음엔 QNAP NAS에서 Docker로 서비스를 운영하는 방법을 정리한다.

**참고 링크**
- [WG-Easy GitHub](https://github.com/wg-easy/wg-easy)
- [WireGuard 공식 사이트](https://www.wireguard.com)
- [Synology DDNS 설정 공식 문서](https://kb.synology.com/ko-kr/DSM/help/DSM/AdminCenter/connection_ddns?version=7)

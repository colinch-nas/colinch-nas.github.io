---
layout: post
title: "Tailscale VPN으로 어디서든 집 서버 안전하게 접속하기"
description: "Tailscale을 NAS와 PC에 설치해서 포트 포워딩 없이 어디서든 집 서버에 안전하게 접속하는 방법. 서브넷 라우팅으로 전체 홈 네트워크에 접근하는 설정 포함."
date: 2026-06-18
tags: [Tailscale, VPN, NAS보안, 홈서버]
comments: true
share: true
---

# Tailscale VPN으로 어디서든 집 서버 안전하게 접속하기

![Tailscale VPN 홈서버 원격 접속](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=800&q=80)

포트 포워딩으로 외부 접속 열면 편한데, 열린 포트가 많을수록 공격 표면이 넓어진다. Tailscale은 WireGuard 기반 P2P VPN인데, 포트 포워딩을 전혀 안 하고 집 서버에 접속할 수 있다. 설치도 간단해서 기술 지식 없는 가족도 쓸 수 있다.

## Tailscale이 하는 일

각 기기에 Tailscale을 설치하면 `100.x.x.x` 대역의 가상 IP를 받는다. 이 기기들끼리는 인터넷을 통해 암호화된 P2P 터널로 연결된다. 같은 집 안에 있는 것처럼 통신이 된다.

무료 플랜으로 기기 100대까지 쓸 수 있다. 개인 홈랩 용도로는 충분하다.

## 환경 정보

| 항목 | 버전/모델 |
|------|----------|
| 홈서버 OS | Debian 12 (Bookworm) |
| Synology DSM | DSM 7.2.2 |
| Tailscale 버전 | 1.68.x |

## 홈서버에 Tailscale 설치

```bash
# Debian/Ubuntu 기반
curl -fsSL https://tailscale.com/install.sh | sh

# 시작 및 인증
sudo tailscale up

# 인증 URL이 출력됨 → 브라우저에서 열고 계정 연결
```

Raspberry Pi (HAOS)에서는 **설정 → 애드온 스토어**에서 Tailscale 애드온 설치.

## Synology NAS에 Tailscale 설치

Synology는 패키지 센터에서 직접 설치가 안 된다. Docker로 설치하는 방법이 있다.

```yaml
# /volume1/docker/tailscale/docker-compose.yml
version: "3.9"
services:
  tailscale:
    image: tailscale/tailscale:latest
    container_name: tailscale
    hostname: synology-nas
    network_mode: host
    environment:
      - TS_AUTHKEY=tskey-auth-xxxxxxxxxxxx    # Tailscale 관리 콘솔에서 생성
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_USERSPACE=false
    volumes:
      - /volume1/docker/tailscale/state:/var/lib/tailscale
      - /dev/net/tun:/dev/net/tun
    cap_add:
      - NET_ADMIN
      - NET_RAW
    restart: unless-stopped
```

**TS_AUTHKEY** 생성: [Tailscale 관리 콘솔](https://login.tailscale.com/admin/settings/keys) → Keys → Generate auth key → 체크박스에서 Reusable 선택.

```bash
cd /volume1/docker/tailscale
sudo docker compose up -d

# 연결 상태 확인
sudo docker exec tailscale tailscale status
```

## PC와 스마트폰에 설치

[tailscale.com/download](https://tailscale.com/download)에서 OS별 클라이언트 설치. Windows, macOS, iOS, Android 모두 지원한다.

설치 후 로그인하면 같은 계정에 등록된 기기들이 보인다. NAS의 Tailscale IP(예: `100.64.0.5`)로 바로 접속 가능.

## 서브넷 라우팅 설정 (전체 홈 네트워크 접근)

기본 설치로는 Tailscale이 설치된 기기에만 접속된다. 서브넷 라우팅을 설정하면 `192.168.1.x` 전체 홈 네트워크에 접근할 수 있어서 IP 카메라, 공유기 관리 페이지 등 Tailscale이 없는 기기에도 접근된다.

```bash
# 홈서버에서 서브넷 광고
sudo tailscale up --advertise-routes=192.168.1.0/24

# Tailscale 관리 콘솔에서 라우팅 승인 필요
# Machines → 해당 기기 → 서브넷 라우트 승인
```

클라이언트 PC에서:
```bash
# 서브넷 라우팅 사용 활성화
tailscale up --accept-routes
```

이렇게 하면 VPN 연결된 상태에서 `192.168.1.50`(IP 카메라 주소)에 직접 접속이 된다.

![Tailscale 기기 목록과 VPN 연결 상태](https://images.unsplash.com/photo-1544197150-880d4212a0c4?w=800&q=80)

## Magic DNS 활용

Tailscale은 기기명으로 접속할 수 있는 Magic DNS를 제공한다. `100.64.0.5` 대신 `synology-nas` 또는 `synology-nas.tailscale.net`으로 접속이 된다.

관리 콘솔 → DNS → Magic DNS 활성화.

## 포트 포워딩과 Tailscale 비교

| 항목 | 포트 포워딩 | Tailscale |
|------|-----------|---------|
| 설정 복잡도 | 중간 | 낮음 |
| 보안 | 포트 노출 | 포트 비노출 |
| 접속 방법 | 공인 IP + 포트 | 가상 IP or 기기명 |
| 비용 | 무료 | 무료 (100기기) |
| 속도 | 직접 연결 | P2P (거의 동일) |

## 삽질했던 부분

**Synology Docker에서 TUN 장치 접근**: NAS에서 Docker로 Tailscale 올릴 때 `/dev/net/tun`이 없으면 컨테이너 시작이 실패한다. NAS에서 커널 모듈이 로드돼 있어야 하는데, DSM 7.2는 기본으로 있다. DSM 6.x는 수동으로 모듈을 로드해야 했다.

**공유기 CGNAT 문제**: 일부 ISP는 공용 IP를 여러 가구가 나눠 쓰는 CGNAT를 쓴다. DDNS가 동작하지 않고 포트 포워딩도 안 된다. 이 경우 Tailscale이 유일한 현실적 해결책이다.

**접속 속도**: P2P 연결이 안 되고 릴레이 서버를 거치면 속도가 느려진다. `tailscale ping [기기명]` 명령으로 릴레이 경유 여부를 확인할 수 있다.

## 한 줄 정리

Tailscale 설치하고 로그인 하나로 끝난다. 포트 포워딩 설정 없이 어디서든 집 서버에 안전하게 접속하고 싶다면 Tailscale이 현재 가장 쉬운 방법이다.

---
다음엔 [Proxmox VE 8 설치 — 미니PC로 가상화 홈서버 만들기]({% post_url 2026-06-19-10-00-00-185175-proxmox-ve8-minipc-virtualization-server %})를 다룬다.

**참고 링크**
- [Tailscale 공식 사이트](https://tailscale.com)
- [Tailscale 서브넷 라우팅 문서](https://tailscale.com/kb/1019/subnets)
- [Tailscale GitHub](https://github.com/tailscale/tailscale)

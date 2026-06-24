---
layout: post
title: "Uptime Kuma로 내 서비스들 모니터링하기 — 다운타임 알림 설정"
description: "Uptime Kuma를 Docker로 설치하고 홈서버에서 운영 중인 서비스들의 가동 상태를 모니터링하는 방법. Telegram 알림 연동과 상태 페이지 공유 설정 포함."
date: 2026-06-17
tags: [UptimeKuma, 자체호스팅, 홈서버, 모니터링]
comments: true
share: true
---

# Uptime Kuma로 내 서비스들 모니터링하기 — 다운타임 알림 설정

![Uptime Kuma 서비스 모니터링 대시보드](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)

Nextcloud, Vaultwarden, Immich가 다 돌아가고 있는데, 이게 실제로 살아 있는지 어떻게 알 수 있을까. 서버가 내려갔을 때 내가 직접 접속하려다 알게 되면 이미 늦다. Uptime Kuma는 각 서비스에 주기적으로 핑을 보내서 응답이 없으면 알림을 보내준다. 설치 5분이면 된다.

이전 글에서 [Immich 사진 서버를 설치]({% post_url 2026-06-16-10-00-00-148140-immich-google-photos-alternative %})했다. 이번엔 이 서비스들을 감시한다.

## 환경 정보

| 항목 | 버전/모델 |
|------|----------|
| NAS 모델 | Synology DS923+ |
| Uptime Kuma 버전 | 1.23.x |
| Docker 이미지 | louislam/uptime-kuma:latest |

## 설치

```yaml
# /volume1/docker/uptime-kuma/docker-compose.yml
version: "3.9"
services:
  uptime-kuma:
    image: louislam/uptime-kuma:latest
    container_name: uptime-kuma
    restart: unless-stopped
    ports:
      - "3001:3001"
    volumes:
      - /volume1/docker/uptime-kuma/data:/app/data
```

```bash
cd /volume1/docker/uptime-kuma
sudo docker compose up -d
```

접속: `http://NAS-IP:3001`

첫 접속 시 관리자 계정을 만든다.

## 모니터 추가

**모니터 추가** 버튼을 클릭하고 모니터링할 서비스를 등록한다.

| 유형 | 설명 | 사용 예 |
|------|------|---------|
| HTTP(s) | URL에 HTTP 요청 | 웹 서비스 전반 |
| TCP Port | 특정 포트 응답 | MQTT 브로커, DB 등 |
| Ping | ICMP 핑 | NAS, 공유기 |
| DNS | DNS 조회 응답 | DDNS 동작 확인 |

예시 설정:

```
이름: Nextcloud
모니터 유형: HTTP(s)
URL: https://cloud.example.com
하트비트 간격: 60초
재시도 횟수: 3
```

```
이름: Vaultwarden
모니터 유형: HTTP(s)
URL: https://vault.example.com
하트비트 간격: 60초
```

```
이름: Home Assistant
모니터 유형: HTTP(s)
URL: http://192.168.1.101:8123
하트비트 간격: 30초
```

## Telegram 알림 설정

서비스가 다운됐을 때 Telegram으로 알림을 받는 방법이다.

1. Telegram에서 **BotFather** 검색 → `/newbot` 명령으로 봇 생성
2. 봇 토큰 저장 (형식: `1234567890:ABCdefGHIjklmNOPqrstu-VWXyz`)
3. 본인의 Telegram 채팅 ID 확인: [@userinfobot](https://t.me/userinfobot) 에 메시지 보내면 알려줌

Uptime Kuma **설정 → 알림**에서 새 알림 추가:

```
알림 유형: Telegram
봇 토큰: (BotFather에서 받은 토큰)
채팅 ID: (본인 채팅 ID)
```

테스트 버튼으로 알림이 잘 오는지 확인 후 저장. 이후 모니터 편집에서 이 알림을 활성화한다.

![Uptime Kuma Telegram 알림 설정](https://images.unsplash.com/photo-1451187580094-2cfb76b7c9b7?w=800&q=80)

## 상태 페이지 공개

**상태 페이지**를 만들면 외부에 서비스 가동 상태를 공개할 수 있다. 가족에게 공유하거나 개인 포트폴리오에 붙일 수 있다.

**상태 페이지 → 새 상태 페이지** 에서:
- 페이지 이름, 슬러그(URL 경로) 설정
- 공개할 모니터 선택
- 커스텀 도메인 연결 가능

`https://status.example.com` 같은 도메인에 연결하면 전문적으로 보인다.

## 모니터링 팁

**응답 시간 추적**: HTTP 모니터는 응답 시간을 기록한다. 그래프에서 특정 시간대에 응답이 느린 패턴이 보이면 서버 부하를 확인해볼 수 있다.

**인증서 만료 모니터링**: HTTPS 서비스는 인증서 만료 알림 기능이 내장돼 있다. 만료 30일 전부터 경고가 온다. Let's Encrypt 자동 갱신이 실패했을 때 빠르게 알 수 있다.

## 삽질했던 부분

**내부 IP 모니터링의 한계**: 로컬 네트워크 서비스를 `192.168.1.x` IP로 모니터링하면 Uptime Kuma가 같은 네트워크 안에 있을 때만 확인할 수 있다. 외부에서 보는 서비스 가동 상태를 알려면 외부 도메인으로 모니터링해야 한다.

**재시도 횟수 설정**: 재시도를 1로 설정했더니 일시적인 네트워크 불안정에도 알림이 왔다. 재시도 3회, 간격 30초 정도로 설정하면 오탐이 줄어든다.

**컨테이너 재시작 후 알림 폭탄**: 서버를 재시작하면 모든 컨테이너가 잠시 내려갔다가 올라오는데, 이때 모든 모니터가 다운 감지 → 복구 알림을 보낸다. 재시작 전에 Uptime Kuma의 **유지 관리** 기능을 켜두면 이 기간 동안 알림을 보내지 않는다.

## 한 줄 정리

Uptime Kuma 하나면 운영 중인 모든 서비스를 한 화면에서 감시하고, 다운타임 발생 시 Telegram으로 즉시 알림 받을 수 있다.

---
다음엔 [Tailscale VPN으로 어디서든 집 서버 안전하게 접속하기]({% post_url 2026-06-18-10-00-00-172830-tailscale-vpn-home-server-remote-access %})를 다룬다.

**참고 링크**
- [Uptime Kuma GitHub](https://github.com/louislam/uptime-kuma)
- [Uptime Kuma Docker Hub](https://hub.docker.com/r/louislam/uptime-kuma)
- [Telegram BotFather](https://t.me/botfather)

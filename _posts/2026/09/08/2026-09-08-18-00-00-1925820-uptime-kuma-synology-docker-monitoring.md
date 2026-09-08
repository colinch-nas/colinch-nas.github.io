---
layout: post
title: "Uptime Kuma 설치 - Synology NAS에서 Docker 서비스 장애 알림 만들기"
description: "Synology DSM 7.4-90075의 Container Manager에서 Uptime Kuma를 설치하고 NAS, Home Assistant, Jellyfin의 상태와 장애 알림을 확인하는 방법을 정리했다."
date: 2026-09-08
tags: [UptimeKuma, Synology, Docker, NAS설정, HomeLab]
comments: true
share: true
---

![Synology NAS 홈서버 모니터링](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

NAS에 Docker 서비스를 여러 개 올리면 고장 자체보다 고장을 늦게 알아차리는 일이 더 곤란하다. Synology DSM 7.4-90075에서 Uptime Kuma를 설치하면 웹 화면, TCP 포트, Docker 서비스의 응답을 주기적으로 확인하고 장애 알림을 받을 수 있다. 이번에는 별도 데이터베이스 없이 Container Manager 하나로 구성했다.

## 확인한 환경

| 항목 | 값 |
|---|---|
| NAS | Synology DS923+ |
| DSM | 7.4-90075 |
| 설치 도구 | Container Manager |
| 내부 주소 | `192.168.0.20` |
| 모니터링 대상 | DSM, Home Assistant, Jellyfin |

DSM 7.4-90075는 설치 후 이전 DSM으로 되돌릴 수 없고 재부팅이 필요하다. 아직 업데이트하지 않았다면 Hyper Backup이나 스냅샷 상태를 확인한 뒤 진행하는 편이 안전하다. 업데이트 정보는 [Synology 공식 릴리스 노트](https://kb.synology.com/ko-kr/search?sources%5B%5D=release_note)에서 확인했다.

## Uptime Kuma 컨테이너 만들기

Container Manager에서 **프로젝트 → 생성**을 누르고 프로젝트 이름을 `uptime-kuma`로 정한다. YAML에는 다음 내용을 넣는다.

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:latest
    container_name: uptime-kuma
    restart: unless-stopped
    ports:
      - "3001:3001"
    volumes:
      - /volume1/docker/uptime-kuma:/app/data
```

컨테이너를 시작한 뒤 브라우저에서 `http://192.168.0.20:3001`을 연다. 첫 화면에서 관리자 계정을 만들고, 데이터 폴더가 실제로 `/volume1/docker/uptime-kuma`에 생성됐는지 File Station에서 확인한다. 이 경로를 빠뜨리면 컨테이너를 다시 만들 때 모니터와 계정이 사라진다.

## 모니터 등록과 장애 테스트

**새 모니터 추가**에서 다음처럼 등록했다.

| 대상 | 유형 | 주소 | 간격 |
|---|---|---|---|
| DSM | HTTP(s) | `https://nas.example.com` | 60초 |
| Home Assistant | HTTP(s) | `http://192.168.0.30:8123` | 60초 |
| Jellyfin | HTTP(s) | `http://192.168.0.20:8096` | 60초 |

DSM처럼 인증서가 적용된 주소는 HTTPS 모니터로 두고, 내부 서비스는 처음부터 외부 도메인으로 검사하지 않았다. 외부 DNS나 역방향 프록시가 잠깐 흔들려도 서비스 자체가 정상인데 장애로 기록될 수 있기 때문이다. Home Assistant는 로그인 화면이 떠도 HTTP 상태 코드가 200이면 정상으로 판단한다.

알림은 **설정 → 알림**에서 Telegram 또는 이메일을 연결했다. 저장 후에는 Jellyfin 컨테이너를 1분 정도 중지해 알림이 도착하는지 확인하고, 다시 시작해 복구 알림까지 확인한다. 정상 동작 여부는 초록색 화면보다 이 복구 테스트가 더 확실했다.

## 외부 공개할 때 주의할 점

Uptime Kuma의 3001 포트를 공유기에서 그대로 포워딩하지 않는다. 외부에서 대시보드가 필요하면 Synology 역방향 프록시(외부 도메인 요청을 내부 서비스로 전달하는 중계 서버)에 HTTPS를 붙이고 관리자 계정에는 2단계 인증을 켠다. 단순히 상태 알림만 받을 목적이면 Tailscale VPN 내부에서만 열어두는 구성이 더 낫다.

`latest` 태그는 편하지만 업데이트 시 예기치 않은 변경이 생길 수 있다. 처음 정상 작동한 뒤에는 이미지 버전을 고정하고, 업데이트 전 `/volume1/docker/uptime-kuma` 폴더를 별도로 백업한다.

정리하면 Uptime Kuma 설치는 컨테이너 하나로 끝나지만, 데이터 볼륨 지정·복구 알림 테스트·외부 포트 비공개까지 해야 운영 가능한 모니터링이 된다. NAS가 살아 있는지만 보지 말고 실제로 사용하는 Home Assistant와 Jellyfin도 함께 등록하는 게 핵심이다.

---
layout: post
title: "Uptime Kuma 설치 - Synology NAS Docker로 홈서버 장애 감시하기"
description: "Synology DSM 7.4 Container Manager에서 Uptime Kuma를 Docker로 설치하고 NAS, Home Assistant, 외부 접속 주소의 장애 알림을 설정하는 방법을 정리한다."
date: 2026-08-21
tags: [UptimeKuma, Synology, Docker, NAS설정, 홈서버, 자체호스팅]
comments: true
share: true
---

![Synology NAS에서 Uptime Kuma로 홈서버 상태를 감시하는 화면](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

Uptime Kuma를 Synology DSM 7.4의 Container Manager에 올리면 NAS, Home Assistant, Jellyfin 같은 서비스를 한 화면에서 감시할 수 있다. HTTP 응답과 TCP 포트를 확인하고 장애 알림도 보낸다.

## 구성과 준비물

내 환경은 Synology DS923+, DSM 7.4, Container Manager다. `/volume1/docker/uptime-kuma` 공유 폴더를 만들고 NAS의 3001 포트만 내부 관리 화면으로 사용한다.

| 항목 | 설정값 | 이유 |
|---|---|---|
| 저장 경로 | `/volume1/docker/uptime-kuma/data` | 컨테이너를 다시 만들어도 기록 유지 |
| 포트 | `3001:3001` | Uptime Kuma 웹 화면 |
| 감시 주기 | 60초 | 너무 짧은 감시로 NAS 부하를 만들지 않음 |
| 알림 | Telegram 또는 이메일 | 장애를 화면 밖에서도 확인 |

![Uptime Kuma 구성에서 확인할 대상과 감시 방식](https://images.unsplash.com/photo-1451187580459-43490279c0fa?w=1000&q=80)

핵심은 저장 경로다. 컨테이너를 다시 만들어도 모니터 목록과 장애 이력이 남아야 한다.

## Container Manager에서 설치

프로젝트 메뉴에서 새 프로젝트를 만들고 `compose.yaml`을 붙여 넣는다. 이미지 태그를 `latest`로 두면 자동 업데이트 때 화면이나 데이터베이스가 예상치 않게 바뀔 수 있어, 운영 환경에서는 확인한 버전으로 고정하는 편이 낫다.

```yaml
services:
  uptime-kuma:
    image: louislam/uptime-kuma:1
    container_name: uptime-kuma
    restart: unless-stopped
    ports:
      - "3001:3001"
    volumes:
      - /volume1/docker/uptime-kuma/data:/app/data
```

프로젝트를 실행한 뒤 `http://NAS주소:3001`에 접속해 관리자 계정을 만든다. 공유기 관리자 화면보다 `http://NAS주소:5000` 같은 실제 서비스부터 추가한다. 로그인 화면이 200 응답을 내는 HTTP 모니터가 이해하기 쉽다.

## 장애 감시와 알림 설정

`Monitor 추가`에서 유형을 `HTTP(s)`로 선택하고 Home Assistant 주소나 Jellyfin 주소를 입력한다. 외부 공개 주소는 내부 주소와 분리해 등록한다. 그래야 NAS 안에서는 살아 있지만 DDNS, 인증서, 포트포워딩 중 하나가 끊긴 상황을 구분할 수 있다.

Telegram 알림은 봇 토큰과 Chat ID를 입력한 뒤 테스트 메시지를 보내면 된다. 재시작 때 메시지가 쏟아지지 않게 60초 간격과 재시도 2회를 주고, 장애가 2~3분 이어질 때 알림이 오도록 조정했다.

외부 주소를 감시할 때는 Uptime Kuma까지 같은 NAS에 두면 인터넷 회선 자체가 끊긴 경우 감시 서버도 함께 사라진다. 이 구성은 “서비스가 죽었는지” 확인하는 용도고, 회선 장애까지 감시하려면 별도 VPS나 외부 모니터링 서비스를 한 개 추가해야 한다.

## 설치 후 확인할 체크리스트

- `data` 폴더에 Uptime Kuma 데이터베이스가 생성됐는지 확인
- NAS를 재부팅해 컨테이너 자동 시작 여부 확인
- Home Assistant 중지 후 2~3분 뒤 Telegram 장애 알림 확인
- 복구 후 정상 알림과 장애 이력이 함께 남는지 확인
- 외부 공개 서비스에는 NAS 관리자 주소를 감시 대상으로 등록하지 않기

Uptime Kuma는 설치보다 감시 위치가 중요하다. 같은 NAS에 두면 전원·인터넷·공유기 장애는 감지하지 못한다. 외부 공개 주소 감시는 별도 VPS나 외부 서비스로 분리하고, 설정 폴더는 Hyper Backup 대상에 포함한다.

참고: [Uptime Kuma 공식 GitHub](https://github.com/louislam/uptime-kuma), [Synology Container Manager 문서](https://www.synology.com/en-global/dsm/feature/container-manager)

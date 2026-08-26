---
layout: post
title: "Home Assistant 2026.8 Gatus 연동 - NAS 서비스 장애를 스마트홈 알림으로 받기"
description: "Home Assistant 2026.8.2의 Gatus 연동을 사용해 NAS의 Jellyfin·Nextcloud 상태를 대시보드와 자동화 알림으로 확인하는 방법을 정리한다."
date: 2026-08-26
tags: [HomeAssistant, Docker, NAS설정, 자체호스팅, UptimeKuma]
comments: true
share: true
---

![Home Assistant 2026.8 Gatus NAS 서비스 모니터링](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

Home Assistant 2026.8.2에서는 Gatus(홈서버 서비스 상태를 확인하는 모니터링 도구)를 공식 연동으로 추가할 수 있다. NAS에 Gatus를 Docker로 실행하고 Jellyfin, Nextcloud 같은 주소를 등록하면 서비스가 죽었을 때 Home Assistant의 센서와 알림 자동화로 받을 수 있다. 2026.8 릴리스 노트에 Gatus 통합이 Silver 품질로 들어왔다는 점을 확인하고 실제 구성을 붙여봤다.

## 구성 환경

| 항목 | 설정값 |
|---|---|
| NAS | Synology DS923+ |
| 컨테이너 | Gatus 최신 이미지, 포트 8080 |
| 스마트홈 | Home Assistant OS 2026.8.2 |
| 감시 대상 | Jellyfin 8096, Nextcloud 11000 |

Gatus와 Home Assistant는 같은 내부 네트워크에 두었다. 외부 도메인으로 감시하면 공유기나 인증서 문제까지 한꺼번에 섞이므로, 서비스 자체가 살아 있는지 확인하는 목적이라면 NAS 내부 주소를 쓰는 편이 원인 파악이 쉽다.

## NAS에 Gatus 설치

Synology Container Manager에서 프로젝트를 만들고 아래처럼 `docker-compose.yml`을 저장한다.

```yaml
services:
  gatus:
    image: twinproduction/gatus:latest
    container_name: gatus
    restart: unless-stopped
    ports:
      - "8080:8080"
    volumes:
      - /volume1/docker/gatus/config:/config
```

Gatus가 확인할 주소는 `/volume1/docker/gatus/config/config.yaml`에 넣는다. `conditions`의 `status == 200`을 빠뜨리면 HTTP 오류 페이지도 정상으로 판단할 수 있다.

```yaml
web:
  port: 8080

endpoints:
  - name: jellyfin
    group: nas
    url: "http://192.168.1.20:8096"
    interval: 30s
    conditions:
      - "[STATUS] == 200"
  - name: nextcloud
    group: nas
    url: "http://192.168.1.20:11000"
    interval: 30s
    conditions:
      - "[STATUS] == 200"
```

`192.168.1.20`은 NAS의 실제 고정 IP로 바꾼다. Container Manager에서 프로젝트를 시작한 뒤 `http://NAS주소:8080`을 열어 두 서비스가 `UP`으로 보이는지 확인한다. 처음에는 Nextcloud가 리버스 프록시 뒤에 있어 301이나 302를 반환할 수 있다. 이 경우 최종 로그인 주소가 아니라 내부 서비스 주소를 넣거나 조건을 `status == 200 || status == 302`로 조정한다.

## Home Assistant 2026.8에 연결

Home Assistant에서 `설정 → 기기 및 서비스 → 통합 추가`를 열고 `Gatus`를 검색한다. Gatus 주소에는 `http://192.168.1.20:8080`을 입력한다. 연결이 되면 `binary_sensor.gatus_jellyfin`처럼 엔드포인트별 센서가 생기며, 정상은 `on`, 장애는 `off`로 표시된다.

자동화는 서비스가 한 번 끊겼다고 즉시 울리지 않도록 2분 이상 꺼진 경우만 알리게 만들었다. 재시작 중 잠깐 발생하는 오탐을 줄이는 데 효과가 있었다.

```yaml
alias: NAS 서비스 장애 알림
trigger:
  - platform: state
    entity_id: binary_sensor.gatus_jellyfin
    to: "off"
    for: "00:02:00"
action:
  - service: notify.mobile_app_my_phone
    data:
      title: "NAS 서비스 확인"
      message: "Jellyfin이 2분 이상 응답하지 않는다. Gatus에서 상세 상태를 확인한다."
mode: single
```

## 삽질한 지점과 주의사항

Gatus 컨테이너에서 `localhost:8096`을 쓰면 NAS가 아니라 Gatus 컨테이너 자기 자신을 가리킨다. 반드시 NAS IP나 Docker 네트워크의 서비스 이름을 사용해야 한다. 또 `latest` 이미지는 편하지만 업데이트 시 설정 동작이 바뀔 수 있으므로, 안정화 후에는 확인한 버전 태그로 고정하는 편이 낫다.

Gatus는 서비스 HTTP 응답만 확인한다. 디스크가 95% 찼거나 ZFS 풀이 degraded인 상태까지 알려주지는 않는다. NAS 자체 상태는 DSM 알림, TrueNAS 알림, SMART 검사와 따로 묶어야 한다. 외부 공개 주소를 감시하려면 내부 감시와 별도 엔드포인트로 나눠야 “서비스는 살아 있지만 포트포워딩만 실패한 상황”도 구분할 수 있다.

짧게 정리하면 `Gatus Docker 설치 → 내부 IP로 엔드포인트 등록 → Home Assistant 2026.8.2 연동 → 2분 지연 알림` 순서다. NAS 서비스가 실제로 죽었을 때 접속을 시도해서 알아채는 구조에서, 장애 발생 시점에 먼저 알림을 받는 구조로 바뀐다.

- [Home Assistant 2026.8 릴리스 노트](https://www.home-assistant.io/blog/2026/08/05/release-20268/)
- [Gatus 공식 문서](https://gatus.io/docs)

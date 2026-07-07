---
layout: post
title: "TrueNAS Home Assistant 앱 - 2026.7.1 설치 전 데이터셋부터 나누기"
description: "TrueNAS Apps의 Home Assistant 2026.7.1을 NAS에 설치할 때 데이터셋, 포트, 백업 범위를 앞에서 정하는 실전 설정 순서다."
date: 2026-07-07
tags: [TrueNAS, HomeAssistant, 스마트홈, NAS설정, HomeLab]
comments: true
share: true
---
![TrueNAS 홈서버에서 Home Assistant 앱을 설정하는 화면](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)
이 그림에서는 NAS 저장소와 스마트홈 서버가 같은 장비에 있을 때, 앱 설정과 데이터 보관 위치를 분리해야 한다는 점을 봐야 한다.

TrueNAS SCALE에서 Home Assistant를 올릴 때는 앱 설치 버튼보다 데이터셋 결정이 앞에 와야 한다. 2026년 7월 7일 기준 TrueNAS Apps 카탈로그의 Home Assistant는 앱 버전 2026.7.1이고, 요구 버전은 TrueNAS 24.10.2.2 이상이다. 설치는 쉽지만 `/config` 위치를 대충 두면 백업과 복구가 바로 꼬인다.

내가 확인한 환경은 TrueNAS SCALE 25.x, 앱 카탈로그 Stable, Home Assistant 2026.7.1, 공유기 내부망 `192.168.0.0/24`다. 처음엔 기본값으로 설치해도 되겠다고 생각했는데, Zigbee 동글, Matter Server, 백업 파일까지 붙이려니 앱 데이터 경로가 안 보이면 장애 때 할 일이 늘었다.

## 설치 전에 정할 값

| 항목 | 권장값 | 이유 |
|---|---:|---|
| 데이터셋 | `tank/apps/home-assistant` | 스냅샷과 백업 대상을 분리한다 |
| 웹 포트 | `8123` | HA 기본 포트라 문서와 예제가 맞다 |
| 네트워크 | 내부망만 접근 | 관리자 화면을 바로 인터넷에 열지 않는다 |
| 백업 | TrueNAS 스냅샷 + HA 백업 | 앱 설정과 NAS 복구 범위가 다르다 |

데이터셋은 TrueNAS 화면에서 **Datasets -> Add Dataset**으로 만든다. 이름은 `apps/home-assistant`처럼 앱 이름이 보이게 둔다. 압축은 기본 `lz4`로 충분하고, SMB 공유는 만들지 않는다. 설정 파일을 PC에서 편집하고 싶어서 SMB를 열어두면 권한이 섞이는 경우가 있다.

앱 설치 흐름은 이렇다.

```text
Apps -> Discover Apps -> Home Assistant -> Install
Application Name: home-assistant
Web Port: 8123
Storage: tank/apps/home-assistant
Timezone: Asia/Seoul
```

설치 후에는 `http://트루나스IP:8123`으로 접속한다. 여기서 바로 외부 접속부터 만들지 말고, 계정 생성과 지역 설정을 끝낸 뒤 **Settings -> System -> Backups**에서 수동 백업을 하나 만든다. 이 파일이 데이터셋 안에 남는지 확인해야 TrueNAS 스냅샷으로 같이 잡힌다.

## 동글과 자동검색은 따로 본다

Zigbee USB 동글을 TrueNAS에 꽂아 쓸 계획이면 앱 설정에서 장치 패스스루가 필요하다. 문제는 NAS 재부팅 뒤 장치명이 `/dev/ttyUSB0`에서 `/dev/ttyUSB1`로 바뀌는 경우다. 내가 삽질한 지점도 여기였다. 가능하면 장치 ID 기반 경로를 확인한다.

TrueNAS Shell에서 USB 장치를 확인한다.

```bash
ls -l /dev/serial/by-id/
```

Home Assistant의 자동검색(mDNS/SSDP)은 네트워크 설정에 영향을 받는다. 스마트 조명이나 ESPHome 기기가 같은 VLAN에 있는데도 안 보이면 HA 문제가 아니라 NAS 앱 네트워크 격리일 수 있다. 이 경우에는 앱 로그보다 공유기 멀티캐스트 설정, VLAN 방화벽, TrueNAS 앱 네트워크 설정을 같이 봐야 한다.

## 외부 공개는 나중에 붙인다

처음 설치한 Home Assistant를 포트포워딩으로 바로 열면 안 된다. 최소 조건은 관리자 계정 2단계 인증, 최신 패치, 백업 확인이다. 외부에서 꼭 봐야 한다면 Tailscale VPN이나 Cloudflare Tunnel처럼 관리자 UI 노출 범위를 줄이는 쪽이 낫다. Reverse Proxy(외부 도메인 요청을 내부 서버로 전달하는 중계 서버)를 쓰더라도 TrueNAS 관리 포트와 Home Assistant 포트를 섞지 않는다.

짧게 정리하면 이렇다. TrueNAS Apps의 Home Assistant 2026.7.1은 설치 자체보다 저장소와 네트워크 경계를 앞에서 정해야 오래 간다. `tank/apps/home-assistant` 데이터셋을 만들고, 8123 포트로 내부망에서만 열고, 설치 직후 HA 백업 파일이 TrueNAS 스냅샷 범위에 들어오는지 확인한다. Zigbee나 Matter는 이후에 붙여도 늦지 않다.

자료: [TrueNAS Apps Home Assistant 카탈로그](https://apps.truenas.com/catalog/home-assistant_stable/), [Home Assistant 2026.7 릴리스 노트](https://www.home-assistant.io/blog/2026/07/01/release-20267/)

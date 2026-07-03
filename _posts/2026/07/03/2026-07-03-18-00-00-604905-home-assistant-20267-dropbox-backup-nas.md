---
layout: post
title: "Home Assistant 2026.7 Dropbox 백업 - NAS Docker 운영자가 빼먹기 쉬운 범위"
description: "Home Assistant 2026.7.0의 Dropbox 백업 통합을 NAS Docker 환경에서 설정하고, 실제로 보호되는 파일과 따로 백업해야 할 항목을 정리한다."
date: 2026-07-03
tags: [HomeAssistant, Docker, 홈서버, 백업전략, NAS보안]
comments: true
share: true
---

![NAS에서 Home Assistant 백업 파일을 확인하는 작업 화면](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)
이 그림에서 볼 건 NAS 디스크 안의 설정 백업과 외부 클라우드 사본을 분리해서 생각해야 한다는 점이다.

Home Assistant 2026.7.0에서 Dropbox 백업 통합이 추가됐다. NAS Docker로 Home Assistant를 돌리는 경우엔 이 기능을 켜도 NAS 전체가 보호되는 건 아니다. 내가 확인한 기준으론 `/config` 안에서 Home Assistant가 만드는 백업을 Dropbox 위치로 하나 더 보내는 용도에 가깝다. 그래서 Docker Compose 파일, Zigbee2MQTT 설정, NAS 공유 폴더는 별도 백업으로 남겨야 한다.

## 왜 이 기능을 바로 켜도 되는가

2026년 7월 1일 공개된 Home Assistant 2026.7 릴리스 노트에는 Dropbox가 새 통합으로 들어왔다. Home Assistant Cloud 구독이나 별도 Dropbox 개발자 앱 없이 계정 연결 방식으로 백업 위치를 추가할 수 있다는 설명이 핵심이다. 예전엔 NAS 로컬 스냅샷만 믿고 있다가 디스크 풀 문제나 실수로 `/volume1/docker/homeassistant`를 지우면 복구가 애매했다.

내 환경은 Synology DS923+, Container Manager, Home Assistant Container 2026.7.0, 설정 경로는 `/volume1/docker/homeassistant/config`다.

| 항목 | Dropbox 백업 포함 | 따로 백업 필요 |
|---|---:|---:|
| Home Assistant 설정 | 예 | 아니오 |
| 자동화·대시보드 | 예 | 아니오 |
| Docker Compose 파일 | 아니오 | 예 |
| Zigbee2MQTT 데이터 | 아니오 | 예 |
| NAS 사진·문서 공유 | 아니오 | 예 |

## 설정 순서

Home Assistant 화면에서 아래 순서로 들어가면 된다.

| 단계 | 위치 | 확인할 것 |
|---|---|---|
| 1 | 설정 → 기기 및 서비스 | Dropbox 통합 추가 |
| 2 | Dropbox 로그인 | 백업용 계정인지 확인 |
| 3 | 설정 → 시스템 → 백업 | 백업 위치에 Dropbox 표시 확인 |
| 4 | 새 백업 만들기 | 암호화 비밀번호 별도 보관 |
| 5 | Dropbox 웹 | `.tar` 백업 파일 생성 여부 확인 |

여기서 삽질한 부분은 권한보다 백업 범위였다. Dropbox에 파일이 생기면 다 끝난 것처럼 보이는데, 실제로는 Home Assistant 컨테이너 바깥 파일이 빠진다. NAS에서 컨테이너를 Compose로 올렸다면 아래 파일은 같이 보관해야 복구가 편하다.

Compose 파일과 주변 설정을 NAS 백업 폴더로 복사하는 예시다.

```bash
mkdir -p /volume1/backup/homeassistant-extra
cp /volume1/docker/homeassistant/docker-compose.yml /volume1/backup/homeassistant-extra/
cp -a /volume1/docker/zigbee2mqtt /volume1/backup/homeassistant-extra/
```

## 복구 테스트는 작은 VM에서 한다

운영 NAS에 바로 복구 테스트를 하면 포트 8123 충돌이나 MQTT 주소 꼬임이 생긴다. 미니 PC나 Proxmox VM에 임시 Docker를 만들고 `/config`만 복원하는 편이 낫다.

```bash
docker run -d \
  --name ha-restore-test \
  -p 18123:8123 \
  -v /tmp/ha-restore/config:/config \
  ghcr.io/home-assistant/home-assistant:2026.7.0
```

브라우저에서 `http://테스트서버IP:18123`으로 접속해 자동화, 통합, 대시보드가 보이면 1차 확인은 끝난다. Zigbee 동글이나 블루투스 어댑터는 테스트 VM에 붙이지 않아도 된다. 여기서는 백업 파일이 열리는지, 핵심 설정이 살아나는지만 보면 된다.

## 주의할 점

Dropbox 백업을 켰다고 3-2-1 백업이 완성되는 건 아니다. 로컬 NAS 스냅샷 1개, 외부 클라우드 1개, 그리고 Docker Compose 같은 운영 메모 1개가 같이 있어야 복구 시간이 줄어든다. 특히 암호화 비밀번호를 비밀번호 관리자에 넣지 않으면 백업 파일은 있어도 못 푸는 상황이 된다.

짧게 정리하면 이렇다.

- Home Assistant 2026.7.0 Dropbox 통합은 켜둘 만하다.
- NAS Docker 운영자는 `/config` 밖 파일을 따로 챙겨야 한다.
- 첫 백업 뒤에는 Dropbox 웹에서 파일 생성 시간을 직접 확인한다.
- 복구 테스트는 운영 NAS가 아니라 임시 컨테이너에서 한다.

출처: [Home Assistant 2026.7 릴리스 노트](https://www.home-assistant.io/blog/2026/07/01/release-20267/)

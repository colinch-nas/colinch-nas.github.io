---
layout: post
title: "Home Assistant 2026.9.2 업데이트 - Synology Docker에서 안전하게 적용하고 롤백하기"
description: "Home Assistant 2026.9.2를 Synology DSM 7.4 Container Manager에 적용하는 방법과 백업, 이미지 고정, 장애 발생 시 롤백 절차를 정리했다."
date: 2026-09-12
tags: [HomeAssistant, Synology, Docker, DSM, NAS보안]
comments: true
share: true
---
![Home Assistant 업데이트 전 백업을 준비한 Synology NAS](/assets/images/2026-09-10-synology-ransomware-backup-strategy.png)

Home Assistant 2026.9.2는 2026년 9월 11일 공개된 현재 최신 패치다. Synology NAS에서 Container Manager로 운영 중이라면 `stable` 태그를 바로 당기기보다, 백업을 먼저 만들고 버전을 고정한 뒤 업데이트하는 편이 안전하다. 이번에는 DSM 7.4, Container Manager, Home Assistant Container를 쓰는 환경에서 실제로 재현할 수 있는 순서를 정리한다.

## 이번 업데이트에서 확인한 것

공식 2026.9 릴리스의 핵심 변화는 Modbus(전력계·인버터 등 산업용 장비 통신 규격) 통합이 여러 연결을 공유할 수 있게 된 점이다. 다만 Container 설치 방식은 Home Assistant OS와 달리 앱(Add-on) 저장소가 없다. 업데이트 알림이 보이지 않아도 컨테이너 이미지를 직접 교체해야 한다.

| 운영 방식 | 업데이트 방법 | 내가 보는 적합한 경우 |
|---|---|---|
| Home Assistant OS | UI에서 Core 업데이트 | HA 전용 미니PC·VM |
| Container | 이미지 pull 후 컨테이너 재생성 | Synology에서 여러 서비스 운영 |
| Core·Supervised | 현재 신규 운영 비추천 | 기존 환경 유지 외에는 피함 |

공식 문서도 현재 Home Assistant OS를 대부분 사용자에게 권장하고, Container는 호스트 OS와 업데이트를 직접 관리하는 방식으로 설명한다. 이미 NAS에 Docker 서비스가 모여 있다면 Container의 관리 편의성이 더 크지만, 장애 시 복구 절차까지 준비해야 한다.

## 업데이트 전 백업

### 1. Home Assistant 백업 파일 만들기

먼저 Synology의 공유 폴더 `docker/homeassistant/config`를 Hyper Backup이나 Snapshot Replication의 대상에 넣는다. Home Assistant Container는 설정 폴더가 곧 데이터베이스와 자동화 파일이므로 컨테이너만 다시 만드는 것으로는 복구되지 않는다.

실행 중인 컨테이너의 설정 폴더를 압축해 별도 폴더에 복사하는 방법도 있다. 아래 경로는 내 NAS의 예시이므로 공유 폴더명은 환경에 맞게 바꾼다.

```bash
cd /volume1/docker/homeassistant
tar -czf /volume1/backup/homeassistant-before-2026.9.2.tgz config
```

### 2. 현재 이미지 버전 기록

롤백할 때 필요한 것은 컨테이너 이름이 아니라 이전 이미지다. Container Manager의 이미지 목록에서 `homeassistant/home-assistant`가 아니라 공식 이미지인 `ghcr.io/home-assistant/home-assistant`의 기존 태그를 확인한다.

```bash
docker inspect homeassistant \
  --format='{{.Config.Image}} | {{.Image}}'
docker image ls ghcr.io/home-assistant/home-assistant
```

## 버전을 고정해 업데이트하기

Container Manager에서 프로젝트로 관리한다면 `compose.yaml`과 `.env`를 같은 폴더에 둔다. `stable`을 그대로 쓰면 다음 업데이트 때 원인을 구분하기 어려워서, 이번에는 `2026.9.2`를 직접 고정했다.

```dotenv
HA_VERSION=2026.9.2
TZ=Asia/Seoul
```

아래 Compose 설정은 Home Assistant가 공유 폴더의 설정을 사용하고, Zigbee USB 스틱을 컨테이너에 전달하는 구성이다. USB 경로는 NAS에서 실제로 확인한 값으로 바꿔야 한다.

```yaml
services:
  homeassistant:
    container_name: homeassistant
    image: ghcr.io/home-assistant/home-assistant:${HA_VERSION}
    restart: unless-stopped
    network_mode: host
    environment:
      TZ: ${TZ}
    volumes:
      - /volume1/docker/homeassistant/config:/config
      - /etc/localtime:/etc/localtime:ro
    devices:
      - /dev/ttyUSB0:/dev/ttyUSB0
```

프로젝트 폴더에서 이미지를 받고 컨테이너를 교체한다. `up -d`는 설정 폴더를 지우지 않고 같은 이름의 컨테이너만 새 이미지로 재생성한다.

```bash
docker compose pull
docker compose up -d
docker compose logs --tail=80 -f homeassistant
```

로그에 `Starting Home Assistant`가 표시된 뒤 웹 UI의 설정 → 시스템에서 2026.9.2를 확인한다. USB 장치가 사라졌다면 `/dev/ttyUSB0`가 `/dev/ttyACM0`로 바뀐 경우가 많다. 이걸 모르고 YAML을 고치는 데 시간을 꽤 썼다.

## 문제가 생겼을 때 롤백

부팅은 되지만 통합이 실패하거나 자동화가 멈추면 새 버전을 계속 만지지 말고, `.env`의 버전만 직전 확인값으로 되돌린다. 예를 들어 업데이트 전 이미지가 `2026.8.3`이었다면 아래처럼 실행한다.

```dotenv
HA_VERSION=2026.8.3
```

```bash
docker compose pull
docker compose up -d
docker compose logs --tail=120 homeassistant
```

이때 설정 폴더까지 복원해야 하는지는 증상으로 판단한다. 단순 호환성 문제면 이미지 롤백만으로 충분하다. 데이터베이스 마이그레이션 오류나 설정 파일 손상이면 컨테이너를 멈춘 뒤 백업 압축을 별도 복사본에 풀어 확인하고, 원본을 바로 덮어쓰지 않는다.

## 업데이트 체크리스트

- [ ] Home Assistant 전체 백업을 NAS 외부에도 보관
- [ ] 기존 이미지 태그와 컨테이너 설정 기록
- [ ] `stable` 대신 2026.9.2로 버전 고정
- [ ] Zigbee USB 경로와 권한 확인
- [ ] 웹 UI 로그인, 자동화 1개, 센서 1개 테스트
- [ ] 이상이 없을 때만 이전 이미지 정리

핵심은 새 이미지보다 복구 가능한 상태를 먼저 만드는 것이다. Synology NAS의 Container Manager에서 Home Assistant를 운영한다면 이 방식으로 업데이트 범위를 좁힐 수 있고, 문제가 생겨도 `.env` 한 줄과 백업으로 되돌릴 수 있다.

- [Home Assistant 2026.9 릴리스 노트](https://www.home-assistant.io/blog/2026/09/02/release-20269/)
- [Home Assistant Container 설치 공식 문서](https://www.home-assistant.io/installation/linux)
- [Home Assistant Container 공통 작업 문서](https://www.home-assistant.io/common-tasks/container/)

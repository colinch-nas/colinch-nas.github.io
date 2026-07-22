---
layout: post
title: "Home Assistant 2026.7 Update all - NAS Docker에서 안전하게 업데이트하는 방법"
description: "Home Assistant 2026.7의 Update all 기능을 NAS Docker 환경에서 점검하는 방법과 자동화·애드온 업데이트 전 백업 순서를 정리했다."
date: 2026-07-22
tags: [HomeAssistant, Docker, NAS설정, 홈서버, 자체호스팅]
comments: true
share: true
---

![Home Assistant 2026.7 NAS Docker 업데이트](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

Home Assistant 2026.7에서는 여러 업데이트를 한 화면에서 처리하는 `Update all`이 추가됐다. NAS에서 Home Assistant Container를 Docker로 운영한다면 버튼을 누르기 전 백업과 재시작 순서를 확인해야 한다. 컨테이너 이미지 업데이트와 Home Assistant 내부 업데이트는 별개다.

## 2026.7에서 바뀐 점

공식 릴리스 기준으로 Update all은 본체, 통합 구성 요소, ESPHome 장치 등의 보류 업데이트를 한곳에서 확인하는 기능이다. 편하지만 Zigbee 장치 펌웨어까지 같은 날 처리하지는 않는 편이 낫다.

| 구분 | Update all에서 확인할 것 | NAS 운영 판단 |
|---|---|---|
| Home Assistant Core | 2026.7.x 패치 버전 | 백업 후 업데이트 |
| 통합 구성 요소 | 제조사 연동 버전 | 장애 영향 확인 |
| ESPHome·장치 | 장치별 펌웨어 | 한 번에 하나씩 |

## Docker NAS에서 업데이트 전 백업

Home Assistant의 **설정 → 시스템 → 백업**에서 전체 백업을 만든다. 표시만 믿지 말고 NAS의 `/config/backups`에 파일이 생겼는지 확인한다. Docker Compose 파일과 환경 변수도 별도 보관한다.

컨테이너 이름이 `homeassistant`일 때 현재 설정 경로를 압축하는 명령은 다음과 같다.

```bash
docker exec homeassistant ha backups list
docker inspect homeassistant --format '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{"\n"}}{{end}}'
```

첫 명령은 Home Assistant OS에서만 정상 동작할 수 있다. Container 설치에서는 `docker inspect`로 실제 `/config` 경로를 확인해야 한다. 다른 폴더를 백업하는 실수가 가장 흔한 삽질 포인트다.

Container 이미지 자체를 올리는 작업은 Update all과 별개다. Compose 파일이 있는 폴더에서 백업 확인 후 다음처럼 실행한다.

```bash
docker compose pull homeassistant
docker compose up -d homeassistant
docker compose logs --tail=100 homeassistant
```

Synology Container Manager에서는 프로젝트의 **이미지 업데이트 → 재생성** 순서다. `latest`를 무조건 당기지 말고 현재 태그와 공식 변경 사항을 비교한다.

## Update all 적용 순서

1. 자동화가 적게 실행되는 시간에 전체 백업을 생성한다.
2. **설정 → 시스템 → 업데이트**에서 대기 항목을 펼쳐 변경 내용을 확인한다.
3. Core만 먼저 업데이트하고 2~3분 뒤 대시보드와 자동화 로그를 확인한다.
4. 문제가 없을 때 통합 구성 요소를 업데이트한다.
5. ESPHome 장치는 전원과 Wi-Fi가 안정된 한 대만 먼저 업데이트한다.

업데이트 후에는 **설정 → 시스템 → 로그**에서 `Error`와 `Failed setup`을 검색한다. 2026.7의 자동화 편집이 쉬워져도 YAML은 사라지지 않는다. 기존 YAML은 UI에서 저장하기 전 원본을 별도 보관한다.

## NAS 운영자가 피할 실수

Home Assistant 2026.7과 NAS 운영체제 업데이트를 같은 날 겹치지 않는 게 좋다. 문제가 생겼을 때 컨테이너 이미지와 DSM·Docker 런타임을 구분하기 어렵다. Update all도 무인 실행하지 말고 수동 적용한다.

`Update all`은 일괄 실행 버튼보다 목록 관리 화면으로 쓰는 편이 안전하다. 백업을 확인하고 Core → 통합 → 장치 순서로 나누면 복구 범위를 작게 유지할 수 있다.

출처: [Home Assistant 2026.7 릴리스 노트](https://www.home-assistant.io/blog/2026/07/01/release-20267/)

---
layout: post
title: "Home Assistant 2026.10 NAS 복원 호환성 - Docker·HAOS VM·라디오별 백업 총비용"
description: "Home Assistant 2026.10을 NAS Docker에서 HAOS VM이나 새 장비로 옮길 때 백업 범위, 애드온·미디어·Zigbee 복원 조건과 장비별 총비용을 정리한다."
date: 2026-10-25
tags: [HomeAssistant, Docker, Synology, HomeLab, 백업전략]
comments: true
share: true
---

![Home Assistant NAS 복원 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

*이 그림에서 볼 것은 Home Assistant 컨테이너보다 백업 파일과 별도 복원 장비를 함께 준비하는 구조다.*

Home Assistant 2026.10을 NAS Docker에서 운영 중이고 자동화·Zigbee·애드온까지 살리고 싶다면 복원 목적에는 HAOS VM이 더 유리하다. 자동화와 대시보드만 쓰는 1인 환경은 Docker를 유지하면서 `/config`와 백업 파일을 별도 저장하는 편이 싸다. Docker 백업 하나로 HAOS의 애드온과 미디어까지 돌아온다고 가정하면 복원 당일에 다시 설치할 항목이 생긴다.

## 설치 방식에 따라 백업 범위가 다르다

공식 문서 기준으로 OS는 `config`, `share`, `addons`, `ssl`, `media`를 전체 백업에 포함할 수 있다. Container는 설정과 선택한 기록 중심이며 앱·미디어·share는 OS 방식의 백업 항목이 아니다. 2025.1부터 모든 설치 방식에서 복원할 수 있지만 “복원 가능”과 “같은 운영 환경이 복원됨”은 다르다. [백업 문서](https://www.home-assistant.io/common-tasks/general/backups/)와 [설치 방식 비교](https://contributors.home-assistant.io/os/installation-types/)를 기준으로 봐야 한다.

| 환경 | 복원되는 핵심 | 별도 준비할 것 | 맞는 사용자 |
|---|---|---|---|
| NAS Docker Container | 설정, 선택한 기록 | Compose 파일, 이미지 버전, Zigbee2MQTT·Mosquitto 데이터 | 1~2인, 앱을 직접 관리할 때 |
| NAS의 HAOS VM | 설정, 앱, 미디어 등 선택 항목 | VM 백업, USB 라디오 연결, 백업 암호 키 | 2~5인, 애드온과 장치 관리가 중요할 때 |
| 새 미니PC의 HAOS | 기존 시스템의 전체 구성 | 새 디스크 여유 공간, USB 동글, 네트워크 재연결 | NAS 장애에도 자동화를 살려야 할 때 |

Zigbee 동글도 별도 복원 대상이다. 새 장비에 라디오를 연결하지 않으면 장치가 unavailable로 보이거나 Zigbee 마이그레이션이 필요할 수 있다. 백업 암호화 키가 든 emergency kit를 잃으면 백업 파일만 남아도 복원할 수 없다.

## 가상 사례로 보는 구매 기준

가격은 2026년 10월 25일 실제 견적이 아닌 비교용 가정이다. Synology DS923+에 Docker를 두고 1명이 조명 자동화만 쓴다면 추가 구매액은 0원에 가깝다. 외장 SSD 10만 원과 USB Zigbee 동글·예비 라디오 6만 원을 더해도 복원 재료는 약 16만 원이다.

4인 가정에서 애드온·카메라·Zigbee를 함께 쓰면 HAOS VM용 RAM과 USB 패스스루를 확인해야 한다. 메모리 증설 8만 원, 백업 SSD 15만 원, 예비 동글 3만 원을 가정하면 약 26만 원이다. NAS 고장 때도 켜져야 한다면 N100급 미니PC와 SSD 비용을 더하고, VM 스냅샷만 믿지 말고 백업 파일을 외부에 복사한다.

기록을 10일만 보관하고 애드온을 거의 쓰지 않으며 NAS와 동글이 정상이라면 Docker를 유지해도 된다. NAS 교체 계획이 있거나 자동화 중단 비용이 크다면 HAOS VM 복원 경로를 미리 만들어 보는 편이 싸다.

## 복원 가능성을 확인하는 체크리스트

- [ ] Home Assistant 2026.10 백업을 만들고 백업 목록에서 내용과 용량을 열어 봤다.
- [ ] 암호화 키 emergency kit를 NAS와 다른 위치에 보관했다.
- [ ] Docker Compose, 이미지 태그, `/config` 경로를 백업했다.
- [ ] Zigbee2MQTT의 `data` 폴더와 네트워크 키를 Home Assistant 백업과 따로 보관했다.
- [ ] 테스트 VM 또는 예비 장비에서 백업 업로드 화면까지 진입했다.
- [ ] 복원 뒤 자동화 3개와 Zigbee 장치 3개의 확인 절차를 적었다.

실제 장비에서 복원하지 않았다면 “복원 완료”라고 기록하지 않는다. 파일 업로드와 라디오·애드온·자동화 동작은 별도 검증이다. 2026.10은 10월 7일 공개됐으므로 패치 버전이 달라지면 [릴리스 노트](https://www.home-assistant.io/blog/2026/10/07/release-202610)를 다시 확인한다.

짧게 판단하면, 1인 Docker 사용자는 외장 백업과 Compose 보관이 우선이고, 2인 이상 또는 Zigbee·애드온 사용자는 HAOS VM 복원 경로를 미리 확보하는 편이 낫다. 총비용은 NAS 디스크 가격만이 아니라 백업 SSD, 예비 라디오, 키 보관, 장애 때 대체 장비까지 포함해야 한다.

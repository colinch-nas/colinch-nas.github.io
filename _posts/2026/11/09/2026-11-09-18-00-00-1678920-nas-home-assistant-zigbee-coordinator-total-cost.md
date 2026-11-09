---
layout: post
title: "NAS Home Assistant Zigbee 코디네이터 선택 - USB·Ethernet 호환성과 총비용"
description: "Synology DSM 7·QNAP QTS·TrueNAS SCALE에서 Home Assistant Zigbee 코디네이터를 USB와 Ethernet 중 고르는 기준을 기기 수·NAS 위치·복구 비용으로 정리한다."
date: 2026-11-09
tags: [HomeAssistant, Zigbee, NAS, Docker, Synology, QNAP, TrueNAS, HomeLab]
comments: true
share: true
---

![NAS에 연결한 Home Assistant Zigbee 코디네이터](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

그림에서 볼 부분은 NAS 본체보다 Zigbee 코디네이터와 USB·네트워크 경로다.

Home Assistant를 1~2명이 20개 안팎의 Zigbee 기기로 운영한다면 **USB 코디네이터를 NAS에 직접 연결하는 선택**이 가장 싸고 복구가 단순하다. 반대로 NAS가 금속 랙 안에 있거나 Zigbee 기기가 50개 이상으로 늘어날 예정이면 유선 Ethernet 코디네이터를 검토할 수 있다. 다만 Wi-Fi·VPN을 거친 직렬 프록시는 안정적인 선택이 아니며, USB 동글을 샀다고 Docker 컨테이너가 자동으로 장치를 보는 것도 아니다.

## 규모별 선택 기준

아래 기기 수는 제조사 제한이 아니라 구매 전 설계를 위한 가상 구간이다. 실제 한계는 집 구조, 라우터 수, 채널 간섭, 메시 전원 기기 수에 따라 달라진다.

| 환경 | 우선 선택 | 이유 | 구매하지 않아도 되는 조건 |
|---|---|---|---|
| 1~20개·NAS와 생활 공간이 가까움 | USB 2.0 코디네이터 | 비용과 설정 경로가 가장 작음 | 이미 호환되는 동글이 있고 USB 연장선으로 위치를 옮길 수 있을 때 |
| 20~50개·NAS가 공유기 옆 또는 랙 안 | USB + 차폐 연장선 | 유선 네트워크 장비를 추가하지 않음 | NAS USB 포트와 컨테이너 장치 매핑이 정상일 때 |
| 50~100개·NAS와 집 중앙이 멂 | 유선 Ethernet 코디네이터 | 라디오 위치를 NAS에서 분리 가능 | NAS 근처에 USB 동글을 두고 메시 라우터를 충분히 배치할 수 있을 때 |

Home Assistant 공식 ZHA 문서는 새 구성에 Connect ZBT-2를 권장하고, ZBT-1은 기존 사용자를 위한 지원 장치로 구분한다. 동시에 USB 3.0이 무선 간섭을 만들 수 있으므로 USB 2.0 포트나 연장 케이블을 권한다. 이는 “최신 동글이면 어디에 꽂아도 된다”는 뜻이 아니다.

## NAS별 호환성에서 막히는 지점

Synology DSM 7의 Container Manager, QNAP QTS의 Container Station, TrueNAS SCALE의 앱·Docker 환경은 모두 컨테이너에 `/dev/ttyUSB0` 또는 `/dev/ttyACM0` 같은 장치를 넘기는 단계가 필요하다. 장치가 호스트에 보여도 컨테이너 권한이 없으면 ZHA가 시작되지 않는다.

코디네이터를 USB로 운영할 때는 Compose에 장치 경로를 고정한다. 실제 경로는 NAS에서 확인한 값으로 바꿔야 한다.

```yaml
services:
  homeassistant:
    image: ghcr.io/home-assistant/home-assistant:stable
    devices:
      - /dev/ttyUSB0:/dev/ttyUSB0
    volumes:
      - ./config:/config
    network_mode: host
    restart: unless-stopped
```

업데이트나 재부팅 뒤 경로가 바뀌면 `/dev/serial/by-id/`의 고정 식별자를 쓰는 편이 낫다. 설정 폴더와 함께 Zigbee 네트워크 키, 장치 데이터베이스, Compose 파일을 백업해야 한다. 코디네이터만 새로 사면 페어링 정보를 자동으로 되살릴 수 있다고 생각하면 안 된다.

Ethernet 모델은 NAS의 USB 패스스루를 피할 수 있지만, ZHA 공식 문서는 Wi-Fi·WAN·VPN을 통한 Serial Proxy를 권장하지 않는다. 쓸 경우에도 유선 LAN, 고정 IP, 같은 스위치, 재부팅 후 자동 연결을 확인해야 한다.

## 가상 총비용과 복구 비용

가격은 판매 시점에 바뀌므로 아래는 2026년 11월 구매 판단용 가정이다. USB 동글 4만 원, USB 2.0 연장선 1만 원, Ethernet 코디네이터 12만 원, 예비 USB 동글 4만 원으로 계산했다.

| 구성 | 초기 장비 가정 | 장애 때 필요한 것 | 2년 비용 가정 |
|---|---:|---|---:|
| USB 단일 | 5만 원 | 동글 교체·설정 복원 | 5만 원 |
| USB + 예비 동글 | 9만 원 | 백업 복원 후 예비품 교체 | 9만 원 |
| Ethernet 단일 | 12만 원 | 네트워크·전원·코디네이터 점검 | 12만 원 이상 |

기존 NAS가 이미 생활 공간에 있고 20개 이하라면 Ethernet 모델을 추가 구매할 이유가 약하다. 반대로 USB 간섭 때문에 반복적으로 끊기거나 NAS를 지하실·랙에 둘 예정이면 초기 비용보다 라디오 위치를 바꾸는 효과가 중요하다. 어떤 구성이든 Home Assistant 설정 백업을 테스트 인스턴스에 복원하고, 센서 하나의 배터리 보고와 조명 하나의 명령을 확인해야 운영 완료로 본다.

## 구매 전 체크리스트

- [ ] NAS 호스트에서 코디네이터가 `/dev/serial/by-id/`로 보이는가
- [ ] Docker 컨테이너에 장치 권한과 고정 경로를 넣었는가
- [ ] USB 3 포트·SSD·공유기와 코디네이터를 떨어뜨렸는가
- [ ] Ethernet 모델이면 Wi-Fi·VPN이 아닌 유선 LAN인가
- [ ] Home Assistant 설정, 네트워크 키, Compose 파일을 별도 저장했는가
- [ ] 새 장비에서 테스트 복원 후 Zigbee 장치 2개를 확인했는가

짧게 판단하면 NAS와 코디네이터를 가까이 둘 수 있는 소규모 집은 USB 2.0 동글과 연장선이면 충분하다. 장비를 사기 전에 위치와 간섭을 해결할 수 있다면 Ethernet 모델은 필요 없다. 구매한다면 코디네이터 가격보다 Docker 장치 매핑과 복원 가능한 백업을 먼저 확인해야 한다.

참고 자료: [Home Assistant ZHA 호환 하드웨어와 USB 간섭 안내](https://www.home-assistant.io/integrations/zha), [Home Assistant Container의 장치 매핑 안내](https://www.home-assistant.io/installation/alternative)

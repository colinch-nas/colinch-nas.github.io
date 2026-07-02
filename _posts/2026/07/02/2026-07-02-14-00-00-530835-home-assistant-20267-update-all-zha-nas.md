---
layout: post
title: "Home Assistant 2026.7 - NAS Docker 운영자는 Update all을 이렇게 쓴다"
description: "Home Assistant 2026.7 정식 릴리스의 Update all과 ZHA 관리 화면 변경을 NAS Docker 운영 기준으로 적용하는 점검 순서다."
date: 2026-07-02
tags: [HomeAssistant, Docker, 스마트홈, NAS설정]
comments: true
share: true
---
![Home Assistant NAS Docker 업데이트 점검](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)

Home Assistant 2026.7 정식 버전은 NAS Docker 사용자도 바로 올릴 만하다. 다만 새로 생긴 `Update all`을 아무 카드에서나 누르면 ESPHome, HACS, 앱성 통합이 한꺼번에 움직여서 장애 지점이 흐려진다. 나는 2026년 7월 2일 기준 공식 릴리스 노트에서 확인한 뒤, 운영 NAS에서는 업데이트 묶음을 나눠 적용하는 쪽으로 정리했다.

## 왜 이번 버전을 그냥 넘기기 애매한가

2026.7에서 눈에 띄는 변화는 자동화 편집기보다 업데이트 화면이다. `Settings -> Updates`가 카드 단위로 묶이고, 일부 영역에 `Update all` 버튼이 생겼다. NAS에서 Home Assistant Core를 컨테이너로 돌리는 경우 Core 자체는 compose로 관리하지만, HACS 통합이나 ESPHome 장치 펌웨어는 Home Assistant 화면에서 처리하는 일이 많다.

예전처럼 하나씩 누르면 귀찮지만 실패 지점은 명확했다. 이번 UI는 빠른 대신 "무엇이 같이 올라갔는지"를 따로 기록해야 한다. 이걸 빼먹고 업데이트했다가 조명 자동화가 죽었는데 HACS 카드 문제인지, ESPHome 펌웨어 문제인지, ZHA 장치 문제인지 헷갈리기 쉽다.

## NAS Docker에서는 이렇게 나눈다

Core 컨테이너는 기존처럼 숫자 태그로 고정한다. `latest`를 쓰면 UI 업데이트 기록과 Docker 이미지 변경 기록이 서로 안 맞는다.

현재 컨테이너 버전과 이미지 태그를 남긴 뒤 작업한다.

```bash
docker compose ps
docker image ls | grep home-assistant
```

그다음 Home Assistant 화면에서는 한 번에 전부 누르지 말고 순서를 나눈다.

1. HACS 프론트엔드 카드만 업데이트
2. HACS 통합 업데이트
3. ESPHome 장치 펌웨어는 1~2개씩 적용
4. 자동화가 정상 동작하는지 확인
5. 문제가 없을 때 남은 항목에 `Update all` 사용

이 순서가 느려 보여도 실제로는 복구 시간이 줄어든다. 특히 전등, 보일러, 도어 센서처럼 가족이 바로 체감하는 장치는 같은 날 몰아서 올리지 않는 게 낫다.

## ZHA 사용자는 새 관리 화면을 확인한다

2026.7에서는 ZHA(Zigbee Home Automation, Home Assistant 기본 Zigbee 연동)의 장치 관리 화면이 전용 페이지로 바뀌었다. 클러스터, 바인딩, 시그니처, 이웃 장치 정보를 탭으로 볼 수 있어서 USB Zigbee 동글을 NAS에 꽂아 쓰는 환경에서는 점검이 쉬워졌다.

업데이트 후에는 아래 세 가지만 본다.

- 배터리 센서가 마지막 보고 시간을 갱신하는지
- 라우터 역할을 하는 콘센트가 이웃 장치에 보이는지
- 자동화에 쓰는 모션 센서가 10분 안에 이벤트를 남기는지

나는 여기서 한 번 삽질했다. NAS 재부팅 뒤 `/dev/ttyUSB0`가 `/dev/ttyUSB1`로 바뀌면서 ZHA 화면은 멀쩡해 보이는데 센서 이벤트만 안 들어왔다. compose에서 장치 경로를 고정하지 않았던 게 원인이었다.

USB Zigbee 동글은 가능하면 by-id 경로로 고정한다.

```yaml
devices:
  - /dev/serial/by-id/usb-Silicon_Labs_CP2102N_USB_to_UART_Bridge_Controller:/dev/ttyUSB0
```

## 주의할 점

`Update all`은 편하지만 롤백 버튼은 아니다. NAS Docker 운영자는 `/config` 백업, compose 파일, 이미지 태그 세 가지가 있어야 되돌릴 수 있다. Home Assistant 2026.7의 새 업데이트 화면은 관리 시간을 줄여주지만, 운영 서버에서는 묶음 업데이트를 어디까지 허용할지 정하는 게 핵심이다. ZHA까지 같이 쓴다면 업데이트 직후 자동화 성공 여부보다 실제 센서 이벤트가 들어오는지부터 확인해야 한다.

---
layout: post
title: "Home Assistant OS 2026.7 - Synology VMM 가상머신 설치와 Zigbee 연결"
description: "Synology DSM의 Virtual Machine Manager에 Home Assistant OS 2026.7을 설치하고 고정 IP와 USB Zigbee 동글까지 연결하는 실제 설정 방법을 정리한다."
date: 2026-08-19
tags: [HomeAssistant, Synology, NAS설정, Zigbee, 스마트홈, 홈서버]
comments: true
share: true
---

![Synology NAS에서 Home Assistant OS 가상머신 설치](https://images.unsplash.com/photo-1558008258-3256797b43f3?w=1200&q=80)

Synology NAS에서 Home Assistant를 오래 운영할 생각이라면 Docker보다 Virtual Machine Manager(VMM)에 Home Assistant OS를 올리는 편이 관리가 수월하다. Home Assistant OS 2026.7 계열은 백업, 애드온, 업데이트를 한 화면에서 처리할 수 있고, 공식 문서도 가상머신 설치에서는 OS 이미지를 권장한다. 내가 확인한 기준은 DSM 7.2.x, x86-64 Synology NAS, VMM 2.x다.

이 그림에서 봐야 할 부분은 NAS 안에 별도 운영체제인 Home Assistant OS를 올려 Docker 컨테이너와 분리하는 구조다.

## Docker 대신 VMM을 고른 이유

기존에 Home Assistant 2026.7 Docker 설치 글이 있어도 기능 차이가 있다. Container 방식은 업데이트와 네트워크를 직접 관리해야 하고, 애드온을 쓸 수 없다. 반면 OS 방식은 Mosquitto, Zigbee2MQTT 같은 주변 서비스를 애드온으로 붙이기 쉽다. 다만 VMM은 NAS RAM을 따로 먹는다. 4GB NAS라면 Jellyfin이나 다른 컨테이너와 동시에 돌릴 때 버벅일 수 있어 8GB 이상 환경이 편하다.

| 항목 | 권장값 | 확인 위치 |
|---|---:|---|
| vCPU | 2개 | VMM 가상머신 설정 |
| 메모리 | 4GB | 2GB는 최소값 |
| 디스크 | 32GB 이상 | SSD 볼륨 권장 |
| 펌웨어 | UEFI | 부팅 옵션 |
| 네트워크 | 브리지 | 공유기에서 IP 할당 |

## Home Assistant OS 이미지 준비

Home Assistant 공식 설치 페이지에서 `Virtual Machine`용 KVM 이미지를 내려받는다. 2026년 8월 19일 확인 기준 문서의 최신 표기는 2026.7.x이며, 압축을 풀면 `.qcow2` 디스크 이미지가 나온다.

공식 문서: [Home Assistant Linux/VM 설치](https://www.home-assistant.io/installation/linux/)

압축을 푼 파일을 DSM File Station에서 VMM이 읽을 수 있는 ISO 저장소가 아닌, VMM 스토리지로 가져온다. 여기서 ISO로 등록하려고 한 것이 첫 번째 삽질이었다. QCOW2는 가상 디스크라서 `이미지` 또는 가상머신 생성 단계에서 가져와야 한다.

## Synology VMM에서 가상머신 만들기

VMM에서 `가상 머신 → 생성 → 가져오기`를 선택하고 디스크 이미지로 QCOW2 파일을 지정한다. 새로 생성하는 방식이라면 다음 값으로 시작하면 된다.

```text
이름: home-assistant
CPU: 2 vCPU
메모리: 4096 MB
디스크: HAOS qcow2, 32GB 이상
펌웨어: UEFI
네트워크: 기본 브리지 또는 물리 LAN 브리지
자동 시작: 활성화
```

네트워크는 NAT보다 브리지를 선택한다. 브리지 모드여야 Home Assistant가 공유기에서 별도 IP를 받아 `http://homeassistant.local:8123` 또는 `http://NAS에 표시된 IP:8123`으로 접근할 수 있다. 생성 후 `전원 켜기`를 누르고 2~5분 기다린다. 첫 부팅 직후에는 브라우저가 안 열려도 디스크 초기화 중일 수 있다.

## 고정 IP와 초기 접속

브라우저에서 `http://가상머신_IP:8123`으로 접속해 계정을 만든다. 가상머신 내부에서 고정 IP를 억지로 설정하기보다 공유기 DHCP 예약에서 MAC 주소를 고정하는 방법이 덜 꼬인다. Home Assistant 재설치나 네트워크 어댑터 변경 때 설정 파일을 다시 만질 일이 줄어든다.

| 확인 항목 | 정상 기준 |
|---|---|
| VMM 상태 | 실행 중 |
| IP 주소 | 공유기 DHCP 목록에 표시 |
| 포트 | 외부 포트포워딩 없이 내부 8123 접속 |
| 시간대 | Asia/Seoul |
| 백업 | 기기 추가 전에 전체 백업 생성 |

## Zigbee USB 동글 연결

Zigbee 동글을 NAS에 꽂았다면 VMM의 `가상 머신 → 편집 → USB 장치`에서 해당 USB 장치를 Home Assistant VM에 연결한다. NAS 재부팅 뒤 연결이 풀리는 모델도 있어 `자동 연결` 옵션을 켜고, 동글의 제조사와 경로를 확인한다.

Home Assistant의 `설정 → 시스템 → 하드웨어`에서 동글이 보이면 Zigbee Home Automation(ZHA) 또는 Zigbee2MQTT를 선택한다. USB 장치가 안 보이면 NAS의 Container Manager가 이미 장치를 점유하고 있지 않은지 확인한다. 한 동글을 Docker와 VM에 동시에 연결하려 하면 둘 다 불안정해진다.

## 운영 중 주의할 점

VMM 스냅샷만 백업으로 믿으면 안 된다. Home Assistant에서 `설정 → 시스템 → 백업`으로 전체 백업을 만들고, NAS의 다른 볼륨이나 외부 저장소에도 복사한다. VM 디스크가 손상되면 스냅샷과 원본이 함께 사라질 수 있기 때문이다.

외부 접속은 8123 포트를 공유기에 바로 열지 않는 편이 낫다. 집 밖에서 관리할 때는 NAS와 휴대폰에 Tailscale을 설치하면 포트포워딩 없이 사설 네트워크로 접근할 수 있다. Synology용 Tailscale은 Package Center에서 설치할 수 있지만, DSM에서는 Tailscale SSH가 동작하지 않는다는 제한이 있다. [Tailscale의 Synology 안내](https://tailscale.com/docs/integrations/synology)도 이 부분을 명시한다.

핵심은 세 가지다. x86-64 NAS에 Home Assistant OS QCOW2 이미지를 올리고, VMM은 UEFI·브리지·4GB RAM으로 시작한다. Zigbee 동글은 VM 하나에만 연결한다. Home Assistant 자체 백업과 NAS 데이터 백업을 분리해 둬야 장애 때 복구할 수 있다.

---
layout: post
title: "Proxmox VE 9.2 설치 - 미니PC 홈서버를 가상화 서버로 만드는 방법"
description: "Proxmox VE 9.2를 미니PC에 설치하고 네트워크 브리지, 저장소, 업데이트까지 홈서버 운영에 필요한 초기 설정을 실제 순서대로 정리한다."
date: 2026-09-13
tags: [Proxmox, 홈서버, 미니PC, HomeLab, 가상화, Docker]
comments: true
share: true
---

![Proxmox VE 9.2 미니PC 홈서버 구축](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

미니PC에 Proxmox VE 9.2를 설치하면 운영체제 하나만 돌리는 장비가 아니라 Home Assistant, Ubuntu Server, Docker를 각각 분리한 홈서버로 쓸 수 있다. 2026년 9월 기준 공식 x86 설치 이미지는 9.2-1이고, ARM64용 9.2-1 이미지도 8월에 공개됐다. 여기서는 일반적인 Intel·AMD 미니PC 기준으로 설정한다. Proxmox 공식 릴리스의 Dynamic Load Balancer는 클러스터에서 의미가 큰 기능이라, 한 대짜리 홈서버에서는 설치와 안정적인 저장소 구성이 더 중요하다.

그림에서 봐야 할 부분은 미니PC 한 대 위에 Proxmox가 올라가고, 그 아래에 용도별 VM(가상 머신)이 나뉘는 구조다.

## 준비할 장비와 설치 기준

| 항목 | 권장 기준 | 이유 |
|---|---|---|
| 미니PC | 4코어 이상, RAM 16GB 이상 | Home Assistant와 서비스 VM을 동시에 실행 |
| 저장장치 | NVMe 512GB 이상 | VM 디스크와 ISO 저장 공간 확보 |
| 네트워크 | 유선 1Gbps | 브리지 네트워크와 파일 접근 안정성 |
| 설치 이미지 | Proxmox VE 9.2-1 | x86 Intel·AMD 장비 기준 |

BIOS에서 Intel VT-x 또는 AMD-V(가상화 기능)를 켜고, 설치 USB를 만든다. 설치 과정에서 선택한 디스크가 초기화되므로 기존 자료가 있는 NVMe를 그대로 넣고 진행하면 안 된다.

## Proxmox VE 9.2 설치

공식 ISO를 내려받은 뒤 SHA256 값까지 확인한다. macOS나 Linux에서는 다운로드한 ISO와 공식 페이지의 해시가 같은지 아래처럼 비교할 수 있다.

```bash
shasum -a 256 proxmox-ve_9.2-1.iso
```

USB로 부팅하고 `Install Proxmox VE`를 선택한다. 미니PC의 내장 디스크를 지정하고 국가·시간대·키보드·root 비밀번호를 입력한다. 관리 화면에서 사용할 고정 IP는 공유기의 DHCP 예약 주소와 겹치지 않게 잡는다. 예를 들어 공유기 대역이 `192.168.0.0/24`라면 Proxmox를 `192.168.0.20`, 게이트웨이를 `192.168.0.1`로 둔다.

설치가 끝나면 브라우저에서 `https://192.168.0.20:8006`으로 접속한다. 자체 서명 인증서 경고가 뜨는 것은 초기 상태에서 정상이다. 로그인 후 `Datacenter → pve → System → Updates`에서 저장소를 확인한다.

## 설치 직후 업데이트와 저장소 정리

새 ISO도 설치 시점 이후 패키지가 바뀌었을 수 있다. Proxmox 공식 안내처럼 GUI 업데이트 기능을 쓰거나 콘솔에서 패키지를 갱신한다.

```bash
apt update
apt dist-upgrade
```

유료 구독이 없다면 `enterprise.proxmox.com` 저장소가 오류를 내는 경우가 있다. 이때는 GUI의 저장소 화면에서 enterprise 항목을 비활성화하고 `pve-no-subscription` 저장소를 추가한다. 운영 중인 VM이 있다면 업데이트 전 스냅샷만 믿지 말고, 중요한 데이터는 NAS나 외장 디스크에도 복사한다.

## vmbr0와 VM 네트워크 확인

설치 프로그램은 보통 물리 NIC에 연결된 `vmbr0` 브리지를 자동으로 만든다. 브리지(가상 스위치)는 VM이 집 안의 다른 기기와 같은 네트워크에 직접 참여하게 하는 설정이다. VM 생성 화면에서 네트워크 장치의 Bridge가 `vmbr0`, Model이 `VirtIO`인지 확인한다.

| 증상 | 확인할 곳 | 조치 |
|---|---|---|
| VM에서 인터넷이 안 됨 | VM의 Bridge | `vmbr0` 선택 |
| IP를 못 받음 | 공유기 DHCP | VM MAC 주소 예약 |
| 관리 화면까지 끊김 | 물리 NIC·케이블 | Wi-Fi 대신 유선 사용 |

Proxmox 관리 IP를 DHCP에 맡기면 공유기 재부팅 뒤 주소가 바뀌어 접속하지 못할 수 있다. 관리 IP는 설치 때 고정하고, VM은 DHCP 예약으로 관리하는 구성이 초기에 덜 헷갈렸다.

## 홈서버용 VM을 만드는 기준

Home Assistant는 2 vCPU·4GB RAM, Ubuntu Server는 2~4 vCPU·4~8GB RAM부터 시작하면 된다. 디스크는 처음부터 전체를 크게 할당하지 말고, thin provision을 사용해 실제 사용량만큼 점유하도록 둔다. 단, 물리 NVMe가 가득 차면 모든 VM이 동시에 멈출 수 있으니 `local-lvm` 사용량 80%를 넘기지 않는 선에서 정리한다.

VM 생성 후에는 `Options → Start at boot`를 켠다. Home Assistant나 MQTT처럼 부팅 직후 필요한 서비스는 시작 순서를 정해 두고, 테스트용 VM은 자동 시작에서 제외하면 재부팅 후 자원 부족을 줄일 수 있다.

## 설치 후 확인 체크리스트

- [ ] BIOS 가상화 기능과 유선 연결 확인
- [ ] Proxmox 관리 IP를 DHCP 예약 또는 고정 주소로 지정
- [ ] `vmbr0`에 물리 NIC가 연결됐는지 확인
- [ ] enterprise 저장소 오류와 업데이트 완료 여부 확인
- [ ] VM별 RAM·디스크 상한 설정
- [ ] 중요한 VM 백업을 별도 NAS에 복사

Proxmox VE 9.2는 미니PC 한 대를 홈랩의 기반으로 삼기 좋은 출발점이다. 다만 가상화 기능을 켜는 것보다 관리 IP를 고정하고, 저장소가 가득 차지 않게 감시하고, VM 백업을 다른 장치에 두는 세 가지가 실제 운영에서 더 오래 남는 설정이다.

참고한 공식 자료는 [Proxmox VE 9.2 릴리스 안내](https://www.proxmox.com/en/about/company-details/press-releases/proxmox-virtual-environment-9-2)와 [Proxmox VE 9.2 ISO 다운로드 페이지](https://www.proxmox.com/en/downloads/proxmox-virtual-environment/iso/proxmox-ve-9-2-iso-installer)다.

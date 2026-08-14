---
layout: post
title: "Proxmox VE 9.2 홈랩 설정 - Dynamic Load Balancer를 켜기 전 확인할 것"
description: "Proxmox VE 9.2의 최신 기능과 미니PC 홈서버 적용 범위를 정리하고, 새 설치·업데이트·클러스터 설정에서 놓치기 쉬운 지점을 실제 명령어로 설명한다."
date: 2026-08-15
tags: [Proxmox, 홈서버구축, HomeLab, 미니PC, NAS보안]
comments: true
share: true
---

![Proxmox VE 9.2 홈랩 가상화 서버](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

Proxmox VE 9.2는 2026년 5월 21일 공개된 현재 기준 최신 정식 버전이다. 핵심 추가 기능은 Dynamic Load Balancer(클러스터 노드의 부하를 보고 VM을 자동 이동하는 기능)지만, 미니PC 한 대짜리 홈서버에서는 켜도 효과가 없다. 새로 구축한다면 9.2로 설치하고, 기존 Proxmox VE 8은 백업과 호환성 확인 뒤에 업그레이드하는 편이 안전하다.

## 9.2에서 실제로 달라진 점

공식 발표 기준으로 Debian 13.5, Linux 커널 7.0, QEMU 11.0, LXC 7.0, ZFS 2.4를 기반으로 한다. Dynamic Load Balancer 외에도 WireGuard·BGP 패브릭, GUI에서 관리하는 사용자 지정 CPU 모델, HA(고가용성) 클러스터 유지보수용 Arm/Disarm가 추가됐다.

| 환경 | 9.2 기능 활용도 | 판단 |
|---|---:|---|
| N100 미니PC 1대 | 낮음 | 안정적인 VM·LXC 운영이 목적 |
| 미니PC 2~3대 클러스터 | 높음 | HA와 부하 분산을 구성할 때 유용 |
| Synology NAS + Proxmox | 중간 | NAS는 백업 저장소로 두는 구성이 현실적 |

그림에서 봐야 할 부분은 서버 한 대의 성능보다 VM을 어디에 배치할지가 중요한 구조라는 점이다.

## 새 미니PC에 9.2 설치하기

공식 ISO는 `9.2-1`이며 SHA256 값까지 공개돼 있다. 다운로드한 ISO를 USB에 기록하기 전에 무결성을 확인한다.

```bash
shasum -a 256 proxmox-ve_9.2-1.iso
```

공식 다운로드 페이지의 해시와 일치하면 USB로 부팅해 설치한다. 설치 화면에서는 관리용 유선 NIC를 고르고, 호스트 이름은 `pve01.home.arpa`처럼 고정한다. DHCP 주소로 시작하면 VM을 옮긴 뒤 접속 주소가 바뀌어 꽤 번거롭다.

설치 직후에는 웹 UI의 `Updates`에서 패키지를 갱신한다. 콘솔에서 처리할 경우 아래처럼 실행한다.

```bash
apt update
apt dist-upgrade
```

구독이 없는 홈랩이면 엔터프라이즈 저장소를 무작정 삭제하기보다, 활성화 상태를 확인한 뒤 `pve-no-subscription` 저장소를 사용한다. 운영 중인 VM이 있다면 이 작업 전에 Proxmox 백업 파일과 VM 설정을 NAS에 복사한다.

## Dynamic Load Balancer는 클러스터에서만 켠다

단일 노드에서는 이동할 다른 노드가 없으므로 이 기능을 설정할 이유가 없다. 세 대의 노드가 있고 HA 리소스를 운영한다면 `Datacenter`의 클러스터 스케줄링 옵션에서 동적 모드를 선택하고, CPU·메모리 사용률이 높은 VM부터 이동하는지 확인한다. 자동 이동은 편하지만 Zigbee 동글이나 GPU처럼 특정 호스트에 묶인 장치는 HA 대상에서 제외해야 한다.

점검 항목은 이 정도면 충분하다.

- VM 디스크가 모든 노드에서 접근 가능한 공유 스토리지인가
- USB 패스스루 장치가 특정 노드에 고정돼 있는가
- 자동 이동 후 Home Assistant의 Zigbee 포트가 바뀌지 않는가
- 노드 장애 시 복구할 Proxmox Backup Server 또는 NAS 백업이 있는가

유지보수 때는 HA를 잠시 `Disarm`하고 작업한 뒤 `Arm`으로 되돌린다. 그래야 재부팅 중 HA가 의도치 않게 VM을 다른 노드로 옮기거나 펜싱(fencing, 장애 노드를 격리하는 절차)하지 않는다.

## 정리

Proxmox VE 9.2는 새 홈서버 설치용으로 선택할 만하다. 다만 Dynamic Load Balancer는 미니PC 한 대의 성능을 높이는 기능이 아니라 여러 노드의 VM 배치를 조정하는 기능이다. 단일 NAS·미니PC 구성이라면 9.2 설치, 고정 IP, 백업 검증에 집중하고 클러스터를 만들 때 동적 스케줄링을 검토하면 된다.

출처: [Proxmox VE 9.2 공식 발표](https://www.proxmox.com/en/about/company-details/press-releases/proxmox-virtual-environment-9-2), [Proxmox VE 9.2 ISO 다운로드](https://proxmox.com/en/downloads/proxmox-virtual-environment/iso/proxmox-ve-9-2-iso-installer)

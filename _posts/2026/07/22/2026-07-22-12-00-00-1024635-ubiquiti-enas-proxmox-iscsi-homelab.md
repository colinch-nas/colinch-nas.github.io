---
layout: post
title: "Ubiquiti Enterprise NAS - Proxmox iSCSI 홈랩 연결 전에 확인할 것"
description: "2026년 6월 공개된 Ubiquiti Enterprise NAS(ENAS)의 ZFS·25GbE·iSCSI 기능을 기준으로 Proxmox VE 9.2 공유 스토리지 연결과 운영 전 점검 순서를 정리했다."
date: 2026-07-22
tags: [Ubiquiti, NAS, Proxmox, HomeLab, iSCSI, ZFS]
comments: true
share: true
---

![Ubiquiti Enterprise NAS와 Proxmox 홈랩 공유 스토리지 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

Ubiquiti가 2026년 6월 18일 공개한 Enterprise NAS(ENAS)는 ZFS, 듀얼 25GbE SFP28, iSCSI(네트워크 디스크)를 앞세웠다. 공식 발표에는 Proxmox 호환도 적혀 있다. 홈랩에서는 용량보다 스토리지 네트워크 분리가 중요하다.

## ENAS가 기존 NAS와 다른 지점

| 항목 | ENAS 공개 사양 | 홈랩에서 볼 부분 |
|---|---|---|
| 파일 보호 | ZFS, ECC 메모리 | 스냅샷과 장애 대응은 별도 확인 |
| 네트워크 | 25GbE SFP28 2포트 | 스위치·트랜시버까지 같은 속도여야 함 |
| 가상화 | UniFi Drive의 iSCSI | VM 디스크용과 백업용 풀을 나눌 것 |
| 백업 | 오프사이트 백업 기능은 Coming Soon | 출시 전에는 기존 백업을 유지할 것 |

그림에서 볼 부분은 ENAS와 Proxmox를 일반 LAN에 몰아넣지 않는 구성이다. 관리 트래픽과 VM 디스크 트래픽이 섞이면 장애 원인을 찾기 어려워진다.

## Proxmox 쪽 네트워크부터 고정한다

환경은 Proxmox VE 9.2 단일 노드, 관리망 `192.168.10.0/24`, 스토리지망 `192.168.20.0/24`다. ENAS는 `192.168.20.20`, Proxmox 스토리지 NIC는 `192.168.20.11`로 고정한다. 중간 스위치가 1GbE면 교체한다.

장비를 고정한 뒤 Proxmox에서 스토리지 경로를 확인한다.

```bash
ip -br addr
ping -c 4 192.168.20.20
```

패킷 손실이 있으면 iSCSI 로그인보다 케이블, SFP 모듈, VLAN부터 고친다.

## iSCSI 연결은 테스트 풀에서 시작한다

ENAS 관리 화면에서 iSCSI 대상(Target)을 만들 때 VM 운영 풀과 테스트 풀을 분리한다. 대상 이름은 `proxmox-test`로 정하고 IQN은 기록해 둔다. 접근 제어는 Proxmox 스토리지 IP만 허용한다.

Proxmox에 iSCSI 대상을 추가하는 명령은 아래처럼 실행한다. 주소와 IQN은 ENAS 화면에서 확인한 값으로 바꾼다.

```bash
apt update
apt install -y open-iscsi
systemctl enable --now open-iscsi

iscsiadm -m discovery -t st -p 192.168.20.20
iscsiadm -m node --login
iscsiadm -m session
```

세션이 보인다고 바로 VM 디스크로 쓰면 안 된다. 여러 노드가 iSCSI LUN을 함께 마운트할 때 일반 ext4를 공유하면 파일시스템이 깨질 수 있다. 클러스터는 LVM-thin·Ceph를 검토하고, 단일 노드는 임시 VM으로 재부팅 후 자동 재연결을 확인한다.

## 속도보다 복구 테스트가 우선이다

25GbE 숫자만 보고 VM을 전부 옮기는 건 위험하다. iSCSI는 네트워크가 끊기는 순간 VM 디스크 I/O가 멈추고, 게스트 OS가 파일시스템 오류를 남길 수 있다. 아래 순서로 확인한다.

- Proxmox에서 ENAS 세션이 재부팅 뒤 자동 로그인되는가
- 스위치에서 스토리지 VLAN이 관리 VLAN과 분리됐는가
- VM 한 대를 중지한 뒤 스냅샷 생성·삭제가 정상인가
- ENAS와 Proxmox 양쪽 백업이 다른 장치에 존재하는가
- 장애 때 복구 절차와 알림을 실제로 확인했는가

ENAS의 멀티사이트 백업 오케스트레이션은 공식 발표상 “Coming Soon”이다. 정식 제공 전에는 외장 디스크나 다른 NAS에 Proxmox 백업을 한 벌 더 둔다. ZFS와 ECC도 삭제·랜섬웨어까지 되돌리지는 않는다.

ENAS의 매력은 Proxmox용 iSCSI와 ZFS를 한 플랫폼에서 다루는 점이다. 실제 체감은 포트 속도보다 스위치 경로, VLAN 분리, 세션 복구, 별도 백업이 좌우한다. 네 가지를 확인한 뒤 작은 VM부터 옮긴다.

- [Ubiquiti Enterprise NAS 공식 발표](https://blog.ui.com/article/introducing-enterprise-nas?from=%2Fblog%2Fintroducing-all-new-g6-entry-lineup)
- [Proxmox VE 공식 문서](https://pve.proxmox.com/pve-docs/pve-storage-iscsi.html)

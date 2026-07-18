---
layout: post
title: "Proxmox VE 9.2 업그레이드 - VE 8 홈서버에서 우선 확인할 ZFS·네트워크 점검"
description: "Proxmox VE 8 홈서버를 Proxmox VE 9.2로 올리기 전 pve8to9 검사, VM 백업, ZFS와 네트워크 상태를 확인하고 실패 지점을 줄이는 실전 절차다."
date: 2026-07-18
tags: [Proxmox, HomeLab, 홈서버, ZFS, NAS보안]
comments: true
share: true
---

![Proxmox VE 9.2 홈서버 업그레이드](/assets/images/2026/07/proxmox-ve-9-2-upgrade-homelab.png)

Proxmox VE 8 홈서버는 `pve8to9` 검사와 외부 백업을 확인한 뒤 Proxmox VE 9.2로 올리는 게 안전하다. 2026년 5월 21일 공개된 9.2는 Debian 13.5, QEMU 11.0, LXC 7.0, ZFS 2.4 계열을 포함한다. 내 환경은 미니 PC 1대, ZFS 데이터 풀, Home Assistant VM과 Ubuntu 컨테이너가 있는 단일 노드다. 클러스터 업그레이드가 아니라 집에서 실제로 자주 쓰는 구성에 맞춘 순서다.

## 업그레이드 전에 확인한 상태

웹 UI에서 `Node → Updates`를 열기 전에 SSH로 현재 버전과 저장소를 기록했다. 이 기록이 있어야 문제가 생겼을 때 “업데이트 전부터 그랬는지”를 구분할 수 있다.

| 확인 항목 | 통과 기준 |
|---|---|
| Proxmox | VE 8 최신 패치 상태 |
| 백업 | VM·컨테이너를 NAS 또는 USB에 복구 가능한 형태로 보관 |
| ZFS | `zpool status`에 DEGRADED나 오류가 없음 |
| 네트워크 | 브리지 이름과 고정 IP를 기록 |
| 저장소 | root와 local-zfs에 여유 공간 확보 |

화면에서 보이는 VM 스냅샷은 백업이 아니다. 스냅샷은 같은 풀의 디스크가 망가지면 함께 사라질 수 있어서, 업그레이드 직전에 NAS의 NFS 공유나 외장 디스크로 `vzdump` 백업을 하나 남겼다.

## `pve8to9` 검사 실행

이 명령은 패키지를 바로 바꾸지 않고 VE 9 업그레이드를 막을 수 있는 설정을 점검한다.

```bash
sudo apt update
sudo apt full-upgrade
sudo pve8to9 --full
```

여기서 멈춰야 했던 항목은 오래된 저장소와 손으로 수정한 네트워크 설정이었다. `/etc/apt/sources.list.d/`에 VE 8용 `bullseye`나 `bookworm` 저장소가 섞여 있으면 경고를 무시하지 않는다. 구독이 없는 테스트 노드라면 enterprise 저장소를 억지로 살려 두기보다 자신이 실제로 쓰는 저장소만 남긴다.

```bash
grep -R "^[^#].*deb" /etc/apt/sources.list /etc/apt/sources.list.d/
ip -br addr
cat /etc/network/interfaces
```

`vmbr0`의 이름을 바꾸거나 DHCP로 돌리는 작업은 업그레이드와 별개다. 네트워크를 같이 고치면 재부팅 후 웹 UI가 안 열린 원인을 찾기 어려워진다.

## 백업과 ZFS 상태 확인

Home Assistant VM은 NAS의 NFS 저장소로 백업하고, 컨테이너 설정 디렉터리는 별도로 tar 파일로 묶었다. 백업 완료 메시지만 보지 말고 실제 파일 크기와 목록을 확인한다.

```bash
sudo vzdump 101 --mode snapshot --compress zstd \
  --dumpdir /mnt/pve/nas-backup/dump
sudo zpool status -v
sudo zfs list -o name,used,avail,mountpoint
```

`zpool status -v`에 checksum 오류가 남아 있으면 업그레이드보다 디스크와 케이블 점검이 우선이다. ZFS 풀이 거의 찬 상태에서 패키지 작업을 진행하는 것도 피한다. VM 디스크를 thin pool에 두었다면 `local-zfs`의 실제 여유 공간도 함께 봐야 한다.

## 업그레이드 후 확인 순서

재부팅 뒤에는 웹 UI가 열린다는 이유만으로 끝내지 않았다. 버전, 브리지, 스토리지, 게스트 부팅을 각각 확인했다.

```bash
pveversion --verbose
ip -br link
sudo pvesm status
sudo zpool status
qm list
pct list
```

Home Assistant VM의 네트워크가 올라와도 USB Zigbee 동글이 사라질 수 있다. `lsusb`와 VM 하드웨어의 USB 패스스루 항목을 비교하고, 자동 부팅 순서가 바뀌지 않았는지도 확인한다. Ubuntu 컨테이너에서는 Docker 서비스와 볼륨 마운트가 정상인지 직접 본다.

```bash
sudo systemctl --failed
sudo journalctl -b -p warning..err --no-pager
```

## 내가 정한 중단 기준

`pve8to9`에 치명적 오류가 남아 있거나, 백업을 다른 저장장치에서 확인하지 못했거나, ZFS에 기존 오류가 있으면 그날 업그레이드하지 않는다. 단일 노드 홈서버는 장애 조치 대상이 없어서 “일단 재부팅”의 대가가 생각보다 크다.

핵심은 VE 9.2의 새 기능을 바로 쓰는 게 아니다. `pve8to9` 검사, 복구 가능한 백업, ZFS 정상 상태, `vmbr0` 기록까지 네 가지를 확인하면 업그레이드 실패가 설정 문제인지 하드웨어 문제인지 좁힐 수 있다. Proxmox 공식 발표의 9.2 변경점은 [Proxmox VE 9.2 릴리스 안내](https://proxmox.com/en/about/company-details/press-releases/proxmox-virtual-environment-9-2)에서 확인할 수 있다.

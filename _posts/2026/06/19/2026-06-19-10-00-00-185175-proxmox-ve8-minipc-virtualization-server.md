---
layout: post
title: "Proxmox VE 8 설치 — 미니PC로 가상화 홈서버 만들기"
description: "Proxmox VE 8을 미니PC에 설치하고 Home Assistant OS와 Ubuntu 서버 VM을 함께 돌리는 방법. 설치부터 VM 생성, 네트워크 브릿지 설정까지."
date: 2026-06-19
tags: [Proxmox, 홈서버, 홈랩, 가상화]
comments: true
share: true
---
![Proxmox 가상화 홈서버 미니PC](https://images.unsplash.com/photo-1518770660439-4636190af475?w=800&q=80)

미니PC 하나로 여러 서버를 돌리고 싶을 때 Proxmox가 답이다. Debian 기반 하이퍼바이저인데 KVM 가상화와 LXC 컨테이너를 모두 지원한다. 웹 UI가 있어서 VM 생성이나 스냅샷을 마우스로 할 수 있다.

실제로 써보면 VMware 수준의 기능을 공짜로 쓸 수 있어서 홈랩 용도로 인기가 많다. [앞에서 Home Assistant 설치 방법 비교]({% post_url 2026-06-10-10-00-00-074070-home-assistant-install-rpi-nas-vm-compare %})할 때 Proxmox를 언급했는데, 이번에 실제로 설치한다.

## 환경 정보

| 항목 | 버전/모델 |
|------|----------|
| 미니PC 모델 | Beelink EQ12 Pro (N100, 16GB RAM, 500GB SSD) |
| Proxmox VE 버전 | 8.2.x |
| 설치 방식 | USB 부팅 설치 |

N100 미니PC는 저전력(6~15W)이면서 가상화 기능(VT-x/VT-d)을 지원해서 홈서버로 인기다. 2025~2026년에 많이 쓰인다.

## 설치 USB 만들기

```bash
# Proxmox VE ISO 다운로드
# https://www.proxmox.com/en/downloads 에서 proxmox-ve_8.x.iso 다운로드

# macOS에서 USB 굽기
diskutil list    # USB 디스크 확인 (예: /dev/disk4)
diskutil unmountDisk /dev/disk4
sudo dd if=proxmox-ve_8.x.iso of=/dev/rdisk4 bs=4m status=progress
```

Windows에서는 [Rufus](https://rufus.ie)를 쓴다.

## 설치

미니PC에 USB 꽂고 부팅. BIOS에서 USB 부팅 순서를 앞으로 설정해야 한다.

설치 화면:
1. **Graphical** 설치 선택
2. 설치 대상 디스크 선택 (내장 SSD)
3. 국가/언어/키보드: Korea 선택
4. 관리자 이메일과 비밀번호 설정
5. 네트워크 설정:
   - 관리 IP: `192.168.1.200/24` (고정 IP로 설정)
   - 게이트웨이: `192.168.1.1`
   - DNS: `8.8.8.8`

설치 완료 후 재부팅. USB 제거.

웹 UI 접속: `https://192.168.1.200:8006`

처음 접속하면 인증서 경고가 뜬다. 그냥 진행(고급 → 계속 진행)하면 된다. 나중에 Let's Encrypt 인증서로 바꿀 수 있다.

## 구독 경고 제거 (선택)

Proxmox는 엔터프라이즈 구독 없이도 쓸 수 있지만, 로그인할 때마다 구독 경고가 뜬다.

```bash
# Proxmox 서버에 SSH 접속 후
sed -i.bak "s/data.status !== 'Active'/false/g" /usr/share/javascript/proxmox-widget-toolkit/proxmoxlib.js
systemctl restart pveproxy
```

비공식 방법이지만 홈랩 용도로 널리 쓰인다.

## 무료 업데이트 저장소 설정

엔터프라이즈 저장소를 비활성화하고 무료 저장소를 활성화한다.

```bash
# 엔터프라이즈 저장소 비활성화
echo "# deb https://enterprise.proxmox.com/debian/pve bookworm pve-enterprise" > /etc/apt/sources.list.d/pve-enterprise.list

# 무료 저장소 추가
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" >> /etc/apt/sources.list

# 업데이트
apt update && apt upgrade -y
```

## VM 생성 — Home Assistant OS

```bash
# HAOS qcow2 이미지 다운로드
cd /tmp
wget https://github.com/home-assistant/operating-system/releases/latest/download/haos_ova-stable.qcow2.xz
xz -d haos_ova-stable.qcow2.xz
```

웹 UI에서 VM 생성 (우상단 Create VM):

```
General: ID = 100, Name = homeassistant
OS: Do not use any media
System: BIOS = OVMF(UEFI), Machine = q35, EFI Storage 선택
Disks: 기본 디스크 삭제 (나중에 이미지 import)
CPU: 2 cores
Memory: 4096 MB
Network: virtio, vmbr0 브릿지
```

VM 생성 후 이미지 임포트:

```bash
# HAOS 이미지를 VM 100의 디스크로 임포트
qm importdisk 100 /tmp/haos_ova-stable.qcow2 local-lvm

# 웹 UI에서 Hardware → Unused Disk 선택 → Edit → Add
# Bus/Device: SATA 0으로 추가
# Options → Boot Order → SATA0을 첫 번째로 설정
```

![Proxmox VM 생성 화면](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)

VM 시작하면 Home Assistant가 부팅된다.

## 삽질했던 부분

**EFI 파티션 설정**: HAOS는 UEFI 부팅이 필요하다. VM 생성 시 BIOS를 SeaBIOS로 하면 부팅이 안 된다. OVMF(UEFI)로 반드시 설정해야 한다.

**네트워크 브릿지**: 기본 브릿지 `vmbr0`이 미니PC의 물리 네트워크 카드와 연결돼 있어야 VM들이 홈 네트워크에 접속된다. Proxmox 설치 시 자동으로 설정되는데, 네트워크 카드가 여러 개인 경우 어떤 카드가 `vmbr0`에 연결됐는지 확인해야 한다.

**N100 CPU 특이사항**: 일부 N100 모델에서 CPU 전원 관리 설정 때문에 VM이 불안정한 경우가 있다. BIOS에서 C-State를 비활성화하면 안정적이다.

## 한 줄 정리

Proxmox를 한 번 올려두면 미니PC 하나에서 HAOS, Ubuntu 서버, 각종 VM을 동시에 돌릴 수 있다. 홈랩 입문으로 N100 미니PC + Proxmox 조합이 2026년 기준 가장 가성비 좋은 선택이다.

---
다음엔 [Cloudflare Tunnel — 포트 포워딩 없이 집 서버 서비스 외부 공개하기]({% post_url 2026-06-20-10-00-00-197520-cloudflare-tunnel-no-port-forwarding %})를 다룬다.

**참고 링크**
- [Proxmox VE 공식 사이트](https://www.proxmox.com/en/proxmox-virtual-environment/overview)
- [Proxmox 설치 문서](https://pve.proxmox.com/wiki/Installation)
- [Home Assistant HAOS Proxmox 설치 가이드](https://community.home-assistant.io/t/installing-home-assistant-os-using-proxmox-8/201835)
- [Beelink EQ12 Pro 구매 페이지](https://www.bee-link.com/products/beelink-eq12)

---
layout: post
title: "TrueNAS SCALE 설치부터 SMB 공유까지 — ZFS 기반 무료 NAS OS 세팅"
description: "TrueNAS SCALE 24.x를 미니PC나 구형 PC에 설치하고 SMB 파일 공유를 설정하는 방법. ZFS 풀 생성, 데이터셋 설정, Windows SMB 연결까지 실전 가이드."
date: 2026-06-22
tags: [TrueNAS, 홈서버, 홈랩, NAS설정]
comments: true
share: true
---

# TrueNAS SCALE 설치부터 SMB 공유까지 — ZFS 기반 무료 NAS OS 세팅

![TrueNAS SCALE ZFS NAS 서버](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)

Synology NAS 살 돈은 없는데 집에 남는 PC가 있다면 TrueNAS SCALE로 그 PC를 NAS로 만들 수 있다. 소프트웨어는 완전 무료고, ZFS 파일 시스템을 써서 데이터 무결성 측면에서는 Synology보다 앞선다. 단, 설정이 Synology보다 복잡하다.

## TrueNAS SCALE vs TrueNAS CORE

| 항목 | SCALE | CORE |
|------|-------|------|
| 기반 OS | Linux (Debian) | FreeBSD |
| 컨테이너 | Docker/Kubernetes 지원 | 없음 |
| 앱 생태계 | TrueCharts | 제한적 |
| 추천 용도 | 홈랩, 신규 설치 | 기존 CORE 사용자 |

신규 설치라면 SCALE을 쓰는 게 낫다. Docker 지원이 되고, 업데이트도 SCALE에 더 집중돼 있다.

## 환경 정보

| 항목 | 버전/모델 |
|------|----------|
| 하드웨어 | 구형 데스크탑 PC (i5-6500, 16GB RAM) |
| TrueNAS SCALE 버전 | 24.04 (Dragonfish) |
| 스토리지 | 4TB HDD x 3 (ZFS RAIDZ1) |

## 설치

[TrueNAS SCALE 다운로드](https://www.truenas.com/download-truenas-scale/)에서 ISO 파일 다운로드.

USB에 Rufus(Windows) 또는 dd(macOS/Linux)로 굽고 부팅.

설치 과정:
1. 설치 대상 디스크 선택 (OS 설치용 디스크 - 별도로 작은 SSD 권장)
2. 관리자 비밀번호 설정
3. 완료 후 재부팅

웹 UI 접속: `http://[IP주소]` (설치 완료 화면에서 IP 확인)

## ZFS 풀 생성

TrueNAS의 핵심은 ZFS다. ZFS 풀(Pool)을 만들어야 데이터를 저장할 수 있다.

**Storage → Create Pool**:

```
풀 이름: tank (관례적으로 많이 쓰는 이름)
레이아웃:
  - Stripe: RAID 없음, 가장 많은 용량, 무결성 없음
  - Mirror: RAID 1 유사, 디스크 하나 고장 허용
  - RAIDZ1: RAID 5 유사, 디스크 하나 고장 허용, 최소 3개 디스크
  - RAIDZ2: RAID 6 유사, 디스크 두 개 고장 허용, 최소 4개 디스크
```

4TB HDD 3개로 RAIDZ1을 설정하면 실제 사용 가능 용량은 약 8TB(4TB×2)다. 1개 드라이브 고장 허용.

## 데이터셋 생성

풀 안에 데이터셋(Dataset)을 만든다. 데이터셋은 공유 폴더와 권한을 분리해서 관리할 수 있게 해준다.

```
Storage → tank → Add Dataset
이름: media
이름: documents
이름: backup
```

각 데이터셋에 압축(LZ4), 중복 제거 등 ZFS 특성을 개별 설정할 수 있다.

## SMB 파일 공유 설정

Windows와 Mac에서 공유 폴더로 접근하려면 SMB(Samba) 공유를 설정한다.

**사용자 생성**:
```
Credentials → Local Users → Add
사용자명: nas_user
비밀번호: (설정)
홈 디렉터리: /mnt/tank
```

**SMB 공유 설정**:

```
Shares → Windows (SMB) Shares → Add
경로: /mnt/tank/media
이름: media
Purpose: No presets
```

서비스 활성화:
```
System → Services → SMB → Start Automatically 체크
```

![TrueNAS SCALE SMB 공유 설정](https://images.unsplash.com/photo-1451187580094-2cfb76b7c9b7?w=800&q=80)

## Windows에서 SMB 연결

파일 탐색기 주소창에:
```
\\192.168.1.200\media
```
사용자명과 비밀번호 입력하면 연결된다.

네트워크 드라이브로 고정하려면:
```
내 PC → 네트워크 드라이브 연결 → \\192.168.1.200\media
로그인 시 다시 연결 체크
```

## 데이터 무결성 — ZFS Scrub

ZFS 스크럽은 전체 데이터의 체크섬을 확인해서 비트 부패(Bit Rot)를 검출한다. 월 1회 자동으로 실행하도록 설정하는 게 좋다.

```
Data Protection → Scrub Tasks → Add
풀: tank
일정: 매월 1일 새벽 2시
```

스크럽 중에는 디스크 I/O가 증가하므로 새벽 시간대가 낫다.

## 삽질했던 부분

**ZFS RAM 요구량**: ZFS는 메모리를 많이 쓴다. 최소 8GB 권장인데, 데이터가 많을수록 RAM을 더 써서 캐시 효율이 높아진다. 4GB 이하에서는 성능이 눈에 띄게 낮아진다.

**OS 디스크를 데이터 디스크와 혼용하면 안 됨**: TrueNAS는 OS용 디스크와 데이터용 디스크를 분리하는 걸 강력히 권장한다. OS 디스크에 ZFS 풀을 만들면 나중에 OS 재설치할 때 데이터가 위험해진다.

**SMB 권한 문제**: ACL(접근 제어 목록) 설정이 Linux 방식(POSIX)과 Windows 방식(NFSv4)이 다르다. 기본값을 쓰면 Windows에서 접근할 때 권한 오류가 날 수 있다. 데이터셋 생성 시 ACL 유형을 명시적으로 `SMB/Windows`로 설정하는 게 낫다.

## 한 줄 정리

TrueNAS SCALE은 무료로 쓸 수 있는 ZFS 기반 NAS OS 중 가장 완성도가 높다. 남는 PC가 있다면 Synology 살 돈을 아끼면서 더 강력한 기능을 쓸 수 있다.

---
다음엔 [Nginx Proxy Manager로 도메인·HTTPS 한 번에 관리하기]({% post_url 2026-06-23-10-00-00-234555-nginx-proxy-manager-domain-https %})를 다룬다.

**참고 링크**
- [TrueNAS SCALE 공식 사이트](https://www.truenas.com/truenas-scale/)
- [TrueNAS SCALE 설치 가이드](https://www.truenas.com/docs/scale/gettingstarted/install/)
- [ZFS 입문 가이드](https://openzfs.github.io/openzfs-docs/Getting%20Started/)

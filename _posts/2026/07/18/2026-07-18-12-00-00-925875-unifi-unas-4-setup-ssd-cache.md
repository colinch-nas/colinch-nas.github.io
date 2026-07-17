---
layout: post
title: "Ubiquiti UNAS 4 초기 설정 - 2.5GbE 공유와 NVMe 캐시를 안전하게 쓰는 법"
description: "Ubiquiti UNAS 4의 4베이·2.5GbE·M.2 NVMe 캐시 구성을 확인하고, UniFi Drive 공유 드라이브와 SMB 접속을 실제 홈서버 기준으로 설정하는 순서다."
date: 2026-07-18
tags: [Ubiquiti, UNAS, NAS설정, 홈서버, 백업전략]
comments: true
share: true
---

![Ubiquiti UNAS 4 홈서버와 NVMe 캐시](/assets/images/unifi-unas-4-home-server.png)

Ubiquiti UNAS 4는 4개의 2.5·3.5인치 드라이브 베이, 2개의 M.2 NVMe 슬롯, 2.5GbE를 넣은 데스크톱 NAS다. 2026년 7월 18일 공식 사양을 확인해보니, UniFi 네트워크를 이미 쓰는 집이라면 설치 난도는 낮지만 Synology처럼 앱을 이것저것 올리는 NAS와는 방향이 다르다. 파일 공유와 UniFi Drive 중심으로 쓸 사람에게 맞는다.

## UNAS 4에서 먼저 확인할 것

| 항목 | 확인한 사양 | 실제 판단 |
|---|---|---|
| 드라이브 | 2.5·3.5인치 4개 | RAID 구성에 따라 usable 용량이 달라진다 |
| 네트워크 | 2.5GbE, USB-C | 공유기·스위치도 2.5GbE여야 속도가 난다 |
| 캐시 | M.2 NVMe 2개 | 작은 파일과 VM에 유리하지만 백업은 아니다 |
| 접근 | 웹, SMB, NFS, VPN | 외부 공개는 VPN을 우선한다 |

처음엔 NVMe부터 꽂으면 빠를 줄 알았는데, 캐시는 큰 동영상 순차 복사보다 자주 여는 작은 파일과 VM 랜덤 I/O에 맞다. Ubiquiti 문서도 큰 순차 파일은 캐시를 우회한다고 설명한다. 그래서 사진 원본과 Jellyfin 영상 저장소라면 HDD 풀을 먼저 안정화하고 캐시는 나중에 추가하는 편이 낫다.

## 초기 설정 순서

UNAS 4에 디스크를 넣고 UniFi Network 앱이나 Site Manager에서 장치를 채택한다. 관리자 계정에는 긴 비밀번호와 2단계 인증을 적용한다. 공유기에서 UNAS 관리 화면 포트를 외부로 포워딩하지 않는다.

UniFi Drive에서 `All Files → New Shared Drive`를 열고 공유 이름을 만든다. 가족 사진용이라면 `family-photos`, 백업용이라면 `pc-backup`처럼 목적을 이름에 남긴다. 사용자별 권한은 공유 드라이브 생성 단계에서 나누고, 모든 사람에게 관리자 권한을 주지 않는다.

Windows에서는 파일 탐색기 주소창에 다음처럼 입력한다.

```text
\\UNAS-IP\family-photos
```

macOS는 Finder에서 `이동 → 서버에 연결`을 선택한 뒤 다음 주소를 쓴다.

```text
smb://UNAS-IP/family-photos
```

내부망에서 SMB가 붙은 뒤에 VPN으로 다시 접속해본다. 외부에서 `https://` 주소를 바로 열어 관리 화면까지 노출하는 방식은 피한다. Ubiquiti 공식 문서 기준 UNAS 드라이브는 VPN을 통한 SMB·NFS 접근을 지원하므로, 공유기 VPN이나 UniFi Gateway의 VPN을 먼저 구성하는 흐름이 안전하다.

## NVMe 캐시는 나중에 추가한다

M.2 슬롯에는 가능하면 같은 모델·같은 용량의 NVMe SSD 두 개를 사용한다. UniFi Drive의 저장소 설정에서 SSD Cache를 추가한 뒤 상태가 정상인지 확인한다. 캐시는 파일의 원본이 아니므로, SSD 두 개를 넣었다고 백업이 생기는 것은 아니다.

내가 잡은 운영 순서는 `HDD 풀 생성 → 공유 드라이브 생성 → PC 한 대에서 읽기·쓰기 확인 → 백업 대상 지정 → NVMe 캐시 추가`다. 캐시를 먼저 만들면 장애 원인이 HDD인지 SSD인지 구분하기 어려워진다. 설정 후에는 5GB 정도의 테스트 파일을 복사하고, 다른 계정으로 접근했을 때 권한이 막히는지도 확인한다.

## 구매 전 현실적인 체크리스트

- UniFi 장비가 없다면 관리 편의성이 가격 차이를 상쇄하는지 계산한다.
- 2.5GbE 스위치와 랜 케이블이 없으면 기존 1GbE 속도에 머문다.
- NAS 안의 RAID는 삭제·화재·랜섬웨어를 막는 백업이 아니다.
- USB 디스크나 다른 NAS로 주기적인 외부 백업을 한 번 더 만든다.
- UniFi Drive는 자체 호스팅 앱 플랫폼보다 파일 공유에 초점을 둔다.

UNAS 4는 Plex, Immich, Docker를 한 장비에서 다양하게 돌리려는 사람보다 UniFi 네트워크 안에서 조용한 파일 서버를 원하는 사람에게 맞다. 4베이와 NVMe 캐시는 매력적이지만, 캐시보다 먼저 2.5GbE 구간과 백업 경로를 완성해야 체감 성능과 안전성을 같이 얻을 수 있다.

자료: [Ubiquiti UNAS 4 공식 사양](https://eu.store.ui.com/eu/en/products/unas-4), [UNAS 드라이브 접근 방법](https://help.ui.com/hc/en-us/articles/39670142044567-Accessing-UniFi-NAS-UNAS-Drives), [UNAS/ENAS SSD 캐시 설정](https://help.ui.com/hc/en-us/articles/34801264699927-Adding-SSD-Cache-on-UNAS-ENAS)

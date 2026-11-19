---
layout: post
title: "NAS SMB3 Multichannel vs Link Aggregation - DSM 7.4·QTS 5.2 사용 규모별 호환성과 총비용"
description: "Synology DSM 7.4·QNAP QTS 5.2에서 SMB3 Multichannel과 Link Aggregation 중 무엇을 선택할지, PC·스위치·사용자 수별 호환 조건과 구매 보류 기준을 정리한다."
date: 2026-11-20
tags: [Synology, QNAP, DSM, HomeLab, 네트워크]
comments: true
share: true
---

![NAS와 멀티기가 네트워크 스위치](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

*이 그림에서 볼 것은 NAS 포트 하나의 속도가 아니라 NAS·스위치·PC 사이의 전체 경로다.*

PC 한 대가 NAS의 여러 포트를 동시에 써야 한다면 SMB3 Multichannel이 맞고, 여러 PC가 한 NAS를 나눠 쓰는 환경이면 Link Aggregation이 더 단순하다. 두 기능을 2.5GbE 업그레이드의 필수 조건처럼 사면 안 된다. 1~2명이 문서와 사진을 다루거나 무선으로 접속한다면 기존 1GbE를 유지하는 편이 총비용이 낮다. 이 글은 DSM 7.4와 QTS 5.2의 공식 문서를 2026년 11월 20일 확인한 기준이며, 특정 장비를 직접 측정한 결과는 아니다.

## 두 기능의 차이

| 선택 | 잘 맞는 환경 | 꼭 필요한 조건 | 피해야 할 오해 |
|---|---|---|---|
| SMB3 Multichannel | PC 1대의 대용량 편집·백업 | NAS와 PC 양쪽의 복수 NIC 또는 RSS 지원, SMB 3 클라이언트 | LACP 스위치 설정만으로 활성화되지 않음 |
| Link Aggregation | PC 여러 대가 동시에 NAS 사용 | NAS·스위치 양쪽의 포트 트렁킹 지원 | PC 한 대의 단일 복사를 2배로 만들지 않음 |
| 단일 2.5GbE | 1~2명, 일반 파일·사진 | NAS와 PC의 양쪽 2.5GbE | NAS 포트만 빠르면 전체가 빨라진다는 생각 |

Synology DSM 7.4의 SMB3 Multichannel은 x86 NAS와 SMB Service 4.15 이상, Windows 8 이상 또는 macOS 11.3 이상 클라이언트가 필요하다. NAS와 PC에 여러 NIC가 있거나 RSS를 지원해야 하며, 두 기능은 함께 켤 수 없다. 암호화된 공유 폴더에서는 쓰기 성능이 떨어질 수도 있다. [공식 사양](https://www.synology.com/en-global/dsm/7.4/software_spec/file_services_protocols)과 [SMB 설정 안내](https://kb.synology.com/en-us/DSM/help/SMBService/smbservice_smb_settings?version=7)를 모델별로 대조해야 한다.

QNAP QTS 5.2는 Microsoft Networking에서 SMB Multichannel을 켤 수 있고 QTS 5.1 이상에서 지원된다. 자동 선택은 비슷한 속도의 어댑터를 고르고, 수동 선택은 같은 속도의 포트를 지정한다. [QNAP 공식 절차](https://www.qnap.com/en/how-to/faq/article/how-do-i-configure-smb-multichannel-on-qts-51x-and-later-versions)는 Windows 8.1 이상, Windows Server 2012 이상, macOS Big Sur 11.3 이상을 안내한다.

## 사용 규모별 구매 판단

가상 사례로 4베이 NAS를 1명이 사진 보관에 쓰면 2.5GbE NIC와 관리형 스위치를 추가해도 HDD·무선 구간이 병목이라 체감이 작다. 반대로 4명이 영상을 읽고 한 PC가 NAS로 백업한다면 SMB3 Multichannel을 검토할 수 있다. 8명 사무실처럼 클라이언트가 많으면 LACP가 유리하지만, 단일 사용자의 복사 속도 보장은 아니다.

구매 전에는 아래 순서만 확인하면 된다.

- NAS가 x86인지, DSM·QTS 버전과 SMB Service 버전이 조건을 만족하는지 확인한다.
- PC와 NAS에 실제로 두 개 이상의 유선 경로가 있는지 확인한다. Wi-Fi와 유선을 섞어 안정적인 대역폭을 기대하지 않는다.
- SMB3 Multichannel을 쓸 때는 Link Aggregation을 끄고, LACP를 쓸 때는 스위치와 NAS의 트렁킹 모드를 맞춘다.
- NAS의 HDD 배열, SMB 암호화, 작은 파일 비율을 포함해 복사 속도를 판단한다.
- 1~2명 사용·무선 접속·문서 위주라면 랜카드와 스위치 구매를 보류하고 백업 디스크에 예산을 남긴다.

요점은 간단하다. 한 PC의 대용량 작업은 SMB3 Multichannel, 여러 PC의 동시 접속은 Link Aggregation, 일반 가정용 파일 보관은 기존 네트워크 유지가 우선이다. 기능 이름보다 NAS·PC·스위치·운영체제의 조합을 먼저 맞춰야 불필요한 장비 구매를 피할 수 있다.

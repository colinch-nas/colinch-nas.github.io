---
layout: post
title: "Synology RackStation RS826+ 초기 설정 - 4베이 홈랩과 원격 사무실 구성"
description: "2026년 8월 출시된 Synology RS826+를 기준으로 DSM 설치, RAID 구성, 네트워크 확장, 백업까지 실제 운영에 필요한 초기 설정을 정리했다."
date: 2026-08-24
tags: [Synology, NAS설정, HomeLab, DSM, NAS보안]
comments: true
share: true
---

![Synology RackStation RS826+ 4베이 랙마운트 NAS](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

*랙에 넣기 전 디스크 순서와 네트워크 구성을 정해두면 RS826+ 초기 설치가 훨씬 덜 꼬인다.*

Synology가 2026년 8월 12일 RackStation RS826+와 이중 전원 모델 RS826RP+를 출시했다. 1U 4베이, AMD Ryzen V1500B, 4개의 1GbE 포트가 기본이고 PCIe 3.0 슬롯으로 10GbE·25GbE를 추가할 수 있는 모델이다. 집에서 쓰기엔 과하지만, 소규모 사무실이나 여러 대의 홈서버를 한 랙에 모으는 사람에겐 DS923+보다 확장 방향이 분명하다.

## 이 모델을 고를 조건

| 상황 | RS826+ 판단 |
|---|---|
| 사진·문서 백업만 필요 | 2~4베이 DiskStation이 더 합리적 |
| 4개 디스크 RAID와 VM·컨테이너 | 적합 |
| 정전에도 서비스 지속 | RS826RP+의 이중 전원 검토 |
| 10GbE 파일 편집·백업 | PCIe NIC 추가 전제로 적합 |

Synology 발표 기준 순차 읽기·쓰기는 2,000/1,300MB/s 이상이지만, 이 수치는 내부 테스트다. 기본 1GbE 네 포트만 연결하면 실제 단일 클라이언트 속도는 네트워크에 막힌다. 처음부터 10GbE가 필요한 게 아니라면 NIC와 스위치를 함께 사지 말고, 스토리지 사용량이 늘 때 확장하는 편이 낫다.

## DSM 설치 전 준비

환경은 RS826+, DSM 7.4, 12TB CMR HDD 4개, 1GbE 관리망으로 잡았다. 디스크는 제조사 호환 목록을 확인하고, 섞어 쓸 때는 가장 작은 디스크 용량에 맞춰진다. 설치 직후 다음 순서로 진행한다.

1. 디스크를 1~4번 베이에 같은 순서로 장착하고 공유기 또는 스위치에 LAN 1을 연결한다.
2. `find.synology.com`에서 NAS를 찾고 DSM 7.4를 설치한다.
3. 관리자 계정은 `admin`이 아닌 새 이름으로 만들고 2단계 인증을 켠다.
4. `저장소 관리자 → 저장소 풀 → 생성`에서 SHR-2 또는 RAID 6을 선택한다.

4베이에서 SHR-2를 고르면 디스크 두 개 장애까지 버티지만, 사용 가능 용량은 2개 분량으로 줄어든다. 사진과 문서처럼 복구 우선이면 SHR-2, 미디어 캐시처럼 원본이 따로 있는 공간이면 SHR-1이 현실적이다. RAID는 백업이 아니므로 별도 USB나 다른 NAS로 Hyper Backup을 예약해야 한다.

## 네트워크와 서비스 설정

`제어판 → 네트워크 → 네트워크 인터페이스`에서 관리용 IP를 공유기 DHCP 예약으로 고정했다. 포트 집성(LACP)은 스위치도 지원해야 하며, 단일 PC 속도를 2배로 만드는 기능이 아니다. 여러 사용자가 동시에 파일을 읽을 때 이점이 생긴다.

관리 화면은 인터넷에 직접 공개하지 않는다. DSM 방화벽에서 사설망만 허용하고, 외부 접속은 Tailscale 같은 VPN으로 제한한다. Docker 컨테이너와 Virtual Machine Manager를 함께 쓸 계획이면 시스템 볼륨과 서비스 볼륨을 나누고, 업데이트 전 스냅샷 또는 백업 성공 여부를 확인한다.

## 설치 직후 체크리스트

- [ ] 관리자 기본 계정 비활성화 및 2단계 인증
- [ ] DSM 자동 업데이트는 보안 업데이트만 자동 적용
- [ ] SHR/RAID 생성 후 스크럽과 S.M.A.R.T. 검사 예약
- [ ] Hyper Backup으로 다른 저장소에 주 1회 백업
- [ ] 1GbE로 시작한다면 PCIe 10/25GbE 구매를 보류

RS826+는 일반 가정용 NAS라기보다 랙과 확장을 전제로 한 장비다. 다만 4베이 RAID, Docker, VMM을 한 장비에 모으면서 향후 네트워크만 키우고 싶다면 구성 선택지가 깔끔하다. 반대로 디스크 두 개와 단일 사용자만 쓸 계획이라면 랙형이라는 장점보다 소음·전력·부가 장비 비용이 먼저 보일 수 있다.

참고: [Synology RS826+/RS826RP+ 공식 발표](https://www.synology.com/en-global/company/news/article/RS826p), [Synology DSM 7.4 발표](https://www.synology.com/en-global/company/news/article/DSM74)

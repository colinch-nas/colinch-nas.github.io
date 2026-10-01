---
layout: post
title: "NAS 외장 백업 디스크 파일시스템 호환성 - DSM 7.4·QTS 5.2·TrueNAS SCALE 25.10 총비용"
description: "Synology DSM 7.4, QNAP QTS 5.2, TrueNAS SCALE 25.10에서 외장 백업 디스크의 NTFS·exFAT·EXT4 호환성과 복원 비용을 비교한다."
date: 2026-10-01
tags: [Synology, QNAP, TrueNAS, DSM, NAS보안]
comments: true
share: true
---

![NAS 외장 백업 디스크와 복원 호환성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

이 그림에서 볼 것은 디스크 용량이 아니라, 백업을 만든 장비와 복원할 장비가 같은 파일시스템과 백업 형식을 읽을 수 있는지다.

NAS 한 대의 장애에 대비해 외장 HDD를 하나 사려는 1~2인 가정이라면 NTFS가 가장 다루기 쉽다. Windows PC에서도 읽을 수 있기 때문이다. 반대로 NAS 앱의 다중 버전 백업을 그대로 보존하거나 LUN까지 복원해야 한다면 장비별 전용 형식과 EXT4 조건을 우선해야 한다. “어느 NAS에나 꽂으면 복원된다”는 전제는 피해야 한다.

## 파일시스템별 호환성

공식 문서 기준을 2026년 10월 1일에 확인해 비교하면 다음과 같다.

| 형식 | Synology DSM 7.4 | QNAP QTS 5.2 | TrueNAS SCALE 25.10 | 선택 기준 |
|---|---|---|---|---|
| NTFS | 외장 장치 읽기·쓰기 지원 | 외장 저장장치 지원 | 디스크 가져오기 용도 | PC와 번갈아 쓸 때 |
| exFAT | 모델·버전에 따라 라이선스 또는 지원 조건 확인 | exFAT 드라이버 라이선스 필요 | 일반 백업 대상 형식으로 가정하지 않음 | 카메라·Mac·Windows 공유가 꼭 필요할 때 |
| EXT4 | 외장 장치 지원, LUN 백업은 EXT4 필요 | Linux·NAS용으로 지원 | ZFS 데이터 풀의 대체 형식이 아님 | Linux 계열 NAS 한 곳에서 복구할 때 |
| 전용 백업 형식 | `.hbk`는 Hyper Backup 계열 도구 필요 | HBS 3 설정과 버전에 종속 | ZFS 스냅샷·복제는 일반 파일 복사와 다름 | 동일 제품군 복원이 목표일 때 |

Synology는 외장 장치로 ext4, FAT32, NTFS, Btrfs, exFAT, HFS+를 인식하지만, Hyper Backup의 다중 버전 파일은 Hyper Backup·Explorer·Vault 계열 도구로 읽는다. QNAP은 QTS에서 exFAT 사용에 라이선스가 필요하다고 안내한다. TrueNAS는 ZFS 풀을 다른 TrueNAS에 가져오는 흐름과 일반 파일시스템 디스크를 일회성으로 가져오는 흐름이 다르다. 따라서 NTFS 디스크에 저장한 `.hbk`나 HBS 데이터가 TrueNAS에서 자동으로 NAS 전체 복원이 된다고 보면 안 된다.

공식 근거: [Synology Hyper Backup 사양](https://www.synology.com/en-eu/dsm/7.4/software_spec/hyper_backup), [Synology 외장 장치 사양](https://www.synology.com/en-global/dsm/7.4/software_spec/external_devices_printing), [QNAP QTS 외장 저장장치 문서](https://docs.qnap.com/operating-system/qts/5.2.x/qts5.2.x-ug-en-us.pdf), [TrueNAS SCALE 25.10 백업 안내](https://www.truenas.com/docs/scale/25.10/printview/)

## 구매 전 총비용 계산

가상 사례로 8TB 원본과 8TB 외장 HDD를 비교한다. 실제 판매가는 확인하지 않은 계산 가정이다.

| 구성 | 디스크 가정 | 추가 조건 | 판단 비용 |
|---|---:|---|---:|
| NTFS 단일 디스크 | 20만 원 | PC 복원은 쉽지만 앱 메타데이터는 별도 | 20만 원 |
| exFAT 단일 디스크 | 20만 원 | QNAP 라이선스 비용과 모델 지원 확인 | 20만 원+α |
| 전용 백업 형식 | 20만 원 | 같은 NAS 계열 또는 Explorer용 복원 환경 필요 | 20만 원+임시 NAS |
| 외장 디스크 2개 교대 | 40만 원 | 한 개는 분리 보관, 분기별 복원 확인 | 40만 원 |

파일만 복사할 사람은 비싼 전용 백업 형식을 살 이유가 적다. 반대로 권한·버전·패키지 설정까지 살려야 한다면 디스크 가격보다 복원 장비를 빌리거나 임시 NAS를 마련하는 비용이 더 커질 수 있다. RAID는 이 문제를 해결하지 않는다.

## 복원 전에 확인할 체크리스트

- [ ] 백업이 일반 파일인지, `.hbk`·HBS 전용 형식인지 적었다.
- [ ] 복원 대상 NAS의 DSM·QTS·TrueNAS 버전과 필요한 앱을 기록했다.
- [ ] 외장 디스크를 실제 복원 대상 장비에 연결해 목록 파일 하나를 복원했다.
- [ ] 암호화 백업의 암호와 키를 NAS와 다른 곳에 보관했다.
- [ ] 백업 완료 후 외장 HDD를 분리해 랜섬웨어 경로를 끊었다.

짧게 판단하면 Windows와 NAS 사이를 오가는 단순 파일 백업은 NTFS, QNAP·Mac·카메라까지 직접 공유해야 할 때만 exFAT, 특정 NAS의 버전·권한·패키지 복원이 목적이면 전용 형식을 고른다. 구매 전에 “어떤 장비에서 어떤 형식으로 복원할지”를 한 줄로 쓰지 못한다면, 더 큰 디스크보다 먼저 복원 계획을 정해야 한다.

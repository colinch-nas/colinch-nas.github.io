---
layout: post
title: "NAS 외장 백업 디스크 파일시스템 호환성 - DSM 7.4·QTS 5.2·TrueNAS SCALE 25.10 총비용"
description: "Synology DSM 7.4, QNAP QTS 5.2, TrueNAS SCALE 25.10에서 외장 USB 백업 디스크의 NTFS·exFAT·ext4 호환성과 버전 백업·복원 조건, 구매 총비용을 비교한다."
date: 2026-11-06
tags: [Synology, QNAP, TrueNAS, DSM, NAS, 백업복원]
comments: true
share: true
---

![NAS 외장 USB 백업 디스크 파일시스템 비교](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)
*이 그림에서 볼 부분은 NAS 본체보다 백업 디스크의 포맷과 복원할 운영체제가 먼저 정해져야 한다는 점이다.*

외장 디스크를 Windows에서도 읽을지, NAS에서 버전 백업을 유지할지에 따라 포맷이 달라진다. 한두 개 파일을 옮기는 1인 사용자는 NTFS나 exFAT도 가능하지만, QNAP HBS 3의 다중 버전 백업과 Synology LUN 백업까지 고려하면 ext4 또는 NTFS가 더 현실적이다. TrueNAS SCALE 25.10은 USB 디스크를 주 저장소로 쓰는 선택이 아니므로, 자동 백업 장비처럼 구매하면 안 된다.

## 포맷별 선택 기준

| 파일시스템 | 장점 | 놓치기 쉬운 제한 | 맞는 경우 |
|---|---|---|---|
| NTFS | Windows PC에서 바로 읽기 쉬움 | QNAP HBS 버전 관리에는 필요하지만 NAS별 권한·메타데이터 보존은 확인해야 함 | QNAP 백업을 PC에서도 확인할 때 |
| exFAT | Windows·macOS와 이동이 편함 | QNAP HBS 3 다중 버전 백업을 지원하지 않음. 일부 NAS는 추가 패키지·라이선스 조건이 있음 | 단순 파일 복사, 임시 이동 |
| ext4 | Linux·NAS 친화적이고 대용량 백업에 적합 | Windows에서 바로 읽기 어렵고, 디스크를 빼서 확인하려면 별도 환경이 필요함 | NAS에 연결한 채 백업·복원할 때 |
| FAT32 | 기기 호환성이 넓음 | 단일 파일 최대 4GB | 작은 파일 이동 외에는 피함 |

Synology DSM 7.4는 외장 디스크에서 Btrfs, ext3, ext4, FAT32, exFAT, HFS Plus, NTFS를 인식한다. 다만 LUN을 외장 디스크에 백업할 때는 ext4 조건이 붙는다. “디스크가 보인다”와 “원하는 백업 작업이 된다”는 같은 뜻이 아니다. QNAP QTS 5.2도 NTFS·HFS+·FAT32·exFAT·EXT3·EXT4를 외장 저장장치 포맷으로 안내하지만, HBS 3의 다중 버전 백업은 FAT32와 exFAT을 지원하지 않는다.

## 총비용은 디스크 가격보다 복원 경로가 좌우한다

아래는 실제 판매가가 아닌 2026년 11월 6일 기준 가상 계산이다. 8TB 외장 HDD 2개를 각각 18만 원, USB-SATA 케이스 1개를 3만 원, 복원용 Windows PC를 이미 보유한 것으로 가정했다.

| 구성 | 초기 비용 가정 | 복원 방식 | 추가 비용·위험 |
|---|---:|---|---|
| exFAT 단순 복사 | 36만 원 | PC·NAS에서 파일 복사 | 버전 백업·권한·메타데이터 보존이 약함 |
| NTFS + QNAP HBS | 39만 원 | QNAP HBS 또는 PC에서 확인 | QTS 장애 때 HBS 메타데이터와 복원 앱 필요 |
| ext4 + Synology Hyper Backup | 39만 원 | NAS·Hyper Backup Explorer | Windows에서 바로 읽기 어렵고 별도 복원 환경 필요 |
| TrueNAS USB 복사 | 39만 원 | SCALE에서 수동 복사·검증 | 자동화가 제한적이고 USB 연결 장애 범위가 큼 |

구매하지 않아도 되는 경우도 분명하다. 사진과 문서를 이미 다른 NAS나 클라우드에 보관하고, 외장 디스크는 월 1회 오프라인 복사만 할 목적이라면 NAS 전용 파일시스템보다 PC에서 확인하기 쉬운 NTFS가 운영 부담을 줄일 수 있다. 반대로 백업 작업의 버전·암호화·권한을 복원해야 한다면 “어느 PC에서 읽히는가”보다 “원래 백업 프로그램이 복원할 수 있는가”가 우선이다.

## 포맷하기 전 복원 체크리스트

1. 백업 대상이 파일인지, Hyper Backup의 `.hbk`인지, QNAP HBS의 버전 저장소인지 적는다.
2. 복원할 장비가 원래 NAS인지, 새 NAS인지, Windows PC인지 정한다.
3. 4GB를 넘는 파일이 있으면 FAT32를 제외한다.
4. QNAP에서 다중 버전이 필요하면 FAT32·exFAT을 제외한다.
5. Synology LUN 백업이면 ext4 조건을 확인한다.
6. 작은 문서·사진·대용량 파일을 각각 다른 복원 폴더에 복원하고 열어 본다.
7. 백업 성공 알림, 파일 목록, 실제 파일 열기 결과를 따로 기록한다.

TrueNAS 공식 하드웨어 가이드는 USB HDD를 필요한 경우의 기본 백업 용도로만 설명하고, 주 저장소로는 피하라고 안내한다. USB 디스크를 연결한 뒤 명령줄로 복제하고 오프사이트에 보관하는 방식은 가능하지만, 자동화된 스냅샷 저장소와 같은 안정성을 기대하면 안 된다. 어느 플랫폼이든 디스크를 포맷하기 전 기존 자료를 다른 위치에 복사하고, 포맷 직후 샘플 복원까지 끝내야 한다.

짧게 판단하면 Windows에서도 자주 읽을 백업은 NTFS, NAS에 붙여 버전 백업을 운영할 디스크는 해당 프로그램의 공식 조건을 만족하는 ext4·NTFS, 단순 이동 디스크는 exFAT이다. TrueNAS에서는 USB 디스크를 주 풀로 확장하지 말고, 수동 백업과 오프사이트 보관용으로만 계산하는 편이 안전하다. 근거는 [Synology 외장 장치 안내](https://kb.synology.com/en-id/DSM/help/DSM/AdminCenter/system_externaldevice_devicelist), [Synology Hyper Backup 사양](https://www.synology.com/en-sg/dsm/7.4/software_spec/hyper_backup), [QNAP QTS 5.2 외장 디스크 포맷](https://docs.qnap.com/operating-system/qts/5.2.x/en-us/formatting-an-external-storage-disk-or-partition-04D7A1AF.html), [QNAP HBS 버전 백업 제한](https://www.qnap.com/en-us/how-to/faq/article/why-hbs-prompt-warning-failed-to-create-multiple-versions-when-creating-backup-job-to-external-drive), [TrueNAS 하드웨어 가이드](https://cdn.truenas.com/docs/scale/gettingstarted/scalehardwareguide/)다.

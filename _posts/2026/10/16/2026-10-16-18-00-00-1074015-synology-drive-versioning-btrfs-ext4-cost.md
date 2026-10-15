---
layout: post
title: "Synology Drive DSM 7.4 버전 관리 - Btrfs·ext4 저장공간과 백업 총비용"
description: "Synology Drive DSM 7.4에서 파일 버전 32개와 보존 기간을 설정할 때 Btrfs·ext4 저장공간 차이, 사용자 수별 선택과 Hyper Backup 비용을 정리한다."
date: 2026-10-16
tags: [Synology, DSM, SynologyDrive, Btrfs, NAS백업]
comments: true
share: true
---

![Synology Drive 파일 버전 관리와 NAS 저장공간](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

Synology Drive를 1~4명이 문서·사진 작업에 쓰는 경우에는 버전 관리를 켜되, 저장공간을 먼저 계산해야 한다. 새 NAS를 사기 전에 삭제 파일 복구만 필요한지, 랜섬웨어와 NAS 고장까지 대비할지 구분하면 된다. Drive 버전 기록은 파일 실수 복구용이고, NAS 자체 장애를 대신하는 백업은 아니다.

## DSM 7.4에서 달라지는 저장공간

Synology 공식 사양 기준으로 Team Folder는 파일당 최대 32개 과거 버전을 보관하고, 7·30·60·120일 보존이나 FIFO·Intelliversioning을 선택할 수 있다. 다만 파일시스템에 따라 공간을 쓰는 방식이 다르다. Btrfs는 변경분 중심이고, ext4는 현재 파일의 복사본을 먼저 만들 수 있다. 암호화 폴더를 Team Folder로 지정하면 초기 버전 때문에 원본 공간이 두 배가 될 수도 있다.

| 환경 | Drive 버전 관리 판단 | 구매·운영 기준 |
|---|---|---|
| 1인, 문서 위주 | 7일·최대 5버전 | 기존 NAS로 충분한 경우가 많음 |
| 2~4인, 자주 수정하는 문서 | 30일·10~20버전 | Btrfs 볼륨과 여유 공간 우선 |
| 사진·영상 원본 중심 | 짧은 버전 보존 | 대용량 HDD와 별도 백업이 유리 |
| 랜섬웨어·장비 고장까지 대비 | Drive+Hyper Backup | 외장 HDD 또는 다른 NAS 비용 포함 |

표의 수치는 권장 시작값이지 Synology의 공식 최소 요구량은 아니다. 파일 크기와 수정 빈도가 다르므로 실제 사용량을 보고 조정해야 한다.

## 비용을 계산하는 방법

가상 사례로 원본 1TB를 3명이 사용하고, 매주 문서를 고친다고 하자. ext4에서 버전 관리가 원본 복사본을 만들면 시작부터 추가 공간이 크게 잡힐 수 있다. Btrfs도 변경분과 메타데이터 공간이 필요하므로 “1TB 데이터면 1TB 볼륨이면 된다”고 계산하면 안 된다.

```text
필요 여유 공간 = 현재 데이터 + 버전 기록 예상량 + 패키지·휴지통 공간
총비용 = NAS·디스크 + 외장 백업 디스크 + UPS·전기료(선택) + 복구 시간
```

가상으로 1TB 원본에 버전·휴지통 여유 40%를 잡으면 최소 1.4TB 이상을 운영 여유로 본다. 실제 구매에서는 RAID 여유와 디스크 표시 용량 차이까지 고려해 2TB급보다 큰 구성을 검토한다. 반대로 파일을 거의 수정하지 않는 1인 사용자는 버전 수를 크게 잡으려고 SSD 캐시나 상위 NAS를 추가 구매할 필요가 적다.

## 설정 전 체크리스트

- [ ] DSM 7.4에서 공유 폴더의 파일시스템이 Btrfs인지 ext4인지 확인했다.
- [ ] Synology Drive Admin Console에서 Team Folder만 선택했다.
- [ ] 보존 기간과 최대 버전을 정하고 2주 뒤 실제 `File versions` 사용량을 확인한다.
- [ ] 삭제 파일을 Version Explorer에서 별도 폴더로 복원해 권한과 파일명을 확인한다.
- [ ] Hyper Backup으로 중요한 공유 폴더와 Drive 설정을 외장 HDD나 다른 NAS에도 저장한다.
- [ ] 백업 암호와 복원에 쓸 관리자 계정을 NAS와 다른 장소에 기록했다.

Drive의 Version Explorer는 관리자가 날짜·시간을 지정해 수정·삭제 파일을 복원할 수 있다. 그러나 NAS가 고장 나면 그 버전 데이터도 함께 접근할 수 없다. 따라서 1~2인 문서용은 Drive 버전 관리와 주기적 Hyper Backup 조합이면 충분한 경우가 많고, 가족 사진·업무 원본은 분리된 백업 장치까지 예산에 넣어야 한다.

핵심은 버전 수를 최대로 올리는 것이 아니다. Btrfs·ext4와 암호화 여부를 확인하고, 사용자 수와 수정 빈도에 맞춰 보존 기간을 시작한 뒤 실제 공간 사용량으로 다시 조정하는 것이다. 참고 기준은 [Synology Drive DSM 7.4 기술 사양](https://www.synology.com/en-us/dsm/7.4/software_spec/synology_drive), [버전 관리 공간 안내](https://kb.synology.com/en-sg/DSM/tutorial/Drive_manage_files_consume_storage), [Synology Drive 파일 복원 안내](https://kb.synology.com/en-global/DSM/help/SynologyDrive/drive_admin_console)다.

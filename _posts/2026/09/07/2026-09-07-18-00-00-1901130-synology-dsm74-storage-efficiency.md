---
layout: post
title: "Synology DSM 7.4 Storage Efficiency 설정 — NAS 용량을 줄이기 전에 확인할 조건"
description: "Synology DSM 7.4의 Storage Efficiency로 Btrfs HDD 볼륨의 중복 제거와 압축을 설정하는 방법을 정리한다. DS923+ 지원 조건과 실제로 켜면 안 되는 데이터까지 확인한다."
date: 2026-09-07
tags: [Synology, DSM, NAS설정, 백업전략]
comments: true
share: true
---

![Synology DSM 7.4 Storage Efficiency 개념 이미지](/assets/images/dsm-74-storage-efficiency.png)

그림에서 왼쪽의 같은 데이터 블록이 합쳐지고 압축 파일 묶음이 줄어드는 부분을 보면 된다. DSM 7.4의 Storage Efficiency는 저장 공간을 즉시 늘리는 기능이 아니라, 작업을 실행한 뒤 중복 제거와 압축으로 실제 사용량을 줄이는 기능이다.

## DSM 7.4에서 달라진 점

Synology가 2026년 6월 공개한 DSM 7.4에는 Storage Efficiency가 추가됐다. HDD 볼륨에서는 데이터 중복 제거와 압축을 공유 폴더 단위로 적용할 수 있다. 다만 모든 NAS에서 활성화되는 기능은 아니다. 2026년 9월 기준 DS923+, DS723+, DS423+, DS1825+ 등이 지원 목록에 있다.

| 확인 항목 | 필요한 조건 |
|---|---|
| DSM | DSM 7.4 이상 |
| 파일 시스템 | Btrfs 볼륨 |
| 드라이브 | 저장 풀 전체가 Synology HDD 또는 Synology SSD |
| HDD 기능 | 중복 제거 + 압축, 공유 폴더 단위 |
| SSD 기능 | 중복 제거, 볼륨 전체 단위 |
| 상태 | 볼륨이 Healthy이고 사용량 상세 분석 활성화 |

## 설정 전에 할 일

내 환경은 Synology DS923+, DSM 7.4, Synology HDD, SHR, Btrfs다. 이 조건이 아니면 메뉴가 보이지 않거나 옵션이 줄어든다. 지원 모델이어도 타사 HDD가 섞인 저장 풀에서는 기대한 기능이 나오지 않을 수 있다.

중복 제거는 같은 블록을 하나만 남기고, 압축은 반복이 많은 데이터를 작게 저장한다. 동영상·사진은 효과가 작을 수 있어 VM 이미지, ISO, 문서 폴더부터 시험하는 편이 낫다.

## DSM 7.4에서 켜는 순서

설정 화면은 Storage Manager(저장소 관리자)에서 찾는다. 공유 폴더 전체에 무조건 적용하지 말고, 복사본을 쉽게 만들 수 있는 폴더 하나로 시작한다.

1. `Storage Manager` → `Storage`에서 대상 Btrfs 볼륨을 선택한다.
2. 볼륨 우측 상단 메뉴에서 `Configure Storage Efficiency`를 연다.
3. `Usage Detail Analysis`를 먼저 활성화하고 분석이 끝날 때까지 기다린다.
4. `Manual Deduplication`에서 `Run Now`를 눌러 예상 절감량을 확인한다.
5. HDD 볼륨이면 `Deduplication and compression`, `Deduplication only`, `Compression only` 중 하나를 고른다.
6. 적용할 공유 폴더를 선택하고, 작업 예약 시간을 NAS 사용량이 적은 새벽으로 지정한다.

처음부터 자동으로 켜지 않은 이유가 있다. 모든 볼륨에서 작업은 한 번에 하나만 실행되므로 대용량 폴더는 NAS 사용량이 적은 시간에 예약해야 한다. Snapshot Replication을 쓴다면 새 스냅샷 전에 실행하는 편이 유리하다.

## 폴더별 선택 기준

| 데이터 | 권장 설정 | 이유 |
|---|---|---|
| ISO·VM 템플릿 | 중복 제거 | 동일 블록이 반복되는 경우가 많다 |
| 문서·로그 | 압축 | 텍스트 반복이 많아 압축 효과를 기대할 수 있다 |
| 사진·동영상 | 보류 또는 시험 | 원본 자체가 압축돼 효과가 작을 수 있다 |
| 암호화 공유 폴더 | 적용 불가 | Storage Efficiency 대상에서 제외된다 |
| 스냅샷 데이터 | 적용 불가 | 기존 스냅샷 내부 데이터는 대상이 아니다 |

## 여기서 헷갈렸던 제한사항

Storage Efficiency는 Hyper Backup을 대신하지 않는다. 중복 블록을 줄일 뿐 디스크 고장, 랜섬웨어, 실수 삭제를 복구해 주지 않는다. 적용 전 별도 백업을 남기고 작업 중 볼륨 성능을 확인한다.

HDD 볼륨에서는 공유 폴더 단위로 적용하지만 데이터베이스·패키지 데이터·Hot tier 폴더 등은 제외될 수 있다. 메뉴가 보인다고 모든 데이터가 처리되는 것은 아니다.

내 결론은 간단하다. DS923+ 같은 지원 모델에 Synology HDD만 사용하고, Btrfs에 문서·ISO·VM 파일을 보관한다면 DSM 7.4 Storage Efficiency를 작은 폴더부터 시험할 만하다. 타사 디스크가 섞였거나 미디어 파일이 대부분이면 업데이트의 주된 이유로 삼기 어렵다.

### 짧은 점검표

- [ ] DSM 7.4와 지원 모델인지 확인
- [ ] 저장 풀이 Synology HDD 또는 SSD로만 구성됐는지 확인
- [ ] Btrfs·Healthy 상태인지 확인
- [ ] Usage Detail Analysis 후 수동 작업으로 절감량 확인
- [ ] 스냅샷·Hyper Backup과 별개인 기능임을 기억

참고 문서: [Synology DSM 7.4 발표](https://www.synology.com/en-us/company/news/article/dsm74), [Storage Efficiency 설정 방법](https://kb.synology.com/en-global/DSM/help/DSM/StorageManager/volume_btrfs_dedup?version=7), [지원 NAS 모델 목록](https://kb.synology.com/en-my/DSM/tutorial/Which_Synology_NAS_models_support_Storage_Efficiency)

---
layout: post
title: "Synology Active Backup for Business 시스템 복구 - 교체 NAS 호환성과 구매 보류 기준"
description: "Synology DSM 7.4 Active Backup for Business로 NAS 전체를 복구할 때 필요한 교체 모델, 디스크 수·슬롯·HDD·SSD 조건과 구매 전 체크리스트를 정리한다."
date: 2026-09-17
tags: [Synology, DSM, ActiveBackup, NAS보안]
comments: true
share: true
---
![NAS 교체와 백업 복구를 점검하는 서버 장비](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

그림에서 볼 부분은 백업 디스크만이 아니라 복구를 실행할 **대체 NAS와 디스크 구성**까지 함께 준비해야 한다는 점이다.

Synology NAS 전체 시스템을 Active Backup for Business(이하 ABB)로 복구하려는 사람이라면, 고장 뒤 아무 2베이 NAS나 사면 안 된다. DSM 7.4 공식 사양 기준으로 복구 대상은 원본과 같은 모델이거나 같은 제품군의 상위 모델이어야 하고, 디스크 수·슬롯 위치·HDD/SSD 종류도 맞아야 한다. 단순히 파일만 꺼내면 되는 가정용 사용자는 ABB 전체 시스템 복구 장비를 미리 살 필요가 없다. Hyper Backup Explorer로 파일을 복원하는 편이 더 싸고 단순하다.

## 복구용 NAS를 고르는 기준

ABB의 NAS 전체 백업은 설정, 패키지, 사용자 데이터를 포함하지만, 복구 호환성이 자유로운 이미지 백업은 아니다. Synology가 2026년 9월 17일 확인 가능한 DSM 7.4 사양에서 제시한 조건을 구매 기준으로 바꾸면 다음과 같다.

| 원본 NAS 조건 | 대체 NAS에서 확인할 것 | 어긋나면 생기는 문제 |
|---|---|---|
| DS923+ 같은 모델 | 같은 모델 또는 같은 제품군의 상위 모델 | 전체 시스템 복구 대상에서 제외될 수 있음 |
| 4개 드라이브 사용 | 최소 4개 드라이브 장착 가능 | 드라이브 수 부족으로 복구 불가 |
| 디스크 1~4번에 순서대로 장착 | 같은 슬롯 위치에 같은 순서로 장착 | 풀·시스템 구성이 예상과 달라짐 |
| HDD만 사용 | 대체 장비도 HDD만 사용 | SSD와 HDD 혼합 구성은 시스템 복구 제한 |
| SSD만 사용 | 대체 장비도 SSD만 사용 | 디스크 종류 불일치 |

여기서 “상위 모델”은 베이 수가 많다는 뜻만으로 판단하면 안 된다. 공식 문서의 표현은 **같은 제품군의 newer model**이다. 예를 들어 2베이 Plus에서 4베이 Value로 바꾸는 식의 교차 제품군 업그레이드는 복구 가능하다고 가정하지 않는 편이 안전하다. 구매 전 Synology의 해당 모델 패키지 지원 목록과 ABB 사양을 함께 확인해야 한다.

## 구매 전에 계산할 총비용

가상 사례로 DS923+에 4TB HDD 4개가 들어 있고 ABB 백업을 유지한다고 하자. 대체 장비 비용만 비교하면 안 된다.

| 항목 | 계산 기준 | 구매 보류가 가능한 경우 |
|---|---|---|
| 대체 NAS 본체 | 같은 제품군의 복구 가능 모델 | 파일만 복원하면 새 NAS 대신 PC에서 Explorer 사용 |
| 드라이브 | 원본과 같은 수·종류, 같은 슬롯 배치 | 기존 디스크를 재사용할 수 있고 상태가 정상일 때 |
| 백업 저장소 | ABB 데이터의 별도 복사본 | 원본 NAS가 살아 있고 단순 마이그레이션일 때 |
| 복구 시간 | 데이터 용량, 네트워크, 디스크 상태 | 긴 중단을 감수할 수 없으면 사전 테스트 필요 |
| 오프사이트 비용 | 다른 NAS, 클라우드, 이동식 디스크 | 가정용 문서 규모라면 암호화 USB 1개로 시작 가능 |

ABB 데이터를 Hyper Backup으로 다른 위치에 복사할 수는 있지만, 공식 가이드의 제한처럼 복구할 때는 먼저 정상 작동하는 Synology NAS에 데이터를 되돌린 뒤 ABB 저장소를 다시 연결해야 한다. USB 디스크 하나만 있으면 즉시 서비스가 살아나는 구조가 아니다.

## 복구 가능성을 미리 검증하는 체크리스트

실제 장비를 사용하지 않은 일반화된 점검 양식이다. 운영 NAS 모델에 맞춰 값을 채운 뒤 구매를 결정한다.

```text
[원본]
모델:
DSM:
ABB Agent/ABB 버전:
드라이브 개수:
슬롯별 종류: 1=HDD, 2=HDD, 3=HDD, 4=HDD
백업 데이터 위치:

[대체 NAS]
모델:
같은 제품군의 상위 모델인가: 예 / 아니오 / 확인 필요
장착 가능한 드라이브 수:
슬롯별 종류와 순서:
ABB 패키지 지원 여부:

[복구 리허설]
백업 저장소 무결성 검사 완료:
테스트 폴더 3개 복원:
패키지 1개 설정 확인:
공유 폴더 권한 확인:
ABB Storage Relink 완료:
```

두 번째 Synology가 있다면 Snapshot Replication은 ABB 데이터의 빠른 장애 전환에 더 어울린다. 반대로 외장 디스크나 클라우드는 장기 보관과 오프사이트 복사에 적합하지만 복구 단계가 길어진다. Synology는 두 방식을 함께 쓰는 3-2-1 구성을 안내하면서, ABB 데이터에 Cloud Sync나 Shared Folder Sync를 사용하지 말라고 설명한다. 중복 제거 메타데이터가 깨져 재연결이 실패할 수 있기 때문이다.

## 요점 정리

- ABB 전체 시스템 복구용 대체 NAS는 같은 모델 또는 같은 제품군의 상위 모델이어야 한다.
- 원본보다 디스크 수가 적거나 슬롯 순서·HDD/SSD 종류가 다르면 구매해도 전체 복구에 사용할 수 없다.
- 파일 몇 개만 복원할 목적이면 새 NAS를 구매하지 말고 Hyper Backup Explorer를 먼저 검토한다.
- ABB 백업을 다른 위치에 복사한 뒤에는 정상 NAS에 복원하고 Storage Relink까지 해보는 테스트가 필요하다.

참고한 공식 문서는 [Active Backup for Business DSM 7.4 기술 사양](https://www.synology.com/en-sg/dsm/7.4/software_spec/abb), [Active Backup 데이터 백업·재연결 절차](https://kb.synology.com/en-us/DSM/tutorial/How_to_backup_and_relink_Active_Backup_for_Business_with_DSM_backup_packages), [Hyper Backup·ABB·Snapshot Replication 조합 안내](https://kb.synology.com/en-ca/DSM/tutorial/How_to_use_Hyper_Backup_Active_Backup_for_Business_Snapshot_Replication_together)다.

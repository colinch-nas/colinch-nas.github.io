---
layout: post
title: "NAS 오프사이트 백업 복원 비용 - Synology DSM 7.4·Hyper Backup에서 B2·Wasabi 선택"
description: "Synology DSM 7.4 Hyper Backup을 S3 호환 클라우드에 연결할 때 저장료보다 복원 비용과 호환성을 비교해 Backblaze B2·Wasabi 선택 기준을 정리한다."
date: 2026-11-17
tags: [Synology, DSM, HyperBackup, NAS보안, 백업복원, HomeLab]
comments: true
share: true
---

![NAS 오프사이트 백업과 클라우드 복원](/assets/images/2026-09-10-synology-ransomware-backup-strategy.png)
이 그림에서 볼 것은 NAS 안의 원본과 다른 장소의 복사본을 분리하는 구조다.

2TB 이하를 가끔 복원하는 가정용 NAS라면 Backblaze B2가 비용 계산이 쉽고, 복원량이 크거나 매달 자주 내려받는다면 Wasabi가 더 예측 가능하다. 다만 이 글은 클라우드가 1차 백업을 대신한다는 뜻이 아니다. NAS 고장 때 바로 쓸 로컬·USB 복사본이 있고, 클라우드는 화재·도난·랜섬웨어를 분리하는 2차 사본으로 쓰는 경우에 맞다. 실제 장비 연결·복원 테스트가 아니라 2026년 11월 17일 공식 가격과 문서로 계산한 예시다.

## Hyper Backup 호환성부터 확인한다

DSM 7.4의 Hyper Backup은 S3 호환 저장소를 대상으로 백업할 수 있고, 다른 S3 호환 클라우드는 S3 Server 항목에 서비스 주소를 직접 넣는 방식이다. 클라우드 백업을 만들면 같은 볼륨에 로컬 인덱스가 사용되며, Synology 문서 기준 사용량의 1~4%를 잡는 편이 안전하다. B2·Wasabi 계정을 만들기 전에 NAS 볼륨에 이 여유 공간과 임시 복원 공간을 남겨야 한다.

| 판단 기준 | Backblaze B2 | Wasabi Hot Cloud Storage |
|---|---:|---:|
| 저장료 기준 | $6.95/TB·월 | $7.99/TB·월 |
| 다운로드 | 월평균 저장량의 3배까지 무료, 초과 $0.01/GB | egress/API 무료 |
| 최소 조건 | 최소 보관 기간·최소 파일 크기 없음 | Pay-as-you-go 최소 보관 90일, 계정 최소 월 1TB |
| 복원 판단 | 평소 복원이 적고 저장량이 작을 때 유리 | 대량 복원을 자주 하거나 비용 상한이 필요할 때 유리 |

가격은 세금·환율·지역별 조건을 제외한 2026-11-17 확인값이다. B2는 [공식 가격표](https://www.backblaze.com/cloud-storage/pricing), Wasabi는 [공식 가격표](https://wasabi.com/pricing/)를 기준으로 했다.

## 저장료보다 복원량을 넣어 계산한다

아래는 **가상 사례**다. 백업 데이터가 2TB이고 한 달 평균 2TB를 저장한다고 가정했다. 장비값·인터넷 회선료·부가세는 제외했다.

| 월 저장량 | B2 월 저장료 | Wasabi 월 저장료 | 2TB 전체 복원 시 예상 egress |
|---:|---:|---:|---|
| 1TB | $6.95 | $7.99 | 두 서비스 모두 기본 조건에서 추가 요금 없음 |
| 2TB | $13.90 | $15.98 | B2는 6TB까지 무료, Wasabi는 egress 무료 |
| 5TB | $34.75 | $39.95 | B2는 15TB까지 무료, Wasabi는 egress 무료 |

대신 10TB를 저장하면서 장애 때 50TB를 한 번에 복원한다고 가정하면 B2의 무료 한도는 30TB다. 초과 20TB에 단순히 $0.01/GB를 적용하면 약 $200이므로, “저장료가 싼 곳”만 보고 선택하면 안 된다. 실제 청구는 평균 저장량, 파트 수, API 호출, 계약 지역에 따라 달라진다.

## 복원 검증은 백업 직후 한 번 한다

다음 순서로 작은 폴더를 먼저 복원한다.

1. Hyper Backup에서 1~5GB 테스트 폴더와 파일 권한을 선택한다.
2. 원래 경로가 아닌 `restore-test` 공유 폴더로 복원한다.
3. 파일 개수, 해시 또는 사진·문서 열림 여부를 확인한다.
4. Synology 패키지 설정과 Docker 데이터는 일반 파일 복원과 별도로 복원 범위를 기록한다.
5. 새 NAS를 가정해 계정·공유 폴더·권한을 다시 만든 뒤 같은 작업을 한 번 더 한다.

`.hbk` 저장소를 다른 NAS나 USB로 내려받은 경우에도 Hyper Backup의 기존 저장소 복원 메뉴로 열 수 있다. 다만 전체 시스템 복원은 모델 조건이 따로 있으므로, 클라우드에 파일이 있다는 사실만으로 DSM 전체가 자동 복구된다고 생각하면 안 된다. 복원 소요 시간은 클라우드 비용보다 인터넷 업로드·다운로드 회선과 NAS의 암호화 처리 속도가 먼저 결정한다.

## 구매하지 않아도 되는 조건

원본이 500GB 이하이고 주 1회 USB 디스크를 다른 장소에 보관할 수 있다면 클라우드 구독보다 USB 두 개를 번갈아 쓰는 편이 저렴할 수 있다. 반대로 USB를 같은 집에만 두거나, 사진·문서가 2TB를 넘고 집을 비우는 시간이 길다면 월 저장료를 내더라도 오프사이트 사본의 가치가 커진다. 어떤 서비스를 고르든 복원 테스트 기록, 암호화 키 보관 위치, 계정 MFA, 삭제 방지 정책을 함께 남겨야 백업이 실제 복구 수단이 된다.

요점은 간단하다. 작은 저장량과 드문 복원은 B2, 대량 복원 비용의 예측 가능성은 Wasabi가 편하다. 그러나 서비스 선택보다 먼저 로컬 복사본과 테스트 복원을 만들고, Hyper Backup 인덱스 공간과 새 NAS에서의 권한 복원까지 확인해야 한다.

자료: [Synology Hyper Backup 백업 대상](https://kb.synology.com/en-uk/DSM/help/HyperBackup/data_backup_destination?version=7), [Synology Hyper Backup 복원](https://kb.synology.com/index.php/en-global/DSM/help/HyperBackup/restore), [Backblaze B2 가격](https://www.backblaze.com/cloud-storage/pricing), [Wasabi 가격](https://wasabi.com/pricing/)

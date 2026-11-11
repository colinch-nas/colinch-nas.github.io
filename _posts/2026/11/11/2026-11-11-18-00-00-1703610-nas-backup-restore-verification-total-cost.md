---
layout: post
title: "NAS 백업 복원 검증 - DSM 7.4·QTS 5.2·TrueNAS SCALE 25.10 실제 복구 가능성 확인과 총비용"
description: "Synology DSM 7.4, QNAP QTS 5.2, TrueNAS SCALE 25.10에서 백업 성공 알림만 믿지 않고 파일·권한·암호화 복원을 검증하는 절차와 예비 장비 총비용을 정리한다."
date: 2026-11-11
tags: [Synology, QNAP, TrueNAS, DSM, 백업복원, NAS보안, HomeLab]
comments: true
share: true
---

![NAS 백업 복원 검증과 별도 복구 장비](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

그림에서 볼 부분은 백업 대상 NAS와 복원할 별도 경로를 함께 준비해야 한다는 점이다.

1~2명이 사진·문서만 보관하는 환경이라면 새 NAS를 바로 사기보다 기존 NAS의 백업에서 파일 하나를 다른 폴더로 복원해 보는 것이 먼저다. 반대로 Docker 데이터베이스, 암호화 공유 폴더, 여러 사용자의 ACL을 운영한다면 백업 성공 알림만으로는 부족하다. 이 글의 절차는 실제 장비 시험 결과가 아니라 DSM 7.4·QTS 5.2·TrueNAS SCALE 25.10 공식 문서를 바탕으로 만든 검증 예시다.

## 플랫폼별로 확인할 범위

| 환경 | 공식 기능 | 복원 검증에서 놓치기 쉬운 조건 | 별도 구매가 필요한 경우 |
|---|---|---|---|
| Synology DSM 7.4 | Hyper Backup 무결성 검사·버전 복원 | 암호화 백업은 비밀번호·키가 있어야 하고, 검사 중 백업 작업이 대기함 | 1대 NAS만 있다면 USB 디스크부터 검토 |
| QNAP QTS 5.2 | HBS 3 Quick Check·Content Check | Content Check는 데이터를 임시로 내려받아 해시를 비교할 수 있어 시간·클라우드 다운로드 비용이 생김 | NAS-to-NAS 또는 별도 USB 저장소가 필요할 때 |
| TrueNAS SCALE 25.10 | ZFS 스냅샷·복제·클론 | 원본 롤백은 복제 작업을 깨뜨릴 수 있어 테스트는 클론으로 해야 함 | 다른 풀·다른 서버에 복제할 때 |

Synology는 무결성 검사가 백업 데이터와 인덱스 구조를 점검하고, 손상되면 작업을 `Restore Only` 상태로 바꾼다고 안내한다. QNAP HBS 3는 Quick Check와 Content Check를 구분한다. TrueNAS는 스냅샷 클론을 새 데이터셋으로 만들어 원본을 건드리지 않는 복구 경로를 제공한다. 즉 “작업 완료”와 “원하는 파일을 꺼낼 수 있음”은 별도 상태다.

## 돈을 쓰기 전 복원 체크리스트

무료로 할 수 있는 1차 검증은 같은 NAS의 임시 폴더에 사진 1개, 문서 1개, Docker Compose 파일 1개를 복원하는 방식이다. 다만 같은 풀의 고장까지 검증하지는 못한다. 아래 순서에서 하나라도 실패하면 디스크나 UPS를 추가하기 전에 백업 작업부터 고친다.

- [ ] 백업 작업 로그가 성공이고 무결성 검사가 100% 완료됐다.
- [ ] 파일명·한글 경로·대용량 파일을 각각 열었다.
- [ ] 원래 사용자와 다른 테스트 계정으로 ACL이 예상대로 보였다.
- [ ] 암호화 백업의 비밀번호·키를 NAS 밖에서 꺼내 복호화했다.
- [ ] 데이터베이스는 파일 복사가 아니라 애플리케이션 절차로 복원했다.
- [ ] NAS 본체가 없어도 복원할 USB 디스크·다른 NAS·임시 서버 중 하나가 있다.

TrueNAS에서 원본 데이터셋을 바로 롤백하는 선택은 자동 복제를 망가뜨릴 수 있다. 복원 테스트는 스냅샷을 새 데이터셋으로 클론하고 SMB나 NFS로 읽어 보는 편이 안전하다. QNAP은 설정 백업 파일만으로 공유 폴더 안의 데이터를 복원할 수 없고, Synology의 전체 시스템 백업도 같은 모델이나 후속 모델 조건이 붙을 수 있다.

## 가상 총비용 계산

아래 금액은 2026년 11월 가격 조사가 아니라 판단용 가상 사례다. 4TB 외장 HDD 12만 원, 예비 HDD 12만 원, 소형 UPS 10만 원으로 가정하면 별도 복구 자산은 34만 원이다. 평균 20W 장비를 월 720시간, 150원/kWh로 계산한 전력비는 월 약 2,160원이다.

| 선택 | 초기비용 가정 | 복구 범위 | 구매하지 않아도 되는 조건 |
|---|---:|---|---|
| 같은 NAS 임시 폴더 | 0원 | 파일·권한의 1차 확인 | 사진·문서만 있고 장애 시 수동 복구를 감수할 때 |
| 외장 HDD 1개 | 약 12만 원 | NAS 풀 고장과 삭제 사고 일부 | 이미 오프사이트 백업이 있을 때 |
| 외장 HDD + 예비 HDD + UPS | 약 34만 원 | 디스크 교체·정전·복원 리허설 | 장애 대응 시간이 중요하지 않은 1인 환경 |

결정 기준은 “백업 장비가 있는가”가 아니라 “NAS가 사라진 날 몇 시간 안에 필요한 파일을 열 수 있는가”다. 1인 사진 보관은 외장 디스크와 월 1회 샘플 복원으로 충분할 수 있다. 반면 3명 이상이 공유하고 Docker DB까지 운영하면 별도 복구 장비와 분기별 복원 리허설 비용을 예산에 넣는 편이 낫다.

## 짧은 판단

DSM 7.4는 무결성 검사와 복원 이미지를, QTS 5.2는 Quick/Content Check와 별도 복원 작업을, TrueNAS SCALE 25.10은 클론 기반 복원을 기준으로 점검하면 된다. 새 디스크나 UPS 구매는 체크리스트를 통과한 뒤 결정한다. 백업 성공 로그, 실제 파일 열기, 암호화 키 보관, NAS 밖의 복원 경로 네 가지가 갖춰져야 백업을 운영 자산으로 볼 수 있다.

참고한 공식 문서: [Synology Hyper Backup 무결성 검사](https://kb.synology.com/en-in/DSM/tutorial/What_is_backup_integrity_check_for_Hyper_Backup_tasks), [QNAP HBS 3 데이터 무결성 검사](https://docs.qnap.com/application/hybrid-backup-sync/3v21.x/en-us/data-integrity-check-71CA59DE.html), [QNAP 복원 작업 생성](https://docs.qnap.com/application/hybrid-backup-sync/3v21.x/en-us/creating-a-restore-job-from-a-destination-CFAEE27B.html), [TrueNAS SCALE 25.10 스냅샷](https://www.truenas.com/docs/scale/25.10/scaletutorials/dataprotection/periodicsnapshottasksscale/)

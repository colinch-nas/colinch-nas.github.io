---
layout: post
title: "NAS 백업 암호키 복원 조건 - Synology DSM 7.4·QNAP HBS 3·TrueNAS SCALE 25.10 총비용"
description: "NAS 백업 성공 알림만 믿으면 안 된다. Synology DSM 7.4, QNAP HBS 3, TrueNAS SCALE 25.10의 암호키·secret seed 보관 조건과 복원 실패 비용을 비교한다."
date: 2026-11-02
tags: [Synology, QNAP, TrueNAS, 백업, NAS보안]
comments: true
share: true
---

![NAS 백업 암호키와 복원 조건 비교](/assets/images/2026-09-10-synology-ransomware-backup-strategy.png)

그림에서 볼 부분은 NAS와 백업 디스크만으로 복원이 끝나지 않고 암호키가 별도 복원 자산이라는 점이다.

Synology DSM 7.4 Hyper Backup은 비밀번호와 내려받은 키 파일이 함께 필요하다. QNAP HBS 3도 작업 비밀번호를 잃으면 새 장비에서 작업을 재현하기 어렵다. TrueNAS SCALE 25.10은 `Export Password Secret Seed`를 빼면 클라우드 자격증명과 일부 암호화 필드가 복원되지 않는다. 1인 미디어 보관은 암호화를 생략할 수 있지만, 업무 문서라면 키 보관 비용까지 NAS 가격에 넣어야 한다.

## 장비별로 복원에 필요한 것

| 환경 | 백업 성공과 별개로 필요한 값 | 빠지는 경우 | 구매·운영 판단 |
|---|---|---|---|
| Synology DSM 7.4 | 작업 비밀번호·`.pem` 키 | 둘 다 잃으면 복원 불가 | 암호관리자와 오프라인 USB에 이중 보관 |
| QNAP QTS 5.2 | HBS 비밀번호·대상 경로 | 새 NAS에서 작업 재현이 어려움 | 관리자 한 명에게만 맡기지 않음 |
| TrueNAS SCALE 25.10 | 설정·`secret seed`·ZFS 키 | 자격증명·암호화 필드 누락 | passphrase를 별도 문서화 |

공식 문서 기준으로 “설정 백업 파일 하나”와 “복원 가능한 백업 세트”는 다르다. TrueNAS dataset passphrase는 설정 파일에 저장되지 않는다.

## 복원 세트를 만드는 최소 절차

1. 백업 작업 이름, NAS 모델, DSM·QTS·TrueNAS 버전, 저장소 경로를 기록한다.
2. 키와 비밀번호를 분리해 암호관리자·오프라인 USB에 각각 보관한다.
3. 작은 문서와 Docker 설정 파일을 다른 경로에 복원해 열기·소유자·권한을 확인한다.
4. TrueNAS는 `secret seed` 포함 설정, ZFS passphrase, 키 파일을 별도 항목으로 보관한다.

## 총비용은 키를 잃었을 때의 시간까지다

실제 판매가가 아닌 2026년 11월 2일 기준 가상 계산이다. 중요 데이터 4TB, 외장 HDD 2개 20만 원, 암호화 USB와 보관함 5만 원으로 잡으면 키 보관을 포함한 로컬 백업 예산은 25만 원이다. 4베이 백업 NAS는 비싸지만 복원 경로와 호환성을 미리 검증할 수 있다.

구매하지 않아도 된다. 공개 미디어처럼 다시 받을 수 있는 데이터라면 두 번째 NAS보다 외장 디스크와 샘플 복원이 낫다. 가족 사진·세금 문서·Docker 데이터처럼 재생성할 수 없는 파일은 키 사본과 복원용 장비를 예산에 넣는다.

## 복원 전 체크리스트

- [ ] 백업 앱 버전과 원본 NAS 모델을 기록했는가
- [ ] Synology `.pem` 또는 작업 비밀번호를 확인했는가
- [ ] QNAP HBS 작업 이름과 대상 폴더를 문서화했는가
- [ ] TrueNAS 설정 파일에 `secret seed`를 포함했는가
- [ ] ZFS passphrase와 키 파일을 설정 파일과 별도로 보관했는가
- [ ] 샘플 파일을 다른 경로에 복원해 실제로 열어봤는가
- [ ] 키 사본 하나를 NAS와 다른 장소에 두었는가

공식 확인 기준은 [Synology Hyper Backup 복원 안내](https://kb.synology.com/en-uk/DSM/help/HyperBackup/restore?version=7), [Synology 암호화 설정](https://kb.synology.com/en-my/DSM/help/HyperBackup/data_backup_settings?version=7), [QNAP HBS 3 빠른 시작](https://www.qnap.com/en-us/how-to/tutorial/article/hbs-hybrid-backup-sync-quick-start-guide), [TrueNAS SCALE 설정 백업](https://www.truenas.com/docs/scale/systemsettings/advanced/managesysconfig/)이다. 실제 장비에서 복원한 결과가 아니라 2026년 11월 2일 공식 문서와 명시한 가상 비용을 바탕으로 한 판단이다.

한 줄로 정리하면, 암호화 백업은 보안을 올리지만 키를 보관하고 시험 복원할 운영비도 함께 만든다. 1인 사용자는 분리 USB와 샘플 복원으로 시작하고, 여러 사용자가 쓰는 TrueNAS·QNAP·Synology는 키·버전·복원 장비를 한 세트로 관리해야 한다.

---
layout: post
title: "Synology NAS 랜섬웨어 대비 백업 전략 - DSM 7.4에서 3-2-1 구성하기"
description: "Synology DSM 7.4에서 Hyper Backup, Snapshot Replication, 외장 USB를 조합해 랜섬웨어에 대비하는 3-2-1 백업 전략을 실제 메뉴 기준으로 설정한다."
date: 2026-09-10
tags: [Synology, DSM, HyperBackup, NAS보안, 백업전략]
comments: true
share: true
---

![Synology NAS 랜섬웨어 대비 3-2-1 백업 구성](/assets/images/2026-09-10-synology-ransomware-backup-strategy.png)

이 그림에서 볼 부분은 NAS 하나에만 파일을 두지 않고, 외장 디스크와 다른 장소의 백업까지 분리하는 구조다.

Synology NAS를 RAID로 묶었다고 백업이 끝나는 건 아니다. RAID는 디스크 고장에는 유리하지만, 랜섬웨어가 공유 폴더를 암호화하면 그 상태가 그대로 모든 디스크에 기록된다. 내가 DSM 7.4 기준으로 권하는 구성은 원본 NAS, 분리 가능한 USB 백업, 다른 장소의 백업을 두는 3-2-1 방식이다. 여기에 Btrfs 볼륨이면 Snapshot Replication을 한 겹 추가한다.

## 구성 조건과 역할

테스트 환경은 Synology DS923+, DSM 7.4, Btrfs 볼륨, 공유기 뒤의 일반 가정 네트워크로 잡았다. 모델마다 메뉴 이름과 지원 패키지가 조금 다르므로, Snapshot Replication이 설치되지 않는 모델이면 Hyper Backup 중심으로 진행하면 된다.

| 계층 | 도구 | 목적 | 랜섬웨어 대응력 |
|---|---|---|---|
| 원본 보호 | Snapshot Replication | 짧은 주기의 이전 시점 복원 | 빠른 복구 |
| 로컬 백업 | Hyper Backup + USB | NAS와 분리된 버전 보관 | NAS 장애 대응 |
| 오프사이트 | Hyper Backup + C2/S3/다른 NAS | 화재·도난·동시 감염 대비 | 최종 복구본 |

Synology 공식 문서 기준으로 Hyper Backup은 다중 버전, 압축, 암호화, 무결성 검사를 지원하고 로컬 폴더·외장 장치·원격 NAS·S3 호환 저장소를 대상으로 쓸 수 있다. 스냅샷은 백업 자체가 아니라 빠른 시점 복원용이라는 점을 구분해야 한다.

## 1. 공유 폴더에 스냅샷 걸기

패키지 센터에서 `Snapshot Replication`을 설치한 뒤 `스냅샷 → 스냅샷 설정 → 생성`으로 들어간다. 대상 공유 폴더를 고르고, 집에서 문서 작업이 많은 폴더는 1시간 간격, 사진·미디어처럼 변경이 적은 폴더는 6시간 간격으로 설정했다. 보존은 24시간 단위 7개, 주 단위 4개 정도면 시작하기 쉽다.

DSM 7.2 이상이고 지원되는 Btrfs 환경이라면 `Immutable Snapshot`을 켤 수 있다. 이 기능은 지정한 기간 동안 스냅샷을 수정하거나 삭제하지 못하게 하는 WORM(Write Once, Read Many) 방식이다. 관리자 계정이 탈취돼도 보호 기간 안의 복원 지점은 지울 수 없다는 게 핵심이다.

단, 스냅샷은 같은 볼륨에 저장된다. NAS 전체가 고장 나거나 도난당하면 같이 사라지므로 이 단계만으로 백업이라고 부르면 안 된다.

## 2. USB로 버전 백업 만들기

외장 디스크를 NAS에 연결하고 `Hyper Backup → 새 데이터 백업 작업 → 로컬 폴더 및 USB`를 선택한다. `Documents`, `Photos`, Docker 서비스의 설정 폴더처럼 다시 만들기 어려운 데이터만 우선 선택한다. 영화처럼 다시 구할 수 있는 파일까지 모두 넣으면 USB 용량과 백업 시간이 불필요하게 커진다.

권장값은 아래처럼 잡았다.

| 항목 | 설정값 |
|---|---|
| 백업 방식 | 다중 버전 백업 |
| 실행 시간 | 매일 03:00 |
| 보존 | Smart Recycle 또는 일 7개·주 4개·월 6개 |
| 암호화 | 클라이언트 측 암호화 켜기 |
| 무결성 | 주 1회 데이터 무결성 검사 |

암호화 비밀번호는 NAS 관리자 비밀번호와 다르게 만들고, 비밀번호 파일을 백업 폴더 안에만 저장하지 않는다. 키를 잃으면 복원도 못 한다. 백업이 끝난 USB는 계속 꽂아 두지 말고, 작업 완료 알림을 확인한 뒤 분리해 보관하는 편이 안전하다.

## 3. 다른 장소에 오프사이트 복사하기

두 번째 NAS가 있으면 `원격 NAS 장치`를, 장비가 없으면 C2 Storage나 S3 호환 저장소를 Hyper Backup 대상에 추가한다. 이 대상은 집 네트워크와 다른 계정·다른 장소에 있어야 한다. 같은 공유기 안의 다른 폴더는 오프사이트 백업이 아니다.

원격 NAS를 쓸 때는 백업 전용 계정을 만들고 공유 폴더 쓰기 권한만 준다. DSM 관리 권한, SSH 권한, 모든 공유 폴더 권한을 함께 주면 원본 계정이 뚫렸을 때 백업본까지 지워질 수 있다. 가능하면 백업 NAS의 Hyper Backup Vault만 외부에 노출하고 DSM 관리 포트는 인터넷에 열지 않는다.

## 복구 테스트를 빼먹으면 안 되는 이유

백업 작업이 성공했다는 알림은 파일을 실제로 복원할 수 있다는 뜻이 아니다. 한 달에 한 번 테스트 폴더를 만들어 `Hyper Backup Explorer`로 파일 3개와 Docker Compose 파일 1개를 복원한다. 스냅샷에서는 랜섬웨어 발생 전 시점의 파일을 별도 폴더로 복구하고, USB 백업에서는 다른 PC에서도 읽히는지 확인한다.

점검 체크리스트는 다음과 같다.

- [ ] 백업 작업 로그에 실패가 없는가
- [ ] 복원 암호와 키를 실제로 찾을 수 있는가
- [ ] NAS와 분리된 USB 백업이 하나 이상 있는가
- [ ] 오프사이트 백업의 최근 성공 시각이 24시간 이내인가
- [ ] 테스트 복원 파일을 열어봤는가
- [ ] 관리자 계정에 2단계 인증과 로그인 알림을 켰는가

내가 처음 구성할 때 놓친 부분은 스냅샷 보존 기간만 길게 잡고 USB를 계속 연결해 둔 것이었다. 그 상태에서는 랜섬웨어가 NAS와 연결된 USB 백업까지 동시에 건드릴 수 있다. 스냅샷은 빠른 복구, USB는 분리 보관, 오프사이트는 재해 복구로 역할을 나누면 비용과 복구 속도 사이의 균형을 잡기 쉽다.

기준을 짧게 정리하면 `Snapshot Replication으로 즉시 복원`, `Hyper Backup으로 버전 보관`, `USB 분리와 오프사이트 저장으로 최악의 상황 대비`다. 백업 개수보다 중요한 건 실제로 복원되는지와 원본과 다른 장소에 복사본이 남아 있는지다.

참고 문서:

- [Synology Hyper Backup 기술 사양](https://www.synology.com/en-eu/dsm/7.4/software_spec/hyper_backup)
- [Synology Snapshot Replication](https://www.synology.com/en-us/dsm/feature/snapshot_replication)
- [Synology Immutable Snapshot 안내](https://kb.synology.com/en-sg/DSM/tutorial/what_is_an_immutable_snapshot)
- [Synology 3-2-1 백업 가이드](https://kb.synology.com/en-global/DSM/help/DSM/Tutorial/backup_backup)

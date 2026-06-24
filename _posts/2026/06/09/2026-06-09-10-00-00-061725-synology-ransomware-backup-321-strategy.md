---
layout: post
title: "Synology NAS 랜섬웨어 대비 백업 전략 — 3-2-1 규칙 실제 구현"
description: "Synology NAS에서 랜섬웨어와 하드 드라이브 고장을 동시에 대비하는 3-2-1 백업 규칙을 DSM 7.2 Hyper Backup과 외부 클라우드로 실제 구현하는 방법."
date: 2026-06-09
tags: [Synology, 백업전략, NAS보안, DSM]
comments: true
share: true
---

# Synology NAS 랜섬웨어 대비 백업 전략 — 3-2-1 규칙 실제 구현

![NAS 백업 전략 데이터 보안](https://images.unsplash.com/photo-1558618047-3c8c76ca7d67?w=800&q=80)

NAS에 데이터를 몰아 넣으면서 백업을 안 하는 사람이 생각보다 많다. RAID는 백업이 아니다. RAID는 하드 고장에 대비할 뿐이고, 랜섬웨어나 실수로 삭제한 파일은 RAID도 못 막는다.

이 글은 Synology 완전 정복 시리즈 마지막 편이다. [이전 편에서 외부 접속 설정]({% post_url 2026-06-08-10-00-00-049380-synology-ddns-letsencrypt-external-access %})을 했다면, 이제 데이터를 지키는 방법을 다룬다.

## 3-2-1 백업 규칙이란

- **3**: 데이터 복사본을 3개 만든다 (원본 1 + 백업 2)
- **2**: 서로 다른 종류의 미디어에 저장 (예: NAS + 외장 HDD)
- **1**: 1개는 오프사이트(다른 물리적 위치)에 보관

랜섬웨어는 네트워크에 연결된 모든 드라이브를 암호화한다. 같은 NAS에만 백업해도 함께 날아간다. 오프사이트 백업이 반드시 필요한 이유다.

## 환경 정보

| 항목 | 버전/모델 |
|------|----------|
| NAS 모델 | Synology DS923+ |
| DSM 버전 | DSM 7.2.2 |
| Hyper Backup 버전 | 4.1.0-3028 |
| 외부 클라우드 | Backblaze B2 |
| 외장 HDD | WD My Passport 4TB |

## 레이어 1: NAS 내부 스냅샷

Synology의 **Snapshot Replication** 앱을 사용하면 Btrfs 파일 시스템에서 스냅샷을 주기적으로 생성한다. 파일을 실수로 지우거나 덮어썼을 때 빠르게 복구할 수 있다.

```
패키지 센터 → Snapshot Replication 설치
```

공유 폴더별로 스냅샷 일정을 설정할 수 있다. 아래 설정을 권장한다:

```
매시간 스냅샷: 최근 24시간 보관
매일 스냅샷: 최근 30일 보관
매주 스냅샷: 최근 12주 보관
```

단, 스냅샷은 NAS 내부에 있어서 랜섬웨어가 NAS를 장악하면 스냅샷도 위험하다. 1차 방어선으로는 유용하지만 이게 전부여서는 안 된다.

## 레이어 2: 외장 HDD 백업

USB로 연결한 외장 HDD에 **Hyper Backup**으로 정기 백업을 만든다.

```bash
패키지 센터 → Hyper Backup 설치
```

백업 작업 설정:
```
백업 대상: 외부 장치 (USB HDD)
백업 소스: 백업할 공유 폴더 선택
백업 일정: 매일 새벽 3시
보관 정책: 최근 30일
```

![Hyper Backup 설정 화면](https://images.unsplash.com/photo-1544197150-880d4212a0c4?w=800&q=80)

Hyper Backup은 증분 백업을 지원해서 첫 번째 백업 이후에는 변경된 파일만 저장한다. 4TB HDD에 수십 버전을 보관할 수 있다.

**중요**: 백업 완료 후 HDD를 NAS에서 분리해두는 습관이 필요하다. 항상 연결돼 있으면 랜섬웨어가 이 HDD도 암호화한다.

## 레이어 3: 오프사이트 클라우드 백업

Backblaze B2는 1TB당 월 $6로 S3보다 저렴하다. Hyper Backup에서 B2를 직접 지원한다.

Backblaze B2 설정:
1. [Backblaze B2](https://www.backblaze.com/b2/cloud-storage.html) 회원가입
2. 버킷 생성 (버킷명: `myhome-nas-backup`)
3. 애플리케이션 키 생성 (keyID, applicationKey 기록)

Hyper Backup에서 새 백업 작업 추가:
```
백업 대상: S3 호환 스토리지
서버: s3.us-west-004.backblazeb2.com
버킷: myhome-nas-backup
디렉터리: /nas-backup
키 ID: (B2에서 생성한 keyID)
애플리케이션 키: (B2에서 생성한 applicationKey)
```

일정은 매주 1회로 설정하고, 보관 기간은 3개월 정도가 현실적이다. 데이터가 많으면 클라우드 비용이 올라가니 중요한 폴더만 선별해서 올리는 게 낫다.

## 랜섬웨어 추가 방어

**쓰기 보호 공유 폴더**: 백업 전용 폴더는 네트워크 공유에서 제외하거나 읽기 전용으로 설정해서 랜섬웨어가 접근하지 못하게 한다.

**Synology Active Backup**: 사무용이지만 Synology에서 무료로 제공하는 백업 솔루션. PC, 서버, 가상머신 백업에 특화됐다.

**DSM 알림 설정**: 백업 실패 시 이메일이나 푸시 알림을 받도록 설정해두면 백업이 조용히 실패하는 상황을 막을 수 있다.

```
제어판 → 알림 → 이메일
→ SMTP 설정 후 "백업 작업 실패" 알림 활성화
```

## 삽질했던 부분

**Hyper Backup 버전 관리 공간 계산 실수**: 처음에 2TB HDD에 백업하면서 버전을 30개 보관하도록 설정했는데, 2TB 이상 쌓이면서 오래된 버전이 자동 삭제되는 줄 몰랐다. 처음부터 충분한 용량의 HDD를 쓰거나 보관 버전 수를 줄여야 한다.

**클라우드 업로드 속도**: 처음 전체 백업(약 1.5TB)은 업로드 속도에 따라 며칠 걸릴 수 있다. 가정용 인터넷은 업로드가 느리니 첫 백업은 기다려야 한다. DSM에서 백업 일정과 대역폭 제한을 설정해 낮 시간대 인터넷 속도에 영향을 안 주도록 하는 게 좋다.

**외장 HDD 포맷**: Hyper Backup은 HDD를 Synology 전용 포맷(하이퍼백업 형식)으로 저장한다. Windows에서는 직접 파일을 읽을 수 없고, Hyper Backup Explorer 앱을 써야 한다.

## 한 줄 정리

스냅샷으로 실수 복구, 외장 HDD로 로컬 백업, 클라우드로 오프사이트 백업 — 세 레이어를 다 갖추면 랜섬웨어도 하드 고장도 무섭지 않다.

---
다음엔 [Home Assistant 설치 방법 비교 2026 — Raspberry Pi vs Synology NAS vs VM]({% post_url 2026-06-10-10-00-00-074070-home-assistant-install-rpi-nas-vm-compare %})을 다룬다.

**참고 링크**
- [Synology Hyper Backup 공식 문서](https://kb.synology.com/ko-kr/DSM/help/HyperBackup/data_backup_and_restoration)
- [Backblaze B2 공식 사이트](https://www.backblaze.com/b2/cloud-storage.html)
- [3-2-1 백업 규칙 설명](https://www.backblaze.com/blog/the-3-2-1-backup-strategy/)

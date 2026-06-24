---
layout: post
title: "QNAP NAS 백업 전략 — 3-2-1 규칙으로 데이터 손실 없이 관리하는 법"
description: "QNAP NAS에서 HBS 3 Hybrid Backup Sync를 활용해 3-2-1 백업 규칙을 실제로 구현하는 방법. Synology와 다른 QNAP만의 특이사항과 Rclone 연동까지."
date: 2026-06-21
tags: [QNAP, 백업전략, NAS설정]
comments: true
share: true
---

# QNAP NAS 백업 전략 — 3-2-1 규칙으로 데이터 손실 없이 관리하는 법

![QNAP NAS 백업 전략 데이터 관리](https://images.unsplash.com/photo-1603732351-97311a4c36c6?w=800&q=80)

QNAP을 쓰는 사람들은 대체로 Synology보다 하드웨어에서 한 수 위를 노리는 경우가 많다. 성능은 좋은데 백업 설정이 Synology보다 조금 복잡하다. HBS 3(Hybrid Backup Sync)라는 앱으로 백업을 관리하는데, 처음 보면 옵션이 너무 많아서 어디서 시작해야 할지 헷갈린다.

앞에서 [Synology 3-2-1 백업 전략]({% post_url 2026-06-09-10-00-00-061725-synology-ransomware-backup-321-strategy %})을 다뤘는데, QNAP도 기본 개념은 같고 도구가 다르다.

## 환경 정보

| 항목 | 버전/모델 |
|------|----------|
| NAS 모델 | QNAP TS-464 |
| QTS 버전 | QTS 5.1.8 |
| HBS 3 버전 | 25.x |
| 외장 HDD | Seagate Backup Plus 8TB |

## QNAP 3-2-1 백업 구성

| 복사본 | 위치 | 방법 |
|-------|------|------|
| 원본 | QNAP 내부 | 스냅샷 (Snapshots) |
| 로컬 백업 | USB 외장 HDD | HBS 3 백업 작업 |
| 오프사이트 | Backblaze B2 / S3 | HBS 3 클라우드 백업 |

## 스냅샷 설정

QNAP Storage & Snapshots 앱에서 스냅샷을 설정한다. Btrfs 또는 특정 볼륨에서만 지원된다.

```
Storage & Snapshots → 볼륨 선택 → 스냅샷 일정
```

권장 설정:
```
매시간: 최근 24시간
매일: 최근 14일
매주: 최근 4주
```

단, 스냅샷이 활성화돼 있으면 디스크 공간을 더 사용한다. 남은 공간의 20% 정도를 스냅샷이 쓴다고 생각하면 된다.

## HBS 3로 외장 HDD 백업

**HBS 3 설치**: App Center에서 HBS 3 Hybrid Backup Sync 설치.

```
HBS 3 → 백업 → 백업 작업 생성
원본: 백업할 공유 폴더 선택
대상: 외장 USB 장치
```

옵션에서 **증분 백업**을 선택한다. 처음만 전체 백업하고 이후에는 변경된 파일만 동기화한다.

**버전 관리**: 삭제된 파일도 일정 기간 보관되도록 설정한다.

```
버전 보관: 30개 버전 또는 최근 90일
```

```bash
# QNAP 스케줄 예시
매일 새벽 2시 자동 실행
완료 후 이메일 알림 발송
```

## HBS 3로 Backblaze B2 클라우드 백업

Backblaze B2 계정 생성 후 버킷과 키를 만든다. ([Synology 글의 B2 설정 방법]({% post_url 2026-06-09-10-00-00-061725-synology-ransomware-backup-321-strategy %}) 참조)

HBS 3에서 새 백업 작업 생성:

```
대상 유형: Backblaze B2
버킷명: qnap-nas-backup
키 ID: (B2 키 ID)
애플리케이션 키: (B2 애플리케이션 키)
폴더: /nas-backup
```

암호화 옵션을 반드시 활성화한다. AES-256으로 암호화하면 B2에 올라간 데이터를 Backblaze도 읽을 수 없다. 단, 암호화 비밀번호를 잊으면 복구가 불가능하므로 따로 안전하게 보관해야 한다.

## Rclone 활용 (고급)

HBS 3의 클라우드 지원 범위를 벗어나는 저장소(예: OneDrive, 특정 S3 호환 등)를 쓰려면 QNAP에서 Rclone을 직접 쓸 수 있다.

```bash
# QNAP SSH 접속 후
# Entware 설치 후 rclone 설치
opkg install rclone

# rclone 설정
rclone config

# 백업 스크립트 예시
rclone sync /share/homes/admin/ remote:backup/homes \
  --progress \
  --log-file=/share/logs/rclone-$(date +%Y%m%d).log
```

이 스크립트를 QNAP Task Scheduler에 등록하면 정기 자동 실행된다.

![QNAP HBS 3 백업 대시보드](https://images.unsplash.com/photo-1544197150-880d4212a0c4?w=800&q=80)

## QNAP 랜섬웨어 추가 방어

**Security Counselor**: QNAP 전용 보안 앱. 취약점 스캔과 권장 설정을 알려준다. 설치 후 실행하면 점검 항목과 위험도가 표시된다.

**SSH 포트 변경**: 기본 포트 22는 공격 타겟. `/etc/ssh/sshd_config`에서 포트 변경.

**myQNAPcloud 연결 주의**: 외부 접속 편의를 위해 myQNAPcloud에 연결하면 QNAP 서버를 경유하는데, QNAP도 공격 대상이 된 전례가 있다. 외부 접속은 Tailscale 같은 VPN이 더 안전하다.

## 삽질했던 부분

**USB 외장 HDD 인식 안 됨**: QNAP QTS 5.1에서 exFAT 포맷 외장 HDD가 자동으로 인식이 안 되는 경우가 있다. Storage & Snapshots에서 외장 기기를 수동으로 초기화하면 인식됐다. 외장 HDD의 기존 데이터를 포맷하게 되니 주의.

**HBS 3 암호화 비밀번호**: 나중에 복구할 때 이 비밀번호가 없으면 파일을 못 꺼낸다. 안전한 곳(Vaultwarden)에 반드시 저장해둔다.

**QNAP 자동 업데이트 주의**: QTS는 주요 업데이트 전에 반드시 릴리즈 노트를 확인한다. QTS 5.x 초기에 일부 업데이트 이후 Container Station(Docker)이 초기화되는 버그가 있었다.

## 한 줄 정리

QNAP도 스냅샷 + HBS 3 외장 HDD 백업 + 클라우드 오프사이트 백업 세 레이어를 갖추면 충분하다. Security Counselor로 정기적으로 보안 점검하는 것도 빠뜨리지 않는다.

---
다음엔 [TrueNAS SCALE 설치부터 SMB 공유까지 — ZFS 기반 무료 NAS OS 세팅]({% post_url 2026-06-22-10-00-00-222210-truenas-scale-install-smb-share %})을 다룬다.

**참고 링크**
- [QNAP HBS 3 공식 문서](https://www.qnap.com/en/how-to/tutorial/article/hybrid-backup-sync-hbs-3)
- [QNAP Security Counselor 소개](https://www.qnap.com/en/software/security-counselor)
- [Rclone 공식 사이트](https://rclone.org)

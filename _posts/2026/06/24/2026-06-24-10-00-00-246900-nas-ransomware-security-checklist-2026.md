---
layout: post
title: "NAS 랜섬웨어 대비 보안 설정 체크리스트 2026 — Synology 기준"
description: "2026년 기준 Synology NAS 랜섬웨어 방어를 위한 보안 설정 체크리스트. 계정 보안, 네트워크 보안, 백업 전략, DSM 설정까지 실제로 적용한 항목만 담았다."
date: 2026-06-24
tags: [NAS보안, Synology, 백업전략, DSM]
comments: true
share: true
---
![NAS 랜섬웨어 보안 설정 체크리스트](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=800&q=80)

NAS 랜섬웨어 피해는 꾸준히 있다. 2021년 QNAP, 2022년 Synology, 2024년에도 사례가 나왔다. 대부분 기본 보안 설정을 안 했거나 취약한 비밀번호, 오래된 펌웨어가 원인이었다. 이 체크리스트는 이 시리즈에서 다뤘던 내용을 항목별로 정리한 거다.

## 계정 보안

### admin 계정 비활성화 ✓
기본 admin 계정은 공격자가 제일 먼저 시도하는 타겟이다. 새 관리자 계정을 만들고 admin은 비활성화해야 한다.

```
제어판 → 사용자 및 그룹 → admin → 계정 비활성화
```

### 강력한 비밀번호 정책 ✓

```
제어판 → 보안 → 계정
→ 비밀번호 강도 규칙 활성화
→ 최소 8자, 대/소문자 + 숫자 + 특수문자 포함
```

### 2단계 인증 (2FA) ✓
모든 계정에 Google Authenticator나 Authy로 TOTP 2FA를 활성화한다.

```
제어판 → 보안 → 계정 → 2단계 인증 강제 적용
```

### 자동 차단 (Brute Force 방어) ✓

```
제어판 → 보안 → 보호
→ 자동 차단 활성화
→ 5분 내 5회 실패 시 차단
```

## 네트워크 보안

### 기본 포트 변경 ✓

| 서비스 | 기본 포트 | 변경 권장 |
|-------|---------|---------|
| HTTP | 5000 | 49000대 |
| HTTPS | 5001 | 49001대 |
| SSH | 22 | 2만번대 이상 |

```
제어판 → 로그인 포털 → DSM
```

### 방화벽 활성화 ✓

```
제어판 → 보안 → 방화벽
→ 필요한 포트만 명시적으로 허용
→ 맨 아래에 "모든 거부" 규칙 추가
```

### 국가 기반 IP 차단 ✓
한국에서만 쓴다면 한국 IP 외 모두 차단.

```
제어판 → 보안 → 방화벽 → 규칙 편집
→ 위치 기반 규칙 → 특정 국가만 허용
```

### SSH 사용 안 한다면 끄기 ✓
```
제어판 → 터미널 및 SNMP → SSH 서비스 비활성화
```

### 외부 접속은 VPN으로 (추천)

포트 포워딩 최소화. [Tailscale VPN]({% post_url 2026-06-18-10-00-00-172830-tailscale-vpn-home-server-remote-access %})으로 외부 접속하면 인터넷에 노출되는 포트를 줄일 수 있다.

## DSM 및 시스템 설정

### DSM 자동 업데이트 활성화 ✓

```
제어판 → 업데이트 및 복구
→ 중요 보안 업데이트 자동 설치
```

메이저 버전 업그레이드는 수동으로. 마이너 보안 패치는 자동이 낫다.

### 불필요한 서비스 비활성화 ✓
QuickConnect, UPnP, Telnet 등 쓰지 않는 서비스는 끈다.

```
제어판 → 외부 액세스 → QuickConnect → 비활성화
제어판 → 네트워크 → 네트워크 인터페이스 → DSM UPnP → 비활성화
```

### SMB 버전 관리 ✓
오래된 SMBv1은 보안 취약점이 많다. SMBv2 이상만 허용.

```
제어판 → 파일 서비스 → SMB → 고급 → 최소 SMB 프로토콜: SMB2
```

![Synology DSM 보안 설정 체크리스트](https://images.unsplash.com/photo-1544197150-880d4212a0c4?w=800&q=80)

## 백업 전략

### 3-2-1 백업 구성 ✓

[3-2-1 백업 규칙]({% post_url 2026-06-09-10-00-00-061725-synology-ransomware-backup-321-strategy %}) 실제 적용:

| 복사본 | 위치 | 방법 |
|-------|------|------|
| 1 (원본) | NAS 내부 | Snapshot Replication |
| 2 (로컬) | USB 외장 HDD | Hyper Backup |
| 3 (오프사이트) | Backblaze B2 | Hyper Backup |

### 백업 분리 ✓
백업용 외장 HDD는 백업 완료 후 NAS에서 분리. 항상 연결해두면 랜섬웨어에 같이 당한다.

### 백업 복구 테스트 ✓
백업이 있어도 복구가 안 되면 의미 없다. 분기마다 한 번씩 파일 복구 테스트를 실제로 해본다.

### 스냅샷 변조 방지 ✓
Synology의 **Snapshot Replication**에서 스냅샷 잠금 기능을 활성화하면 스냅샷을 삭제하거나 변조하는 것을 막을 수 있다.

```
Snapshot Replication → 공유 폴더 → 스냅샷 설정 → 스냅샷 잠금 활성화
```

## 모니터링

### 로그 알림 설정 ✓

```
제어판 → 알림 → 규칙
→ 로그인 실패 알림
→ 백업 실패 알림
→ 디스크 오류 알림
```

### Uptime Kuma로 서비스 모니터링 ✓
[Uptime Kuma]({% post_url 2026-06-17-10-00-00-160485-uptime-kuma-service-monitoring %}) 설치해서 NAS 자체와 운영 중인 서비스들의 가동 상태를 24시간 모니터링.

## 자주 빠뜨리는 항목

**공유 폴더 권한 검토**: Everyone 권한이 열려있는 공유 폴더가 있으면 랜섬웨어가 쓰기 접근을 할 수 있다. 필요한 사용자에게만 최소 권한을 부여한다.

**Docker 컨테이너 보안**: 컨테이너를 `privileged` 모드로 실행하면 컨테이너가 뚫렸을 때 NAS 전체가 위험하다. 꼭 필요한 경우만 privileged 쓰고, 가능하면 특정 capability만 추가한다.

**패키지 출처 확인**: DSM 패키지 센터에서 비공식 패키지를 설치할 때 출처를 확인한다. 악성코드가 숨겨진 패키지가 간혹 발견된다.

## 보안 수준별 정리

| 수준 | 설정 항목 |
|-----|---------|
| 필수 | admin 비활성화, 2FA, 자동 차단, DSM 업데이트, 3-2-1 백업 |
| 권장 | 포트 변경, 방화벽, SMBv1 비활성화, 스냅샷 잠금 |
| 고급 | 국가 IP 차단, VPN 접속, 공유 폴더 권한 최소화 |

## 한 줄 정리

필수 항목 5가지(admin 비활성화, 2FA, 자동 차단, DSM 업데이트, 3-2-1 백업)만 해도 일반적인 랜섬웨어 공격의 90%는 막힌다. 이 체크리스트를 월 1회 점검하는 습관이 중요하다.

**참고 링크**
- [Synology 보안 권장 사항](https://kb.synology.com/ko-kr/DSM/tutorial/How_to_add_extra_security_to_your_Synology_NAS)
- [Synology Security Advisories](https://www.synology.com/ko-kr/security/advisory)
- [Backblaze B2 클라우드 스토리지](https://www.backblaze.com/b2/cloud-storage.html)

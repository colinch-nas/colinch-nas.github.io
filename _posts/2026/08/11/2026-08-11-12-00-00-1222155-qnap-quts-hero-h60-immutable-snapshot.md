---
layout: post
title: "QNAP QuTS hero h6.0 불변 스냅샷 설정 - 랜섬웨어 복구 테스트까지"
description: "QNAP QuTS hero h6.0 정식 버전에서 불변 스냅샷을 설정하고 보존 기간, 권한, 복구 테스트까지 NAS에서 직접 점검하는 방법을 정리한다."
date: 2026-08-11
tags: [QNAP, QuTSHero, NAS보안, 백업전략, 랜섬웨어]
comments: true
share: true
---

![QNAP QuTS hero h6.0 불변 스냅샷과 NAS 보안](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

QNAP이 2026년 5월 29일 QuTS hero h6.0 정식 버전에서 Immutable Snapshot(정해진 기간 동안 삭제·변경할 수 없는 스냅샷)을 모든 QuTS hero 모델에 넣었다. RAID만 믿고 있었다면 켜볼 만하다. 단, 스냅샷은 별도 백업이 아니다.

## 환경과 설정

QNAP TS-h 계열, QuTS hero h6.0.0, HDD 4개 RAIDZ1, 공유 폴더 `family` 기준이다.

| 항목 | 값 |
|---|---:|
| 스냅샷 주기 | 1시간 |
| 불변 보호 기간 | 14일 |
| 외부 복제 | 주 1회 |

`Storage & Snapshots → Snapshot → Snapshot Manager`에서 `Create Snapshot`을 누른다. 데이터셋을 선택하고 `Schedule`을 1시간으로 지정한 뒤 `Immutable Snapshot`을 켠다. 보호 기간은 14일로 설정했다.

보호 기간과 보존 개수는 별개다. ZFS 풀 여유 공간은 20~30% 남겨둔다.

## 복구 테스트

테스트 파일을 만들고 스냅샷을 생성한 뒤 삭제한다.

```bash
mkdir -p /share/family/_snapshot-test
printf 'snapshot-test-2026-08-11\n' > /share/family/_snapshot-test/restore.txt
sha256sum /share/family/_snapshot-test/restore.txt
```

`Snapshot Manager → Browse`에서 삭제된 파일을 `restore-check` 폴더로 복사한다. 보호 기간 안에 삭제 요청이 거부되고 관리자 계정으로도 지워지지 않아야 정상이다.

복원한 파일의 SHA-256 값이 같으면 복구는 통과다.

## 외부 백업과 보안

스냅샷은 같은 풀에 있으므로 디스크 고장과 NAS 도난을 막지 못한다. `HBS 3`에서 `family`를 USB 디스크나 다른 NAS로 주 1회 복제한다. 관리자 화면은 포트포워딩 대신 Tailscale 또는 QVPN으로 접속하고 MFA, 별도 관리자 계정, QuFirewall도 확인한다.

정리하면 중요한 데이터셋에 1시간 주기와 14일 불변 보호를 적용한 뒤, 복구 테스트와 HBS 3 외부 복제까지 확인해야 한다.

참고: [QNAP QuTS hero h6.0 정식 출시 안내](https://www.qnap.com/en/news/2026/qnap-officially-releases-quts-hero-h6-0-official-featuring-dual-nas-high-availability-immutable-snapshots-and-more)

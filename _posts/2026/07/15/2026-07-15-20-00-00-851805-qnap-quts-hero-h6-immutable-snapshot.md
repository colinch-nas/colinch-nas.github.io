---
layout: post
title: "QNAP QuTS hero h6.0 불변 스냅샷 설정 - 랜섬웨어 복구 지점 만들기"
description: "2026년 5월 정식 출시된 QNAP QuTS hero h6.0에서 Immutable Snapshot을 설정하고, 공유 폴더 파일을 실제로 복구하는 순서와 주의사항을 정리한다."
date: 2026-07-15
tags: [QNAP, NAS보안, 백업전략, 홈서버, 자체호스팅]
comments: true
share: true
---

![QNAP QuTS hero h6.0 불변 스냅샷과 홈서버 보안](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

QNAP이 2026년 5월 29일 QuTS hero h6.0 정식 버전을 공개했다. 이번 버전에서 홈 NAS 사용자가 바로 써볼 만한 기능은 **Immutable Snapshot(변경·삭제 잠금 스냅샷)**이다. 파일이 암호화되거나 실수로 지워져도 정해진 기간 동안 복구 지점을 지울 수 없게 만든다. 단, 스냅샷은 같은 NAS 안에 있으므로 외장 디스크나 다른 NAS 백업을 대체하지는 않는다.

내 기준 환경은 QNAP TS-h 계열, QuTS hero h6.0, 4TB 디스크 4개 RAID-Z1, 공유 폴더 `documents`다. QTS가 아니라 QuTS hero에서만 가능한 기능이므로, TS-464 같은 QTS 모델에 같은 메뉴가 안 보인다고 오류로 생각하면 안 된다.

## 업데이트 전에 확인할 것

| 항목 | 확인 기준 |
|---|---|
| 펌웨어 | QuTS hero h6.0 정식 및 모델별 지원 여부 |
| 백업 | HBS 3 또는 외장 디스크에 최근 복구본 존재 |
| 여유 공간 | 스냅샷 예약 공간을 확보할 수 있음 |
| 앱 | Container Station, Jellyfin 작업이 중단돼도 되는 시간 |

QuTS hero 업데이트는 먼저 **Control Panel → System → Firmware Update**에서 모델별 릴리스 노트를 확인한다. 업데이트 직전에는 컨테이너를 멈추고 HBS 3 작업이 끝났는지 본다. 스냅샷이 생긴다고 백업이 끝난 건 아니다. NAS 자체가 고장 나면 스냅샷도 같이 사라진다.

## 불변 스냅샷 설정

먼저 스냅샷이 저장될 공간을 예약한다. 공식 문서의 권장값은 스토리지 풀의 20%다.

1. **Snapshot Manager → Snapshot**으로 들어간다.
2. `documents` 공유 폴더의 작업 메뉴에서 **Pool Guaranteed Snapshot Space**를 연다.
3. `Enable`을 선택하고 10~20%를 지정한다.
4. 공유 폴더의 **Snapshot → Schedule Snapshot**에서 1시간 간격, 24개 보관을 만든다.
5. 보호 정책에서 **Prohibit recycle and delete until expired**를 선택하고 7일을 지정한다.

처음부터 7일을 잠그기보다 테스트용 폴더로 24시간 정책을 먼저 적용하는 게 낫다. 불변 기간 안에는 관리자 계정도 스냅샷을 지울 수 없다. 이게 기능의 핵심이면서, 잘못 설정했을 때 공간을 빨리 소모하는 이유다.

```text
대상: documents
주기: 매시간
보관: 24개
삭제 정책: 만료 전 자동 삭제 금지 + 수동 삭제 금지
보호 기간: 7일
```

## 복구 테스트까지 해야 한다

테스트용 파일 `restore-test.txt`를 만들고 스냅샷을 수동 생성한 뒤 파일 내용을 바꾼다. File Station의 `@Recently-Snapshot` 폴더가 보이지 않으면 **Snapshot Manager → Global Settings**에서 snapshot directory 표시를 켠다. 해당 시점의 파일을 별도 `restore-check` 폴더로 복사하고 내용이 원래대로 돌아왔는지 확인한다.

여기서 한 번 삽질했다. 스냅샷을 만들었으니 원본 파일을 바로 되돌려 줄 거라 생각했는데, 실제 복구는 스냅샷 안의 파일을 다른 위치로 복사하는 작업이었다. 대량 복구 전에는 파일명, 권한, Docker가 사용하는 공유 폴더인지까지 확인해야 한다.

## 주의사항

- 스냅샷은 백업이 아니다. 다른 NAS, 외장 디스크, 오프사이트 저장소를 함께 둔다.
- 스토리지 풀이 가득 차면 새 스냅샷이 실패할 수 있다. 예약 공간과 풀 사용률을 같이 모니터링한다.
- QuTS hero h6.0의 불변 스냅샷은 High Availability 클러스터에서는 지원되지 않는다.
- 관리자 계정이 탈취되면 NAS 전체가 위험하다. FIDO2 패스키, 2단계 인증, 외부 관리 포트 차단을 함께 적용한다.

QNAP QuTS hero h6.0의 Immutable Snapshot은 랜섬웨어에 당한 뒤 복구할 마지막 안전판으로 쓸 만하다. 시간별 스냅샷과 7일 잠금부터 시작하고, 실제 파일 복구가 끝난 뒤에 보호 기간을 늘리는 방식이 홈서버에서는 관리하기 쉽다.

출처: [QNAP QuTS hero h6.0 정식 발표](https://www.qnap.com/en/news/2026/qnap-officially-releases-quts-hero-h6-0-official-featuring-dual-nas-high-availability-immutable-snapshots-and-more), [QuTS hero h6.0 릴리스 노트](https://www.qnap.com/en/release-notes/quts_hero/overview/h6.0.0), [QNAP 스냅샷 보호 정책 문서](https://docs.qnap.com/operating-system/quts-hero/5.3.x/en-us/editing-a-snapshot-91DAC816.html)

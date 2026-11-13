---
layout: post
title: "NAS HDD 4Kn·512e 호환성 - DSM 7.4·QTS 5.2·TrueNAS SCALE 25.10 교체 비용"
description: "NAS용 HDD를 교체할 때 4Kn과 512e 중 무엇을 고를지 Synology DSM 7.4·QNAP QTS 5.2·TrueNAS SCALE 25.10의 호환 목록, RAID 복구와 총비용으로 판단한다."
date: 2026-11-14
tags: [NAS, Synology, QNAP, TrueNAS, 저장장치, 호환성, 백업]
comments: true
share: true
---

![NAS용 HDD의 4Kn·512e 섹터 형식과 RAID 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)
이 그림에서 볼 부분은 같은 12TB라도 모델 번호와 논리 섹터 형식이 다르면 RAID 교체 디스크가 되지 않을 수 있다는 점이다.

NAS의 고장 디스크를 바꾸는 상황이라면 성능보다 **기존 풀의 섹터 형식과 제조사 호환 목록**을 맞춰야 한다. 1~2인 사진 보관용이면 512e 호환 HDD를 그대로 고르는 편이 안전하고, 새 TrueNAS 풀을 처음 만드는 경우에만 4Kn을 검토한다. 4Kn이 더 빠르다는 이유로 기존 Synology·QNAP 풀에 섞어 사는 선택은 피해야 한다. 아래 내용은 2026년 11월 14일 공식 문서를 확인한 기준이며, 특정 장비를 직접 시험한 결과는 아니다.

## 4Kn과 512e는 무엇이 다른가

4Kn은 논리·물리 섹터가 모두 4,096바이트다. 512e는 물리 섹터는 4,096바이트지만 운영체제에는 512바이트처럼 보인다. QNAP 공식 설명처럼 NAS의 캐시·RAID 계층이 정렬을 처리하므로 4Kn으로 바꾼다고 성능이 자동으로 좋아지지 않는다.

| 상황 | 우선 확인할 것 | 구매 판단 |
|---|---|---|
| Synology DSM 7.4 기존 풀 교체 | 모델·용량·4Kn 여부가 호환 목록에 있는가 | 기존 디스크와 같은 형식 우선 |
| QNAP QTS 5.2 RAID 교체 | QNAP 호환 목록과 RAID 재빌드 시험 여부 | 4Kn 전환보다 검증된 모델 우선 |
| TrueNAS SCALE 25.10 새 ZFS 풀 | `ashift`와 디스크가 보고하는 섹터 크기 | 풀 생성 전에 형식을 고정 |
| 외장 단일 백업 디스크 | 복구할 PC·NAS가 읽을 수 있는가 | 속도보다 범용성과 교체 가능성 |

Synology와 QNAP은 같은 제품군명이라도 정확한 모델·펌웨어·용량별 호환 결과가 달라진다. QNAP은 호환 목록에서 디스크를 NAS 모델로 선택하라고 안내하고, Synology도 호환 목록에 4K 네이티브 여부를 별도 항목으로 둔다. “인식된다”는 말과 RAID에 공식 지원된다는 말은 구분해야 한다.

## 교체 전에 기록할 값

디스크를 빼기 전에 아래 양식을 저장한다. 화면에 섹터 정보가 보이지 않으면 제조사 데이터시트나 `smartctl -i /dev/sdX` 결과를 함께 보관한다.

```text
NAS 모델 / OS 버전:
풀·RAID·파일시스템:
기존 디스크 모델 / 용량:
논리 섹터: 512n / 512e / 4Kn
새 디스크 정확한 모델:
호환 목록 URL과 확인 날짜:
백업 위치와 마지막 복원 확인일:
```

TrueNAS에서는 새 풀을 만들 때 ZFS의 섹터 정렬을 결정한다. 기존 풀의 `ashift`를 나중에 바꾸는 작업으로 생각하면 안 된다. 512e와 4Kn을 섞어도 된다는 커뮤니티 사례를 근거로 기존 풀을 구성하지 말고, 새 풀·교체 디스크·예비 디스크의 형식을 한 정책으로 맞추는 편이 복구가 단순하다.

## 가격보다 큰 교체 비용

실제 판매가가 아닌 가상 계산이다. 4베이 RAID 5에서 12TB HDD 한 개를 교체한다고 가정하면 총비용은 `새 HDD + 배송·작업 + 예비 디스크 + 외부 백업 + 재빌드 중 서비스 중단 비용`이다. 4Kn 디스크가 512e 모델보다 3만 원 싸더라도 호환되지 않아 임시 디스크를 다시 사야 하면 절약액은 사라진다. 반대로 단일 외장 백업이라면 4Kn을 피하고 PC에서 바로 읽히는 512e 모델을 고르는 것이 복원 시간을 줄일 수 있다.

- [ ] 기존 풀의 정확한 모델·섹터 형식을 기록했다.
- [ ] DSM 7.4·QTS 5.2 호환 목록에서 새 모델을 확인했다.
- [ ] TrueNAS는 풀 생성 전 `ashift` 정책과 예비 디스크를 정했다.
- [ ] 교체 전 외부 백업에서 사진 한 개와 Docker 설정 파일을 복원해 봤다.
- [ ] 재빌드가 끝날 때까지 두 번째 디스크를 교체하지 않을 계획을 세웠다.

참고: [QNAP 4Kn·512e 설명](https://www.qnap.com/it-it/how-to/faq/article/will-configuring-4kn-4k-native-sector-in-my-hdd-improve-the-nas-performances), [QNAP HDD 호환성 목록](https://www.qnap.com/en-us/support/drive-compatibility), [Synology HDD 마이그레이션 환경](https://kb.synology.com/en-global/DSM/tutorial/How_to_migrate_between_Synology_NAS_DSM_6_0), [TrueNAS SCALE 25.10 하드웨어 가이드](https://www.truenas.com/docs/scale/25.10/gettingstarted/scalehardwareguide/).

짧게 정리하면 **기존 RAID 교체는 같은 섹터 형식과 공식 호환 모델**, 새 TrueNAS 풀은 생성 시점의 정렬 정책, 외장 백업은 범용 복원이 기준이다. 4Kn이라는 숫자만 보고 구매하지 말고, 디스크 한 개가 실패했을 때 같은 형식의 예비품과 백업에서 실제로 돌아오는지까지 비용에 넣어야 한다.

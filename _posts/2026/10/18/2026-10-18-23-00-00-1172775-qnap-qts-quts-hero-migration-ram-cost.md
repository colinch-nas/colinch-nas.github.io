---
layout: post
title: "QNAP QTS·QuTS hero 전환 호환성 - 8GB·16GB RAM과 재초기화 비용"
description: "QNAP QTS에서 QuTS hero h6.0으로 바꿀 때 필요한 호환성, 8GB·16GB RAM 기준, 백업·복원과 저장장치 총비용을 구매 전에 판단한다."
date: 2026-10-18
tags: [QNAP, QTS, QuTSHero, ZFS, NAS, 백업복원]
comments: true
share: true
---

![QNAP NAS 운영체제 전환과 저장장치 백업 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

QNAP을 새로 사면서 ZFS가 필요한 사람은 QuTS hero를 처음부터 선택할 수 있지만, 이미 QTS NAS를 운영 중이라면 **RAM 가격보다 임시 백업 저장장치와 복원 시간을 먼저 계산해야 한다**. QTS와 QuTS hero는 파일시스템이 달라서 무중단 업그레이드가 되지 않고, 전환 과정에서 디스크가 초기화된다. 단순 파일 공유와 8GB 이하 장비라면 굳이 바꿀 필요가 없다.

## 8GB와 16GB의 차이

QNAP 공식 기준으로 QuTS hero는 최소 8GB 메모리가 필요하고, 인라인 중복 제거는 최소 16GB, 권장 32GB 이상이다. 따라서 “ZFS니까 RAM을 많이 사야 한다”보다 실제 기능을 나눠 봐야 한다.

| 사용 조건 | QTS | QuTS hero 판단 | 추가 비용의 핵심 |
|---|---|---|---|
| 2~4베이, 문서·사진·SMB 중심 | 충분 | 8GB 미만이면 부적합 | RAM을 사지 않아도 됨 |
| 4~8베이, 압축·스냅샷 중심 | 가능 | 8GB 이상부터 검토 | RAM과 백업 디스크 |
| VM 백업·반복 데이터·중복 제거 | 제한적 | 16GB 이상, 32GB 권장 | RAM, SSD 시스템 풀, 복원 시간 |

첫 번째 스토리지 풀에는 시스템 드라이브와 앱도 놓인다. NVMe SSD 두 개를 RAID1로 추가하면 호환성·유지보수 비용이 생기므로, 앱이 적은 2베이는 HDD 풀로 시작해도 된다.

## 전환은 업그레이드가 아니라 마이그레이션이다

QNAP FAQ와 QuTS hero 제품 문서 기준으로 QTS(ext4)에서 QuTS hero(ZFS)로 직접 전환할 수 없다. NAS를 재초기화한 뒤 새 풀을 만들고 데이터를 되돌려야 한다. 디스크를 그대로 꽂은 채 시도하면 데이터가 지워질 수 있으므로, 아래 비용을 한 묶음으로 계산한다.

`총비용 = RAM 업그레이드 + 임시 백업 디스크/다른 NAS + 새 SSD 또는 HDD + 작업 중 전기료·중단 시간`

가상 사례로 4베이에서 4×12TB를 운영한다면, 백업 대상은 RAID 표시 용량이 아니라 실제 데이터와 버전 보관분이다. USB HDD 한 개에 들어가지 않으면 여러 디스크나 다른 NAS를 임시 경로로 잡는다. QTS 설정 백업만으로 ZFS 풀이 만들어지지는 않는다.

## 구매 전 복원 체크리스트

- [ ] 내 모델이 QuTS hero 지원 목록에 있고, 필요한 펌웨어 조건을 만족하는가
- [ ] 공유 폴더·Docker 볼륨·앱 설정을 별도 저장했는가
- [ ] 원본 NAS를 초기화해도 남는 백업 사본이 두 곳 이상인가
- [ ] 사진·대용량 파일·권한 폴더를 실제로 복원해 열어봤는가
- [ ] 복원 중단 시간과 실패 시 QTS 복귀 경로를 정했는가

QTS에서 SMB·미디어·가벼운 Docker만 쓰고 중복 제거가 필요 없다면 운영체제 전환보다 외장 백업 디스크와 UPS가 먼저다. 스냅샷·압축이 필요하고 초기화 후 복원할 임시 저장공간까지 확보했을 때 QuTS hero 비용을 계산할 근거가 생긴다.

참고한 기준은 [QNAP QTS·QuTS hero 비교 및 메모리 요구사항](https://www.qnap.com/en-us/operating-system/quts-hero), [QTS에서 QuTS hero로 직접 마이그레이션할 수 없는 이유](https://www.qnap.com/en-us/how-to/faq/article/can-i-migrate-the-firmware-from-qts-to-quts-hero), [QNAP 재초기화 안내](https://www.qnap.com/en/how-to/faq/article/how-do-i-reinitialize-the-nas)다. 페이지는 2026년 10월 18일 확인했다.

짧게 판단하면 **새 장비라면 QuTS hero 지원 모델과 16GB 이상 RAM을 함께 사고, 기존 QTS라면 백업·임시 저장공간·중단 시간을 확보하지 못하는 한 전환하지 않는 편이 낫다**. RAID와 RAM은 초기화로 지워진 데이터를 대신 복구해 주지 않는다.

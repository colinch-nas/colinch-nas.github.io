---
layout: post
title: "TrueNAS SCALE 25.10 부트 디스크 - USB·SATA SSD 호환성과 복구 총비용"
description: "TrueNAS SCALE 25.10 홈서버에서 USB 플래시, SATA SSD, NVMe SSD 중 부트 디스크를 고르는 기준과 설정 복구·미러 구성까지 총비용으로 판단한다."
date: 2026-10-14
tags: [TrueNAS, TrueNASSCALE, HomeLab, NAS보안, 백업복원]
comments: true
share: true
---

![TrueNAS SCALE 홈서버의 부트 디스크와 저장장치 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

그림에서 볼 부분은 데이터 디스크와 운영체제가 올라가는 부트 디스크를 따로 계산해야 한다는 점이다.

TrueNAS SCALE 25.10을 1~2명이 SMB와 가벼운 앱만 쓰면 20GB 이상 SATA SSD 한 개로 시작해도 된다. USB 플래시를 부트 디스크로 계속 쓰는 구성은 새로 만들지 않는 편이 안전하다. 반대로 VM·Docker 앱·업무 데이터를 함께 운영하면 SATA SSD 2개 미러와 설정 파일 백업 비용까지 예산에 넣어야 한다. 부트 디스크는 데이터 풀의 백업이 아니므로, 부트 디스크를 비싸게 산다고 복구 문제가 사라지지는 않는다.

## USB·SATA SSD·NVMe 선택 기준

TrueNAS SCALE 25.10 공식 설치 문서는 20GB 부트 장치를 요구한다. 하드웨어 가이드는 SSD를 필수라고 하지는 않지만 USB 스틱과 회전식 디스크는 권장하지 않는다. 설치할 때 쓰는 USB와 실제 운영 부트 장치를 혼동하면 안 된다.

| 선택지 | 맞는 환경 | 비용과 실패 조건 |
|---|---|---|
| USB 플래시 | 기존 장비의 임시 테스트 | 초기 비용은 낮지만 쓰기 내구도와 교체 시점 예측이 어렵다 |
| SATA SSD 1개 | 2~4베이, 파일 공유·백업 중심 | 가장 단순하지만 SSD 고장 때 재설치와 설정 복원이 필요하다 |
| SATA SSD 2개 미러 | VM·앱·여러 사용자 | 디스크 1개 고장 대응은 좋아지지만 슬롯·전원·SSD 비용이 늘어난다 |
| NVMe SSD 2개 미러 | M.2 슬롯이 남는 미니PC·홈랩 | 베이를 아끼지만 보드의 부팅 지원과 NVMe 인식 여부를 따로 확인한다 |

TrueNAS는 부트 풀 미러를 구성할 수 있지만, 미러가 설정 백업을 대신하지는 않는다. NVMe 부팅이 불확실한 예시 환경이라면 SATA SSD가 더 낮은 운영 리스크를 가진다. 호환성은 메인보드의 UEFI, 슬롯 공유, 부트 순서를 확인해야 한다.

## 용량보다 복구 시간을 계산한다

가격은 2026년 10월 14일 실제 판매가가 아닌 가상 비교값이다. 1개당 4만 원인 SATA SSD 2개, 예비 SSD 1개 4만 원, 설정 백업을 보관할 외장 디스크 비용 8만 원으로 잡으면 부트 복구 예산은 20만 원이다. SSD 미러만 사고 예비 디스크를 생략하면 고장 당일 구매와 재설치 시간을 비용으로 떠안는다.

부트 디스크 장애에 대비해 `System Settings → General Settings → Manage Configuration`에서 설정 파일을 내보내고, 암호화 키와 앱·VM 데이터 위치를 기록한다. 새 SSD에 같은 SCALE 계열을 설치한 뒤 설정을 복원하고, IP·공유 폴더·앱을 확인하는 절차까지 적어 둔다.

## 구매 전 체크리스트

- [ ] TrueNAS SCALE 25.10 설치 대상이 데이터 디스크와 분리되어 있는가
- [ ] 메인보드가 선택한 SATA·NVMe 장치의 UEFI 부팅을 지원하는가
- [ ] 부트 장치 1개 고장 뒤 재설치할 예비 디스크가 있는가
- [ ] 설정 파일, 암호화 키, 앱·VM 복구 위치를 별도로 보관했는가
- [ ] 설정 복원 후 SMB·앱·VM 확인 절차를 적어 두었는가
- [ ] USB는 운영용이 아니라 설치·긴급 복구 매체로만 쓰는가

2베이 NAS에서 파일 공유만 한다면 부트 SSD 미러보다 외장 백업 디스크가 우선이다. 4베이 이상에서 VM과 앱을 함께 돌린다면 SSD 미러와 예비 장치가 복구 시간을 줄인다. 부트 디스크는 최대 성능보다 “고장 당일 얼마 만에 서비스를 올릴 수 있는가”로 판단한다.

출처: [TrueNAS SCALE 25.10 설치 문서](https://www.truenas.com/docs/scale/25.10/gettingstarted/install/installingscale/), [TrueNAS 하드웨어 가이드](https://www.truenas.com/docs/scale/25.04/gettingstarted/scalehardwareguide/), [TrueNAS 25.10 업그레이드·설정 복구 안내](https://www.truenas.com/docs/scale/25.10/gettingstarted/printpreview/)

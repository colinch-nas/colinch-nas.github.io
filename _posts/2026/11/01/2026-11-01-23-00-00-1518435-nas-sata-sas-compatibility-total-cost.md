---
layout: post
title: "NAS SATA·SAS 호환성 - TrueNAS SCALE 25.10 HBA·백플레인 총비용 판단"
description: "TrueNAS SCALE 25.10과 QNAP 엔터프라이즈 NAS에서 SATA·SAS 디스크, HBA, 백플레인의 호환 조건과 복구·교체까지 포함한 총비용을 판단한다."
date: 2026-11-01
tags: [TrueNAS, QNAP, HomeLab, 저장장치, NAS호환성, 백업]
comments: true
share: true
---
![NAS 백플레인과 하드디스크가 연결된 홈서버 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

그림에서 볼 것은 디스크 모양이 아니라 연결 경로다. 2~4베이 일반 NAS에 디스크만 꽂는다면 SATA HDD가 맞지만, TrueNAS SCALE 25.10을 8~12베이 서버나 JBOD로 확장한다면 SAS HBA·백플레인까지 함께 사야 한다. SAS 디스크를 SATA 전용 포트에 꽂아도 작동하지 않으므로 중고 서버 디스크를 싸게 사는 선택은 연결 규격부터 확인해야 한다. 이 글은 장비를 직접 시험한 결과가 아니라 2026년 11월 1일 공식 문서와 가상 비용을 바탕으로 한다.

## SATA와 SAS는 이렇게 갈린다

| 환경 | 디스크·연결 선택 | 구매하지 않아도 되는 것 |
|---|---|---|
| 2~4베이 Synology·QNAP | NAS용 SATA HDD + 기본 백플레인 | SAS HBA, SAS 디스크 |
| 4~8베이 자작 TrueNAS | SATA HDD + 메인보드 SATA 또는 IT-mode HBA | SAS 디스크와 확장기 |
| 8베이 이상 서버·JBOD | SAS HBA + SAS 백플레인, 필요 시 SAS expander | 소비자용 SATA 확장카드 |
| 기존 SAS 디스크 재활용 | SAS를 지원하는 HBA·백플레인 | SATA 포트만 있는 본체 |

TrueNAS 공식 가이드는 SATA를 일반적인 디스크 인터페이스로 설명하면서, SAS 기반 백플레인은 SATA 디스크를 STP로 연결할 수 있다고 안내한다. 반대 방향은 성립하지 않는다. SAS 디스크는 SATA 포트용 장비에서 동작하지 않는다. 또 ZFS가 디스크 일련번호와 SMART 정보를 직접 확인해야 하므로 하드웨어 RAID 카드보다 HBA의 디스크 패스스루 구성이 적합하다.

QNAP의 QDA-SA 같은 어댑터는 SATA 6Gbps 디스크를 3.5인치 SAS 베이에 넣는 용도다. 이것이 모든 SAS 장비를 SATA NAS로 바꿔 주는 변환기는 아니다. 모델별 백플레인, 펌웨어, 호환 목록을 확인하지 않고 어댑터부터 사면 인식 실패 뒤 교체 비용이 생긴다.

## 가격보다 복구 경로를 먼저 계산한다

가상 사례로 8베이 TrueNAS 서버에 12TB SATA HDD 8개를 넣는다고 하자. SATA 구성은 HBA 없이 시작할 수 있지만, 메인보드 포트가 부족하면 IT-mode HBA와 케이블 비용이 추가된다. SAS 중고 디스크 구성은 디스크 단가가 낮아 보여도 HBA, 백플레인, 팬과 호환 케이블, 예비 디스크까지 필요하다. 구매비는 아래처럼 계산한다.

`총비용 = 디스크 + HBA/백플레인 + 케이블·팬 + 예비 디스크 + 백업 저장소 + 복구 시간`

SATA 디스크가 있는 1~4베이 가정 NAS라면 SAS로 바꿀 이유가 거의 없다. 반대로 12개 이상 디스크를 한 섀시나 외장 선반에 연결하고 확장기 하나로 포트를 늘릴 계획이면 SAS가 관리와 확장에 유리하다. 다만 중고 SAS 디스크는 SMART 기록, 사용 시간, 펌웨어, 섹터 형식을 확인하고 동일 모델 예비품을 한 개 남겨야 한다.

## 장착 전 체크리스트

- [ ] 본체 포트가 SATA인지, SAS HBA와 백플레인이 있는지 적었다.
- [ ] SAS 디스크를 SATA 포트에 연결하는 계획을 세우지 않았다.
- [ ] TrueNAS에서는 HBA를 IT/JBOD 패스스루로 구성할 수 있는지 확인했다.
- [ ] 디스크 모델·섹터 형식·펌웨어가 호환 목록에 있는지 확인했다.
- [ ] 풀 생성 전에 설정 파일, 암호화 키, 별도 백업을 준비했다.
- [ ] 장애 디스크를 교체할 동일 용량 이상의 예비품과 복원 순서를 적었다.

공식 기준은 [TrueNAS SCALE 하드웨어 가이드](https://www.truenas.com/docs/scale/gettingstarted/scalehardwareguide/), [QNAP 드라이브 호환성 목록](https://www.qnap.com/en-us/support/drive-compatibility), [QNAP QDA-SA 제품 문서](https://www.qnap.com/en-us/product/qda-sa)다.

요점은 SAS가 더 좋은 디스크라는 뜻이 아니다. 2~4베이는 SATA 구성이 단순하고 싸며, 대규모 TrueNAS·JBOD는 HBA와 백플레인까지 맞출 때만 SAS의 확장성이 살아난다. 디스크 가격만 보고 중고 SAS를 사기보다, 인식 실패와 복구용 예비품까지 포함한 총비용으로 판단해야 한다.

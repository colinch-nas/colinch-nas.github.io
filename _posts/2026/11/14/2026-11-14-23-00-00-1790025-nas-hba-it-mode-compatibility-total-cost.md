---
layout: post
title: "NAS HBA 카드 호환성 - TrueNAS SCALE 25.10·Synology DSM 7.4·QNAP QTS 5.2 총비용"
description: "NAS 디스크를 늘리려고 HBA·RAID 카드를 살 때 TrueNAS SCALE 25.10, Synology DSM 7.4, QNAP QTS 5.2의 호환 조건과 케이블·냉각·복구까지 포함한 총비용을 비교한다."
date: 2026-11-14
tags: [TrueNAS, Synology, QNAP, HBA, NAS, HomeLab]
comments: true
share: true
---

![NAS HBA 카드와 디스크 베이 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

이 그림에서 볼 부분은 HBA 카드가 디스크를 직접 연결하는 부품이지, NAS 운영체제의 RAID를 대신하는 장치가 아니라는 점이다.

TrueNAS SCALE 25.10에서 직접 조립한 서버의 디스크 수를 늘릴 목적이라면 **IT 모드 HBA**가 선택지다. 반대로 Synology DSM 7.4나 QNAP QTS 5.2 완제품에 범용 RAID 카드를 꽂아 베이를 늘리려는 계획은 호환성과 지원 범위 때문에 피하는 편이 안전하다. 1~4베이에서 디스크 두세 개만 쓰는 환경이라면 카드보다 외장 백업 디스크를 사는 편이 총비용이 낮다.

## 운영체제별로 달라지는 조건

| 환경 | 가능한 방향 | 피해야 할 가정 | 추가 비용이 생기는 지점 |
|---|---|---|---|
| TrueNAS SCALE 25.10 | LSI/Broadcom 계열 HBA를 IT·패스스루 모드로 연결 | 하드웨어 RAID 볼륨을 ZFS에 그대로 전달 | HBA, SAS 케이블, 팬, 예비 카드 |
| Synology DSM 7.4 | 공식 호환 확장 유닛·네트워크 카드 사용 | 아무 PCIe HBA가 내부 저장 풀을 늘림 | 전용 확장 유닛, 케이블, 호환 HDD |
| QNAP QTS 5.2 | 모델별 QM2·QXP·확장 유닛 조합 확인 | 제3자 M.2/RAID 카드가 저장 풀을 보장 | 전용 카드, 외장 선반, 전원·랙 공간 |

TrueNAS 공식 하드웨어 가이드는 HBA의 RAID 기능을 끄고 JBOD 또는 패스스루로 쓰라고 안내한다. HBA가 디스크의 시리얼 번호와 SMART 정보를 그대로 운영체제에 전달해야 ZFS가 디스크 상태를 제대로 판단할 수 있다. 배터리 백업이 고장 난 하드웨어 RAID 카드의 쓰기 캐시는 데이터 손실을 만들 수 있다는 경고도 있다. [TrueNAS SCALE 25.10 하드웨어 가이드](https://www.truenas.com/docs/scale/25.10/gettingstarted/scalehardwareguide/)

QNAP QTS 5.2는 PCIe M.2를 제3자 어댑터 카드에 꽂은 경우 저장 풀·정적 볼륨에 사용할 수 없다고 명시한다. QNAP 확장 선반도 NAS 모델에 따라 Mini-SAS 포트나 QXP 호스트 버스 어댑터가 따로 필요하다. “PCIe 슬롯이 있다”보다 `NAS 모델 → QTS 버전 → 카드 모델 → 확장 유닛` 순서로 호환표를 확인해야 한다. [QNAP QTS 5.2 디스크 유형](https://docs.qnap.com/operating-system/qts/5.2.x/en-us/disk-types-B53F91D1.html), [QNAP 확장 유닛 호환성](https://www.qnap.com/en-us/compatibility-expansion)

Synology는 호환 목록에서 확장 유닛과 PCIe 네트워크 카드 등을 모델별로 검증하며, 목록 밖 장치는 인식 불가·불안정·제한적인 지원이 생길 수 있다고 안내한다. DSM 7.4 본체의 빈 슬롯을 범용 HBA 슬롯으로 해석하지 말고, 지원되는 확장 유닛과 독립 볼륨 여부를 확인해야 한다. [Synology 호환 목록 안내](https://kb.synology.com/en-my/DSM/tutorial/Can_I_use_devices_that_are_not_on_the_Products_Compatibility_List)

## 카드값보다 복구 경로가 먼저다

아래는 실제 판매가가 아닌 **가상 사례**다. TrueNAS DIY 서버에 중고 HBA 8만 원, 내부 SAS 케이블 3만 원, 냉각 팬과 브래킷 2만 원, 같은 계열 예비 카드 8만 원을 잡으면 카드 추가 비용은 21만 원이다. 여기에 디스크 선반과 전원 장치를 더하면 기존 4베이 NAS를 확장 유닛으로 바꾸는 비용과 차이가 작아질 수 있다.

| 선택 | 유리한 환경 | 구매하지 않아도 되는 조건 | 장애·복구 비용 |
|---|---|---|---|
| TrueNAS + IT 모드 HBA | 5베이 이상, 직접 조립, ZFS 풀을 장기 운영 | 현재 메인보드 SATA 포트가 남고 디스크가 4개 이하 | 펌웨어 플래시와 케이블 장애를 직접 해결 |
| 제조사 확장 유닛 | 기존 DSM·QTS 관리 화면과 지원을 유지 | 단순히 백업 용량만 늘리려는 1~2인 사용자 | 본체 장애 때 확장 볼륨 복구 장비가 필요 |
| 외장 USB HDD | 백업·아카이브, 연결을 자주 분리할 수 있음 | 상시 실행 VM이나 데이터베이스 저장소 | USB 케이스·파일시스템·복원 PC 호환성 |

HBA를 설치한 뒤에는 카드가 디스크를 보이는지만 확인하지 않는다. 각 디스크의 모델·시리얼·SMART가 보이는지, 재부팅 뒤 장치 순서가 바뀌지 않는지, 풀을 다른 HBA에 연결했을 때 설정 백업으로 가져올 수 있는지를 점검한다. 펌웨어가 IR 모드라면 IT 모드 전환 가능 여부와 제조사 지원 범위를 기록한다.

### 구매 전 체크리스트

- [ ] 운영체제가 디스크를 개별 장치로 인식하는가
- [ ] HBA가 IT·JBOD·패스스루 모드를 지원하는가
- [ ] SAS 케이블의 내부·외부 규격과 커넥터가 맞는가
- [ ] 카드 방열판과 케이스 airflow가 충분한가
- [ ] 카드 고장 때 교체품과 설정 백업이 있는가
- [ ] HBA 비용 대신 외장 백업 HDD를 추가했을 때 복구 시간이 더 짧지 않은가

짧게 정리하면 **TrueNAS는 IT 모드 HBA, Synology·QNAP은 제조사 호환표에 있는 확장 경로**를 기준으로 잡으면 된다. 디스크 수가 적고 백업이 부족한 상태에서 HBA부터 사면 저장 용량은 늘어도 복구 가능성은 좋아지지 않는다. 카드·케이블·냉각·예비품까지 계산한 뒤에도 운영 중단 시간이 줄어드는 경우에만 확장을 선택하는 편이 맞다.

참고 문서는 2026년 11월 14일 확인했다. 특정 HBA나 NAS를 직접 연결해 시험한 결과가 아니라 제조사 문서와 가상 비용을 바탕으로 한 구매 판단이다.

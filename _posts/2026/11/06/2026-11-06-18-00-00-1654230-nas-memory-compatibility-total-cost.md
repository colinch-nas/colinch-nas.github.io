---
layout: post
title: "NAS 메모리 호환성 - Synology DSM 7·QNAP QTS·TrueNAS SCALE 25.10 증설과 총비용"
description: "NAS RAM을 무작정 늘리기 전에 Synology DSM 7, QNAP QTS, TrueNAS SCALE 25.10의 호환 조건과 사용자 수·컨테이너 규모별 증설 기준, 실패와 복구 비용을 정리한다."
date: 2026-11-06
tags: [Synology, QNAP, TrueNAS, DSM, HomeLab]
comments: true
share: true
---

![NAS 메모리 모듈과 홈서버 저장장치 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

그림에서 볼 것은 메모리 용량보다 NAS 운영 범위다. 파일 공유만 하는 1~3명 환경이면 기본 RAM을 유지하고 외부 백업에 예산을 쓰는 편이 낫다. 반대로 Docker 컨테이너, VM, 여러 사용자, TrueNAS ZFS 캐시를 함께 쓰면 호환되는 RAM 증설이 장애와 교체 시간을 줄인다. 이 글은 장비를 직접 시험한 결과가 아니라 2026년 11월 6일 공식 문서와 가상 비용을 바탕으로 한다.

## 장비별로 “호환”의 뜻이 다르다

| 환경 | 구매 전 확인할 것 | 증설 판단 | 실패했을 때 비용 |
|---|---|---|---|
| Synology DSM 7 | 정확한 모델·개정판·호환 목록·슬롯 순서 | 파일 공유만 하면 기본값, 컨테이너가 늘면 공식 목록의 모듈 | 부팅 실패, 지원 제한, 설정 복구 시간 |
| QNAP QTS 5 | 하드웨어 사양의 최대 용량·모듈 규격·인증 RAM | 1~3명은 기본값, 앱·가상화 사용 시 사양표 안에서 증설 | 메모리 오류 진단과 서비스 중단 |
| TrueNAS SCALE 25.10 | 메인보드의 DDR 세대·ECC 지원·슬롯 구성 | 최소 8GB에서 시작하고 VM·iSCSI·L2ARC에 여유분 추가 | 풀 접근 지연, 앱 재시작, 복구용 콘솔 작업 |

Synology는 제품 호환성 목록과 데이터시트를 확인하라고 안내하고, QNAP도 모델별 최대 메모리와 인증 모듈을 따로 본다. “DDR4 SO-DIMM”처럼 규격 일부가 같다는 이유만으로 구매하면 안 된다. [Synology RAM 호환성 목록](https://www.synology.com/en-us/compatibility?category=rams&search_by=category)

## 사용 규모별 구매 기준

가상 사례로 RAM 단가를 8GB 3만 원, 16GB 6만 원, 32GB 12만 원으로 잡으면 8GB에서 16GB로 올리는 비용은 3만 원이다. 하지만 장비가 8GB를 지원하지 않으면 NAS 교체비가 발생한다.

- 사진·문서 공유 1~3명: 기본 RAM, 외장 HDD 백업과 UPS를 우선한다.
- 컨테이너 3~8개, 사용자 3~8명: 제조사가 허용한 8~16GB를 검토한다. 앱마다 메모리 상한을 확인한다.
- VM·iSCSI·대규모 인덱싱: TrueNAS는 8GB를 시작점으로 보되 서비스 수와 풀 규모에 맞춰 16~32GB 이상을 계산한다. L2ARC를 추가하면 캐시 메타데이터도 RAM을 사용하므로 SSD보다 메모리를 먼저 계산한다.

TrueNAS 공식 하드웨어 가이드는 기본 작업에 최소 8GB를 제시하고 디스크 수·클라이언트·VM·iSCSI·L2ARC에 추가 메모리를 권장한다. ECC는 CPU와 메인보드까지 지원해야 하므로 RAM만 ECC로 바꾸는 방식은 성립하지 않는다. [TrueNAS SCALE 하드웨어 가이드](https://www.truenas.com/docs/scale/gettingstarted/scalehardwareguide/)

## 구매·복구 체크리스트

- [ ] 모델명 뒤의 `+`, `xs`, 개정판까지 확인했다.
- [ ] DDR 세대, SO-DIMM/UDIMM, ECC/비ECC, 전압과 최대 용량을 대조했다.
- [ ] 모듈 추가 전 설정 백업과 앱별 데이터 백업을 외부 저장소에 두었다.
- [ ] 교체 후 부팅, 전체 메모리 인식, 메모리 진단, SMART와 스크럽 일정을 확인한다.
- [ ] 문제가 생기면 기존 모듈로 되돌리고, 앱·VM을 순서대로 복원한다.

증설 전 설정 파일과 중요한 공유 폴더를 다른 장치에 복사하지 않았다면 RAM 구매는 복구 계획이 아니다. 메모리 오류가 반복되면 기존 모듈로 되돌리고, 풀 데이터가 NAS 한 대에만 있다면 실험보다 백업 복원이 우선이다.

요점은 큰 RAM이 항상 이기는 것이 아니라는 점이다. Synology와 QNAP은 모델별 호환표, TrueNAS는 서비스 규모와 ECC·캐시 조건이 기준이다. 기본 파일 공유라면 RAM을 사지 않아도 되며, 컨테이너와 VM을 늘릴 때만 증설하고 외부 백업 검증 비용까지 더해야 한다.

출처: [Synology 메모리 확장 안내](https://kb.synology.com/en-global/DSM/tutorial/How_to_upgrade_memory_capacity_for_Synology_NAS), [QNAP 최대 메모리 안내](https://www.qnap.com/en/how-to/faq/article/what-is-the-maximum-memory-size-that-could-be-supported-on-my-nas), [TrueNAS SCALE 설치 요구사항](https://www.truenas.com/docs/scale/gettingstarted/install/installingscale/)

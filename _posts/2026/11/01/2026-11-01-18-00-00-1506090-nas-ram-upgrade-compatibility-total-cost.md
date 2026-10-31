---
layout: post
title: "NAS RAM 증설 호환성 - Synology DSM 7.3·QNAP QTS·TrueNAS 25.10 총비용 판단"
description: "NAS RAM을 늘릴 때 모델별 최대 용량, ECC·비ECC 호환성, Docker·VM 사용량과 메모리 구매·복구 비용을 함께 판단하는 기준을 정리한다."
date: 2026-11-01
tags: [Synology, QNAP, TrueNAS, DSM, QTS, HomeLab]
comments: true
share: true
---
![NAS 메모리와 저장장치가 보이는 홈서버 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

그림에서 볼 것은 저장장치만이 아니다. NAS의 RAM은 파일 공유, Docker, VM, 캐시가 함께 쓰는 운영 자원이다.

NAS가 1~2명 파일 서버로만 쓰인다면 RAM을 바로 사지 않는 편이 낫다. 반대로 3명 이상이 동시에 작은 파일을 다루거나 Docker·VM·iSCSI를 운영한다면, SSD 캐시보다 먼저 RAM 한도와 호환 모듈을 확인하는 편이 실패 비용을 줄인다. 이 글은 장비를 직접 시험한 결과가 아니라 2026년 11월 1일 확인한 제조사 문서와 가상 구매 조건을 바탕으로 한다.

## 사용자 수보다 서비스 조합을 본다

| 환경 | 우선 확인할 것 | RAM 증설 판단 |
|---|---|---|
| 1~2명, 사진·문서·백업 | 현재 메모리 사용률, 네트워크 병목 | 대체로 구매 보류 |
| 3~5명, Docker·DB·썸네일 | 컨테이너와 파일 공유의 동시 사용량 | 8→16GB를 검토 |
| VM·iSCSI·여러 앱 | VM별 할당량, ZFS ARC, 스왑 | 16GB 이상을 모델별 계산 |
| TrueNAS 25.10, 8베이 초과 | 디스크 수와 서비스 수 | 기본 8GB 뒤 디스크당 추가 RAM을 계산 |

Synology는 메모리 모듈을 모델별 호환 목록에서 확인하라고 안내하며, 비정품 모듈 사용 시 지원 조건이 달라질 수 있다. QNAP도 모델별 최대 용량과 슬롯 순서를 따로 제시한다. 따라서 DDR 세대와 용량만 맞춰 구매하면 된다는 판단은 위험하다. 납품된 NAS가 메모리 납땜형이면 증설 자체가 불가능한 경우도 있다.

TrueNAS 25.10 공식 하드웨어 가이드는 기본 작업에 8GB를 제시하고, 8개를 넘는 디스크마다 1GB를 추가하는 일반 기준을 설명한다. VM·iSCSI에는 16GB 이상, 최적 성능에는 32GB 이상을 권장한다. 중복 제거를 켜면 TB당 약 5GB, L2ARC를 붙이면 캐시 용량에 따른 RAM까지 추가로 잡아야 한다. 이 수치는 모든 홈서버에 적용하는 절대값이 아니라 TrueNAS 공식 계산 출발점이다.

## 구매 전에 계산할 총비용

가상 사례로 4베이 NAS에 3명이 접속하고 Docker DB와 Home Assistant를 함께 운영한다고 하자. 8GB 한 장을 16GB로 바꾸는 비용만 보지 말고, 호환 모듈 1세트, 배송·교체 시간, 부팅 실패 시 원래 모듈로 되돌릴 비용, 설정 백업과 복원 점검 시간을 포함한다. ECC가 필요한 플랫폼에서 비ECC를 싸게 사면 가격 차이는 절약이 아니라 재구매 비용이 된다.

반대로 메모리 사용률이 낮고 느린 이유가 1GbE 네트워크, HDD 순차 읽기, 백업 작업 자체라면 RAM을 늘려도 체감이 작다. 이 경우에는 RAM보다 백업 디스크나 UPS가 우선이다. RAM 증설은 백업이 아니며, 교체 전에는 DSM·QTS 설정 백업과 TrueNAS 설정 파일을 별도 저장한다.

## 복사해 쓰는 구매 체크리스트

- [ ] 정확한 모델명과 DSM·QTS·TrueNAS 버전을 적었다.
- [ ] 제조사 호환표에서 타입, 속도, ECC 여부, 슬롯 수를 확인했다.
- [ ] 현재 메모리와 최대 지원 용량을 확인했다.
- [ ] Docker·VM·iSCSI·중복 제거·L2ARC의 추가 요구량을 따로 계산했다.
- [ ] 교체 전 설정 백업과 원래 모듈로 되돌릴 계획을 준비했다.
- [ ] 증설 후 메모리 테스트와 백업 복원 점검 일정을 잡았다.

공식 근거는 [Synology RAM 호환 목록](https://www.synology.com/en-us/compatibility?category=rams&search_by=category), [Synology 메모리 증설 안내](https://kb.synology.com/da-dk/DSM/tutorial/How_to_upgrade_memory_capacity_for_Synology_NAS), [QNAP 메모리 업그레이드 안내](https://www.qnap.com/en-us/support/con_show.php?cid=9&op=showone), [TrueNAS 25.10 하드웨어 가이드](https://www.truenas.com/docs/scale/25.10/gettingstarted/scalehardwareguide/)다.

요점은 “RAM이 많을수록 좋다”가 아니다. 모델별 호환성과 서비스 조합을 먼저 계산하고, 네트워크·디스크가 병목이면 구매를 보류한다. 증설한다면 모듈 가격보다 실패 시 원상복구와 설정 복원 시간을 총비용에 넣어야 한다.

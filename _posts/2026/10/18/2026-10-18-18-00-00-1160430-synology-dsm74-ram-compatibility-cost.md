---
layout: post
title: "Synology DSM 7.4 RAM 업그레이드 - DS923+·DS1522+ 호환성과 교체 비용"
description: "Synology DSM 7.4 NAS의 공식·비공식 RAM 호환 차이와 메모리 테스트, 보증 리스크를 DS923+·DS1522+ 기준으로 비교한다."
date: 2026-10-18
tags: [Synology, DSM, NAS호환성, 메모리, HomeLab]
comments: true
share: true
---

![Synology NAS 메모리 업그레이드와 호환성 확인](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

그림에서 볼 부분은 RAM 가격보다 모델·규격·복구 시간을 함께 계산해야 한다는 점이다.

DS923+·DS1522+에서 Container Manager와 인덱싱을 함께 돌리면 증설을 검토할 만하다. SMB만 쓰는 1~2인은 현재 사용량부터 본다. 아래는 2026년 10월 18일 공식 문서 기준이며 비공식 모듈을 실제 시험한 결과는 아니다.

## 공식 모듈과 비공식 RAM의 차이

DS923+는 기본 4GB, DS1522+는 8GB DDR4 ECC SODIMM이다. 두 모델 모두 공식 표에 8GB·16GB와 최대 32GB 구성이 있다. 두 슬롯은 같은 모델 번호로 맞춘다.

| 선택 | 호환 확인 | 비용 판단 | 위험 |
|---|---|---|---|
| 공식 ECC SODIMM | 목록 등재 부품 | 비싸지만 지원 예측 가능 | 가격 |
| 비공식 ECC | 모델·속도·랭크 대조 | 초기비 절감 | 부팅·지원 제한 |
| 그대로 사용 | 추가 구매 없음 | 파일 공유면 충분 | 앱 메모리 부족 |

Synology는 비공식 메모리 사용 시 완전한 보증·지원을 제공하지 않을 수 있다고 명시한다. “DDR4 ECC SODIMM”만 같다고 호환되는 것은 아니므로 부품 번호를 확인한다.

## 구매 전 계산

가상 사례로 16GB 모듈을 비공식 8만 원, 공식 20만 원으로 잡으면 32GB 구성 차이는 24만 원이다. 실제 가격은 다르다. 유일한 사진·문서 사본이 있다면 RAM보다 외부 백업을 먼저 산다.

## 교체 후 확인 체크리스트

- [ ] Synology Compatibility List에서 정확한 모델과 RAM 부품 번호를 확인했다.
- [ ] DSM 설정과 중요 폴더를 별도 저장소에 백업했다.
- [ ] 전원을 끄고 모듈을 장착한 뒤 DSM에서 총 용량을 확인했다.
- [ ] Synology Assistant의 Memory Test를 3회 실행했다.
- [ ] SMB 파일, Container Manager 앱, 백업 작업을 확인했다.

공식 안내는 메모리 테스트 3회를 권장한다. 16GB 이하도 1~1.5시간 걸릴 수 있고 NAS가 재부팅된다. 오류가 나오면 공식 부품으로 되돌린다.

파일 공유만 하면 증설하지 않아도 된다. 앱을 함께 돌릴 때는 공식 ECC의 지원 비용, 비공식 RAM의 테스트·백업 비용까지 합산해 결정한다.

출처: [Synology RAM 호환 목록](https://www.synology.com/en-us/compatibility?category=rams&search_by=category), [DS923+ 제품 설명서](https://kb.synology.com/en-ca/HIGs/DS923p_HIG/2), [Synology 메모리 테스트 안내](https://kb.synology.com/en-sg/DSM/tutorial/How_can_I_run_a_memory_test_on_my_Synology_NAS), [Synology DDR4 메모리 안내](https://www.synology.com/en-global/products/DDR4)

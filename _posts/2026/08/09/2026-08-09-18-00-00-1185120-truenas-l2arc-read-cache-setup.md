---
layout: post
title: "TrueNAS L2ARC 설정 - RAM 부족한 홈서버에 읽기 캐시를 붙이는 기준"
description: "TrueNAS 25.10.5 홈서버에서 L2ARC 읽기 캐시를 추가하기 전 필요한 RAM·recordsize 계산과 SSD 설정, ARC 적중률 확인 방법을 정리한다."
date: 2026-08-09
tags: [TrueNAS, OpenZFS, 홈서버, HomeLab, SSD캐시, NAS설정]
comments: true
share: true
---

![TrueNAS 홈서버와 NVMe 읽기 캐시](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

TrueNAS 25.10.5 홈서버에서 파일 읽기가 반복적으로 느리다면 L2ARC(Level 2 Adaptive Replacement Cache)용 SSD를 붙일 수 있다. 다만 L2ARC는 RAM을 대체하지 않는다. 2026년 7월 24일 TrueNAS 공식 글도 RAM이 부족한 상태에서 큰 캐시를 무작정 추가하면 오히려 ARC 메타데이터가 메모리를 잡아먹을 수 있다고 설명한다. 내 기준에서는 RAM 확장이 막혔고, VM 이미지나 프로젝트 파일을 반복해서 읽는 환경일 때만 검토할 기능이다.

## L2ARC를 붙여도 되는 환경

| 환경 | 판단 |
|---|---|
| 32GB RAM, Jellyfin 영상 스트리밍 중심 | 효과가 작다. 순차 읽기라 HDD 풀도 충분하다 |
| 32GB RAM, VM 3~4대와 작은 파일 반복 읽기 | 효과를 측정해볼 만하다 |
| 백업 파일을 한 번 쓰고 다시 읽지 않음 | L2ARC보다 백업 디스크와 네트워크가 우선이다 |
| RAM을 64GB 이상으로 확장 가능 | SSD 캐시보다 RAM 확장이 먼저다 |

TrueNAS 공식 글 기준 L2ARC는 읽기 전용이고, 장치가 고장 나도 원본 데이터는 풀에 남는다. SLOG(동기식 쓰기 보호용 로그 장치)와 역할이 다르므로 쓰기 성능을 올리려고 L2ARC를 추가하면 방향이 틀린다.

## SSD 추가 전 계산

L2ARC 블록마다 ARC에 약 96바이트의 헤더가 필요하다. 예를 들어 1TB SSD에 `recordsize=1M`인 데이터만 담으면 대략 아래처럼 계산한다.

```text
1,000,000,000KB ÷ 1,024KB × 96B ≈ 91MB
```

반대로 VM처럼 `recordsize=16K`에 가까운 데이터가 1TB를 채우면 메타데이터가 약 5.7GB까지 늘어난다. 그래서 16GB RAM 미니PC에 2TB L2ARC를 먼저 붙이는 구성은 피하는 게 좋다. 캐시를 더 크게 만드는 것보다 실제 반복 읽기 데이터에 맞춰 256~512GB부터 시작하는 편이 안전하다.

## TrueNAS 25.10.5에서 SSD 추가

환경은 TrueNAS Community Edition 25.10.5, HDD 4개 RAIDZ1 풀, RAM 32GB, NVMe 500GB다. 중요한 앱과 VM을 중지하고 설정 파일을 외부에 저장한 상태에서 작업한다.

`Storage → Pools`에서 대상 풀의 메뉴를 열고 **Add Vdev**를 선택한다. Vdev 유형은 **Cache**로 지정하고 NVMe SSD를 선택해 저장한다. 데이터 vdev나 로그로 잘못 선택하면 용도가 달라지므로 화면의 유형을 한 번 더 확인한다. 캐시 장치를 하나만 써도 데이터 손실은 없지만, SSD가 죽는 동안 읽기 속도가 원래 풀 수준으로 내려간다.

설정 직후 속도가 오르지 않는 건 정상이다. ARC에서 밀려난 데이터가 L2ARC에 쌓이는 시간이 필요하다. 장치와 풀 상태는 SSH에서 확인한다.

```bash
zpool status
zpool iostat -v 5
arc_summary | egrep "L2ARC|ARC hit"
```

파일 복사 한 번으로 판단하지 말고, 같은 VM 부팅이나 같은 프로젝트 폴더 열기를 3회 이상 반복한다. `zpool iostat`에서 풀 디스크 읽기가 줄고 `arc_summary`의 L2ARC 적중률이 계속 올라가야 투자할 이유가 있다. 적중률이 거의 0%면 SSD를 빼도 체감 손실이 없다는 뜻이다.

## 설정 후 체크리스트

- SSD의 TBW와 온도를 확인하고 SMART 경고를 등록한다.
- L2ARC를 백업으로 생각하지 않는다. 풀 스냅샷과 외부 백업은 그대로 유지한다.
- TrueNAS 업데이트 전 `System → General Settings → Manage Configuration`에서 설정 파일을 내려받는다.
- 1~2주 동안 ARC·L2ARC 적중률과 VM 응답 시간을 기록한다.
- 효과가 없으면 캐시 장치를 제거해도 풀 데이터에는 영향이 없다.

L2ARC는 “SSD를 넣으면 NAS가 빨라진다”는 옵션이 아니다. RAM을 더 넣을 수 없는 TrueNAS 25.10.5 홈서버에서 반복 읽기가 분명하고, 적중률로 효과를 검증할 수 있을 때만 붙이는 보조 계층이다. 공식 참고 자료는 [TrueNAS의 L2ARC 설명](https://www.truenas.com/blog/when-ram-runs-out-l2arc-truenas/)과 [OpenZFS 문서](https://openzfs.github.io/openzfs-docs/Performance%20and%20Tuning/Workload%20Tuning.html)다.

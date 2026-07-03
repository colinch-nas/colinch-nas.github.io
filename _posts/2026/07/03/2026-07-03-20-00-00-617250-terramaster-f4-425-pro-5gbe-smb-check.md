---
layout: post
title: "TerraMaster F4-425 Pro 5GbE SMB - LACP 전 확인할 속도 병목"
description: "TerraMaster F4-425 Pro와 TOS 7 환경에서 5GbE SMB 복사 속도가 기대보다 낮을 때 링크, PC 포트, iperf3, 디스크 병목을 나눠 확인하는 순서다."
date: 2026-07-03
tags: [TerraMaster, NAS설정, HomeLab, 홈서버, NAS보안]
comments: true
share: true
---

![NAS 네트워크 케이블과 스위치 포트를 점검하는 장면](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)
이 그림에서 볼 건 NAS 스펙보다 실제로 어떤 포트와 케이블을 지나가는지가 속도를 결정한다는 점이다.

TerraMaster F4-425 Pro의 듀얼 5GbE는 바로 LACP부터 묶기보다 단일 5GbE 링크가 정상인지 확인해야 한다. 2026년 7월 초 공개된 리뷰들에서도 F4-425 Pro는 Intel N350, 16GB DDR5, 3개 M.2 슬롯, TOS 7을 앞세우지만 실제 체감은 네트워크와 디스크 구성이 같이 맞아야 나온다. 내가 확인한 기준으론 SMB 복사가 280MB/s 근처에서 멈추면 NAS보다 PC 포트나 스위치 경로가 원인인 경우가 많았다.

내 기준 환경은 TerraMaster F4-425 Pro, TOS 7, 4TB HDD 4개 RAID 5, 5GbE 스위치, Windows 11 PC다. 기대치는 단일 대용량 파일 기준 450~550MB/s 정도로 잡았다. 1GbE면 110MB/s, 2.5GbE면 270~290MB/s 근처가 흔하다.

| 증상 | 확인할 곳 | 판단 기준 |
|---|---|---|
| 110MB/s 근처 | 링크 속도 | 1GbE로 협상됨 |
| 280MB/s 근처 | PC NIC·스위치 | 2.5GbE 경로 |
| 450MB/s 이상 | 정상 범위 | 단일 5GbE 가능 |
| 속도 출렁임 큼 | 디스크·SMB | 작은 파일 또는 캐시 영향 |

## 링크부터 고정해서 본다

TOS 7에서 `제어판 -> 네트워크 -> 네트워크 인터페이스`로 들어가 연결 속도를 확인한다. 여기서 5000Mbps가 아니라 1000Mbps로 보이면 SMB 설정을 만질 필요가 없다. 케이블을 Cat.6 이상으로 바꾸고, 중간에 공유기 1GbE 포트를 지나지 않는지 본다.

Windows에서도 어댑터 상태를 확인한다.

```powershell
Get-NetAdapter | Select-Object Name, LinkSpeed
```

여기서 PC가 2.5Gbps로 붙어 있으면 NAS가 5GbE여도 복사 속도는 2.5GbE 한계에 걸린다. 나는 이걸 놓치고 TOS 7 SMB 옵션만 20분 넘게 만졌다.

## iperf3로 디스크를 빼고 잰다

파일 복사는 HDD RAID, 캐시, 백신 검사까지 섞인다. 네트워크만 따로 재야 한다. TOS 7에서 Docker를 쓴다면 iperf3 컨테이너를 잠깐 띄우면 된다.

NAS에서 iperf3 서버를 띄운다.

```bash
docker run --rm -p 5201:5201 networkstatic/iperf3 -s
```

PC에서 병렬 연결로 테스트한다.

```bash
iperf3 -c 192.168.10.20 -P 4
```

결과가 4.5Gbps 전후면 네트워크는 정상이다. 2.3Gbps 전후면 PC, 스위치, 케이블 중 하나가 2.5GbE다. 900Mbps 전후면 거의 1GbE 경로다. 이 단계에서 숫자가 안 나오면 SMB 튜닝은 뒤로 미룬다.

## SMB는 큰 파일 하나로 확인한다

네트워크가 정상이라면 20GB 이상 단일 파일로 복사한다. 사진 1만 장 같은 작은 파일 묶음은 메타데이터 처리 때문에 NAS 성능 비교에 맞지 않는다.

| 테스트 | 목적 | 기대값 |
|---|---|---|
| iperf3 | 네트워크만 확인 | 4.5Gbps 전후 |
| 20GB 단일 파일 | SMB 순차 복사 | 450MB/s 전후 |
| 작은 사진 폴더 | 실제 체감 | 편차 큼 |
| 동시 2대 복사 | LACP 체감 | 클라이언트별 분산 |

LACP(링크 어그리게이션)는 PC 한 대의 단일 복사를 10GbE처럼 만들어주지 않는다. 여러 클라이언트가 동시에 붙을 때 포트가 나뉘는 효과에 가깝다. 집에서 PC 한 대로 영상 편집 파일을 옮기는 용도라면 LACP보다 단일 5GbE가 제대로 붙는지가 기준이다.

주의할 점은 TOS 7이 DSM만큼 오래 검증된 운영체제는 아니라는 부분이다. 새 NAS를 업무 원본 저장소로 바로 쓰기보다 기존 NAS나 외장 디스크에 한 번 더 백업을 둔다. 짧게 정리하면 F4-425 Pro 5GbE 속도 점검은 링크 속도, iperf3, 큰 파일 SMB, 작은 파일 체감 순서로 보면 된다. 숫자가 2.5GbE에서 막히면 NAS 스펙이 아니라 경로 문제일 가능성이 높다.

출처: [TechRadar TerraMaster F4-425 Pro 리뷰](https://www.techradar.com/computing/terramaster-f4-425-pro-nas-review)

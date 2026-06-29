---
layout: post
title: "TerraMaster F4-425 Pro TOS 7 초기 설정 — AI 내장 NAS, 실제로 써보니"
description: "TerraMaster F4-425 Pro(Intel N350 8코어, 16GB DDR5, 듀얼 5GbE)에 TOS 7 설치 및 초기 설정 방법. OpenClaw AI 기능과 Security Advisor 버그 등 실사용 후기."
date: 2026-06-27
tags: [TerraMaster, NAS설정, 홈서버구축, 홈랩, NAS]
comments: true
share: true
---

![TerraMaster F4-425 Pro NAS 서버 하드웨어](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=800&q=80)

TerraMaster F4-425 Pro에 TOS 7을 올리면 Intel N350 8코어에 듀얼 5GbE, 그리고 AI 기능 OpenClaw까지 한 번에 쓸 수 있다. 같은 가격대 Synology 대비 하드웨어 스펙이 눈에 띄게 좋아서 설정해봤는데, 소프트웨어 쪽에서 걸리는 부분이 몇 가지 있었다.

## 환경 정보

| 항목 | 사양 |
|------|------|
| NAS 모델 | TerraMaster F4-425 Pro |
| CPU | Intel N350 (8코어, 최대 3.4GHz) |
| RAM | 16GB DDR5 (기본 포함) |
| 네트워크 | 듀얼 5GbE (LACP 링크 어그리게이션 지원) |
| OS | TOS 7.0 (Linux 커널 6.12 기반, 현재 베타) |
| M.2 슬롯 | 3개 (NVMe SSD 캐시·스토리지 겸용) |

## 왜 지금 F4-425 Pro인가

Synology DS923+와 비슷한 가격대에 N350 8코어, DDR5 16GB 기본 탑재, 듀얼 5GbE까지 들어있다. Synology는 메모리 별도 구매에 NVMe 슬롯도 2개인데 여기는 M.2 슬롯이 3개다. 하드웨어 수치만 보면 가성비 면에서 상당히 앞선다.

올해 TOS 7과 함께 출시된 OpenClaw가 관심을 끌었다. NAS OS 레벨에서 AI를 내장한 건 TerraMaster가 처음이라고 하는데, 실제로 얼마나 쓸 만한지 확인하고 싶었다.

## TOS 7 초기 설정 방법

### 1. 접속 및 어드민 계정 설정

전원 켜고 LAN 연결하면 DHCP로 IP가 할당된다. 같은 네트워크에서 브라우저로 아래 주소에 접속한다.

```
http://find.terra-master.com
```

공유기 DHCP 목록에서 IP를 직접 확인해도 된다. `192.168.x.x:8181` 형태로 접속하면 관리 페이지가 뜬다.

초기 어드민 계정:
```
사용자: admin
비밀번호: 없음 → 첫 접속 시 설정
```

TOS 7은 초기화 과정이 이전 버전보다 훨씬 간결해졌다. 이전에 TOS 5를 써봤던 사람이라면 차이가 바로 느껴진다. 클릭 수가 줄었고 UI도 정리됐다.

### 2. 스토리지 풀 생성

초기 설정 마법사 끝나면 스토리지 풀부터 만든다.

**제어판 → 스토리지 → 스토리지 풀 → 생성**

RAID 옵션 선택 기준:

| RAID 유형 | 필요 디스크 | 특징 |
|----------|-----------|------|
| RAID 1 | 2개 이상 | 완전 미러링, 속도보다 안정 |
| RAID 5 | 3개 이상 | 1개 고장 허용, 용량 효율 중간 |
| TRAID | 2개 이상 | 디스크 추가 시 확장 가능 |

4베이에 4TB × 4개라면 RAID 5가 제일 현실적이다. 유효 용량 약 12TB에 디스크 1개 고장까지 버틴다.

```
팁: 스토리지 풀 초기화는 4TB × 4기준 30분~1시간 소요.
완료 전에 브라우저 닫으면 중간에 끊기는 경우 있음.
```

### 3. 듀얼 5GbE 링크 어그리게이션 설정

두 5GbE 포트를 LACP로 묶으면 실측 1010 MB/s까지 나온다. 이게 F4-425 Pro의 핵심 장점이다.

**제어판 → 네트워크 → 네트워크 인터페이스 → 결합 만들기**

```
결합 유형: IEEE 802.3ad 동적 링크 어그리게이션 (LACP)
```

단, 공유기·스위치 쪽에서도 LACP를 지원해야 속도가 나온다. 일반 가정용 공유기는 LACP 미지원이 대부분이다. 언관리드 스위치도 마찬가지.

LACP 지원 스위치가 없다면 **Active-backup** 모드로 설정하는 게 낫다. 링크 하나 끊겨도 자동 전환이라 가용성은 높아진다.

### 4. OpenClaw AI 기능 설치

TOS 7의 차별점인 OpenClaw. 앱 센터에서 설치한다.

**앱 센터 → OpenClaw → 설치**

주요 기능:
- 사진 얼굴 인식 및 자동 분류
- 문서 내용 AI 검색 (로컬 인덱싱)
- 로컬 LLM 연동 (내장 AI 어시스턴트)

솔직히 말하면 현시점에서 완성도는 낮다. 얼굴 인식 속도는 [Immich]({% post_url 2026-06-16-10-00-00-148140-immich-google-photos-alternative %})에 비해 느리고 정확도도 애매하다. 로컬 LLM은 N350으로 돌리기엔 버거워서 응답이 답답하게 느리다.

"세계 최초 AI 내장 NAS"라는 마케팅 문구와 실제 사이 간극이 크다. 6개월 후에 다시 보는 게 맞을 것 같다.

## 삽질했던 부분

### Security Advisor 버그

보안 설정에서 SPC(Storage Pool Check)를 활성화하고 재부팅하면, Security Advisor가 계속 "SPC가 비활성화되어 있습니다"라고 뜬다. 실제로는 활성화되어 있는데 UI에서만 잘못 표시하는 것이다.

TOS 7이 작년 12월부터 베타인데 이 버그가 아직 남아있다. 무시해도 실제 기능은 정상 동작하지만, 처음 보면 뭔가 잘못된 줄 알고 삽질하게 된다.

### M.2 슬롯 호환성

NVMe SSD 3개 다 꽂았더니 2개만 인식됐다. 슬롯마다 지원 규격이 달라서 특정 드라이브가 안 잡히는 경우가 있다. TerraMaster 호환성 목록에서 미리 확인하는 게 필수다. 아무거나 꽂는 건 돈 낭비다.

![NAS 스토리지 풀 RAID 구성](https://images.unsplash.com/photo-1544197150-b99a580bb7a8?w=800&q=80)

## Synology와 비교하면

| 항목 | F4-425 Pro | Synology DS923+ |
|------|-----------|----------------|
| CPU | N350 8코어 | Ryzen R1600 2코어 |
| 기본 RAM | 16GB DDR5 | 4GB DDR4 |
| 5GbE | 듀얼 (기본 탑재) | 별도 확장 카드 필요 |
| M.2 슬롯 | 3개 | 2개 |
| 소프트웨어 완성도 | TOS 7 베타 수준 | DSM 7.2 — 안정적 |
| 앱 생태계 | 제한적 | 풍부 |

하드웨어는 F4-425 Pro가 압도적이다. 소프트웨어는 Synology [DSM 7.2]({% post_url 2026-06-05-10-00-00-012345-synology-dsm-initial-setup-security %})가 훨씬 안정적이고 앱 생태계도 넓다.

"안정성과 생태계가 중요하다" → Synology  
"성능과 가성비가 우선, 베타 소프트웨어 감수 가능하다" → F4-425 Pro

## 한 줄 정리

TOS 7이 정식 출시되면 진짜 경쟁자가 될 것 같은데, 지금 베타 상태로 구매하면 버그와 씨름하는 시간을 감수해야 한다.

---

다음엔 Docker Compose로 자체 호스팅 서비스 여러 개를 한 번에 관리하는 방법을 다룬다.

**참고 링크**
- [TerraMaster F4-425 Pro NASCompares 리뷰](https://nascompares.com/news/terramaster-f4-425-pro-nas-review-tos-7-intel-n350-dual-5gbe/)
- [TechPowerUp F4-425 Pro 리뷰](https://www.techpowerup.com/review/terramaster-f4-425-pro/)
- [Neowin — F4-425 Pro AI NAS 리뷰](https://www.neowin.net/reviews/terramaster-f4-425-pro-review-an-octa-core-intel-nas-that-ships-with-ai-openclaw/)

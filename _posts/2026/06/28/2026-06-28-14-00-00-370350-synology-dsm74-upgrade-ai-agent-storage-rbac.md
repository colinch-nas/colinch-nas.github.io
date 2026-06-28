---
layout: post
title: "Synology DSM 7.4 업그레이드 — DSM Agent AI·HDD 중복제거·RBAC 신기능 정리"
description: "2026년 6월 16일 출시된 Synology DSM 7.4의 주요 신기능을 정리했다. DSM Agent AI 어시스턴트, HDD 볼륨 중복제거·압축, RBAC 역할 기반 권한 관리까지 — 업그레이드 전 호환성 체크 포함."
date: 2026-06-28
tags: [Synology, DSM, NAS설정, 홈서버구축]
comments: true
share: true
---

# Synology DSM 7.4 업그레이드 — DSM Agent AI·HDD 중복제거·RBAC 신기능 정리

![Synology NAS 서버 데이터 관리 이미지](https://images.unsplash.com/photo-1558494949299-3ad8a4f4c2ed?w=800&q=80)

DSM 7.4가 2026년 6월 16일 공식 출시됐다. 처음엔 그냥 마이너 업데이트겠지 싶었는데, 릴리즈 노트를 읽어보니 생각보다 내용이 많았다. AI 관리 어시스턴트, HDD용 중복제거·압축, RBAC 역할 기반 권한 체계 — 단순 버그픽스 수준이 아니라 운영 방식에 영향을 주는 변경들이다.

## 환경 정보

| 항목 | 값 |
|------|---|
| NAS 모델 | Synology DS923+ |
| 기존 버전 | DSM 7.2.2-72806 Update 7 |
| 업그레이드 대상 | DSM 7.4 |
| 확인 날짜 | 2026-06-28 |

## 업그레이드 전 — 호환성 먼저 확인

DSM 7.4는 **21시리즈 이전 모델과 호환되지 않는다.** DS218+, DS918+ 같은 구형 모델을 쓰고 있다면 이번 업그레이드는 불가능하다. DS923+는 23시리즈라 문제없지만, 몇 년 된 NAS를 쓰는 경우 [Synology 릴리즈 노트 호환 목록](https://www.synology.com/en-us/releaseNote/DSM)을 반드시 먼저 확인하자.

업그레이드 방법 자체는 간단하다.

```
제어판 → 업데이트 및 복구 → DSM 업데이트 → 지금 업데이트
```

그런데 솔직히 업그레이드 전에 Hyper Backup으로 전체 스냅샷 하나 찍어두는 걸 강하게 권한다. DSM 7.1 → 7.2 때 Docker 패키지가 일시 충돌난 경험이 있어서, 이번엔 백업부터 했다. 복구할 일이 없으면 다행이고, 있으면 정말 다행인 게 백업이다.

## DSM Agent — AI가 NAS를 관리해준다 (Q3 2026 출시 예정)

이번 업데이트의 핵심 기능으로 발표된 건데, 자연어로 NAS에 명령을 내리면 설정을 찾아주거나 작업을 수행해준다는 개념이다.

예를 들어 "Docker 컨테이너가 왜 계속 재시작되는지 확인해줘"라고 입력하면 로그를 분석해 원인을 알려주는 식이다. 향후 DSM Agent 2.0에서는 헬스 모니터링, 백업 검증, 태스크 자동화까지 확장된다고 한다.

![Synology DSM Agent AI 관리 인터페이스 예시](https://images.unsplash.com/photo-1518770660439-4636190af475?w=800&q=80)

**단, 현재 DSM 7.4를 설치해도 DSM Agent는 아직 활성화되지 않는다.** Q3 2026 중 별도 업데이트로 제공될 예정이다. 릴리즈와 함께 쓸 수 있는 기능이 아니라는 점은 솔직히 좀 아쉽다. 기능 발표 타이밍과 실제 출시 타이밍이 따로 간다.

## 스토리지 효율화 — HDD에도 중복제거·압축 적용 가능

이게 지금 당장 쓸 수 있는 실질적인 신기능이다. 

DSM 7.3까지는 중복제거(Deduplication)와 압축(Compression)이 SSD 볼륨에서만 작동했다. 7.4부터는 HDD 볼륨에서도 두 기능을 동시에 적용할 수 있다.

```
스토리지 관리자 → 볼륨 선택 → 스토리지 효율성 → 중복제거 + 압축 활성화
```

주의사항이 있다. HDD에서 중복제거를 켜면 쓰기 오버헤드가 생겨서 속도가 약간 느려진다. 영상·음악 같은 미디어 파일이 대부분인 볼륨에는 효과가 거의 없고, 문서·코드·백업 파일이 많은 볼륨에서 공간 절약 효과가 크다.

DS923+에서는 미디어 볼륨엔 끄고 백업 전용 볼륨에만 활성화했다. Hyper Backup 대상 폴더에 중복 파일이 쌓이는 경우라면 이 설정이 공간을 꽤 아껴준다.

## RBAC — 역할 기반 권한 관리

NAS를 여러 명이 함께 쓰거나, 소규모 팀 환경에서 계정 관리를 할 때 유용한 기능이다.

기존 DSM은 사용자별로 공유 폴더 권한을 일일이 설정해야 했다. RBAC(Role-Based Access Control)는 역할을 먼저 정의하고 사용자에게 역할을 할당하는 방식이다. "미디어 읽기 전용", "백업 관리자" 같은 역할을 만들어두면 사람이 바뀌어도 역할만 재할당하면 된다.

```
제어판 → 사용자 및 그룹 → 역할 관리 → 역할 추가
```

혼자 쓰는 NAS라면 당장 필요한 기능은 아니다. 가족 공유 NAS나 팀 환경에서는 이전보다 훨씬 체계적으로 관리할 수 있다.

## 로그 센터 개편

운영 로그와 애플리케이션 로그가 한 화면으로 통합됐다. 이전에는 DSM 시스템 로그와 앱별 로그를 각각 다른 곳에서 봐야 했는데, 이제 통합된 필터로 한 번에 검색할 수 있다.

Graylog, Splunk 같은 외부 관측 플랫폼으로 로그를 내보내는 Syslog/Webhook 연동도 추가됐다.

```
로그 센터 → 내보내기 → Syslog 서버 주소 설정
```

이건 실제로 써보니 편하다. Docker 컨테이너 로그와 DSM 시스템 로그를 같이 보고 싶을 때 화면 전환이 줄어들었다.

## 삽질했던 부분

업그레이드 직후 Container Manager(Docker)가 패키지 업데이트를 요청했다. 업데이트했더니 실행 중이던 컨테이너들이 자동 재시작됐다. 설정이 날아가진 않았지만 재생 중이던 Jellyfin 미디어가 끊겼다. **저녁에 업그레이드하는 걸 권한다.**

포럼 글들을 보니 업그레이드 후 DNS 설정이 초기화되는 케이스가 몇 건 보였다. 외부 접속을 설정해둔 경우엔 업그레이드 후 제어판 → 네트워크 → DNS에서 한 번 확인해볼 것. ([DDNS와 Let's Encrypt 설정 방법]({% post_url 2026-06-08-10-00-00-049380-synology-ddns-letsencrypt-external-access %}) 참고)

## 한 줄 정리

DSM 7.4는 AI 방향으로의 전환을 선언한 업데이트다. 당장 체감할 수 있는 건 HDD 스토리지 효율화와 로그 센터 개편이고, DSM Agent는 Q3 2026을 기다려야 한다.

---

다음엔 Frigate NVR을 Docker로 설치해서 홈캠에 AI 객체 감지를 붙이는 방법을 다룬다.

**참고 링크**
- [Synology DSM 7.4 공식 릴리즈 노트](https://www.synology.com/en-us/releaseNote/DSM)
- [Synology DSM 7.4 공식 발표 블로그](https://www.synology.com/en-global/company/news/article/dsm74/Synology%20releases%20DiskStation%20Manager%207.4,%20bringing%20AI-guided%20management,%20smarter%20collaboration,%20and%20higher%20storage%20efficiency)
- [DSM 7.4 분석 — NAS Compares](https://nascompares.com/2026/06/10/synology-dsm-7-4-the-good-the-bad-and-the-ai-dsm-8-0-lite/)

---
layout: post
title: "QNAP myQNAPcloud One 백업 설정 - HBS 3로 NAS 외부 복사본 만들기"
description: "QNAP이 2026년 공식 출시한 myQNAPcloud One을 HBS 3와 연결해 NAS 데이터를 외부에 백업하는 설정 순서와 복구 테스트 방법을 정리한다."
date: 2026-08-18
tags: [QNAP, NAS보안, 백업전략, 자체호스팅, 홈서버]
comments: true
share: true
---

![QNAP NAS와 외부 백업 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

NAS 안에만 파일을 보관하면 디스크 고장보다 관리자 계정 탈취가 더 곤란하다. QNAP이 2026년 공식 출시한 **myQNAPcloud One**은 NAS 백업과 확장형 오브젝트 스토리지를 묶은 서비스다. 여기서는 QTS 5.2 계열의 HBS 3(Hybrid Backup Sync)에서 문서 폴더를 외부 복사본으로 보내고, 실제 복구까지 확인한다.

## 내가 적용한 조건

| 항목 | 기준 |
|---|---|
| NAS | QNAP TS-464 |
| OS | QTS 5.2.x |
| 백업 도구 | HBS 3 최신 버전 |
| 대상 | `documents` 공유 폴더 |
| 방식 | 주 1회 버전 백업, NAS와 다른 위치에 저장 |

myQNAPcloud One 계정과 저장 공간을 먼저 준비한다. 무료 용량이나 요금은 시점과 지역에 따라 달라질 수 있어, 가입 화면의 현재 조건을 기준으로 잡는 편이 맞다. NAS가 인터넷에 직접 노출될 필요는 없지만 DNS와 HTTPS로 QNAP 계정에 연결돼야 한다.

## HBS 3에 외부 저장소 추가

패키지 센터에서 HBS 3를 열고 `Storage Spaces → Create → myQNAPcloud Storage`를 선택한다. QNAP ID 로그인이 뜨면 NAS 관리자의 비밀번호가 아니라 myQNAPcloud 계정으로 인증한다. 저장 공간이 보이지 않으면 QNAP ID의 지역과 NAS 계정 지역이 다른지부터 확인했다.

백업 작업은 `Backup & Restore → Create → New backup job` 순서다. 소스는 `documents` 하나만 선택하고, 대상은 방금 만든 myQNAPcloud Storage로 지정한다. 사진 전체를 한 번에 올리면 첫 작업이 오래 걸리므로 문서·설정 파일처럼 복구 우선순위가 높은 폴더부터 시작하는 게 낫다.

```text
일정: 매주 일요일 03:00
버전 관리: 활성화
보관 버전: 12개
암호화: 활성화
클라이언트 측 암호: 별도 보관
알림: 작업 실패 시 이메일
```

클라이언트 측 암호를 켜면 클라우드 운영자도 백업 내용을 바로 읽기 어렵다. 대신 암호를 잃으면 복구도 안 된다. 비밀번호 관리 앱과 오프라인 메모 두 곳에 보관하고, NAS의 공유 폴더에는 평문으로 저장하지 않았다.

## 복구 테스트를 빼먹으면 안 된다

백업 완료 표시만 믿지 말고 HBS 3의 `Backup & Restore → Restore`에서 `documents` 아래 테스트 파일 하나를 별도 폴더로 복구한다. 파일 이름, 한글 파일명, 수정 날짜가 유지되는지 확인한다. 처음 테스트에서 공유 폴더 권한이 달라 파일은 복구됐지만 가족 계정에서는 보이지 않는 일이 있었다.

| 점검 항목 | 통과 기준 |
|---|---|
| 작업 기록 | 실패·건너뜀 0건 |
| 복구 파일 | 열림·한글명·날짜 정상 |
| 암호화 키 | NAS 밖에도 보관 |
| 계정 보안 | QNAP ID 2단계 인증 활성화 |

myQNAPcloud One은 NAS 내부 스냅샷이나 RAID를 대신하지 않는다. 스냅샷은 빠른 실수 복구용이고, 외부 백업은 NAS 자체가 잠겼을 때 필요한 복사본이다. 또한 SMB 포트를 인터넷에 열어 백업을 해결하려 하지 말고, 관리 화면에는 VPN이나 허용된 HTTPS 경로만 사용한다.

QNAP 공식 발표 기준 myQNAPcloud One은 NAS 백업과 오브젝트 스토리지를 제공한다. 실제 메뉴 이름과 요금은 QTS·지역별로 달라질 수 있으므로, 적용 전 [QNAP 2026 뉴스룸](https://www.qnap.com/en-us/news/2026)과 HBS 3의 현재 도움말을 함께 확인하는 게 안전하다.

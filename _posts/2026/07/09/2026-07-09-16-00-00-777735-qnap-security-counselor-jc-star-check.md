---
layout: post
title: "QNAP Security Counselor - JC-STAR 소식 뒤에 홈 NAS에서 바로 점검할 것"
description: "2026년 7월 QNAP JC-STAR Level 1 보안 검증 소식을 기준으로 QTS 5.2 홈 NAS에서 Security Counselor와 Security Center를 점검하는 순서다."
date: 2026-07-09
tags: [QNAP, NAS보안, QTS, 홈서버]
comments: true
share: true
---

![QNAP NAS 보안 점검 화면을 확인하는 작업 환경](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)
이 그림에서는 NAS 자체보다 관리자 화면, 계정, 외부 접속 경로가 같이 보안 대상이라는 점을 보면 된다.

QNAP NAS를 QTS 5.2 계열로 쓰고 있다면 오늘 할 일은 새 기능 구경이 아니라 `Security Counselor` 결과를 기준선으로 고정하는 것이다. QNAP은 2026년 7월 7일 NAS가 일본 `JC-STAR Level 1` 보안 검증을 받았다고 발표했다. 이건 제품 보안 기준을 설명하는 소식이지, 내 집 TS-464 설정이 자동으로 안전해졌다는 뜻은 아니다.

내 기준 환경은 QNAP TS-464, QTS 5.2, 공유기 뒤 사설 IP, 외부 접속은 Tailscale VPN만 허용한 구성이다. 예전에 myQNAPcloud와 포트포워딩을 같이 켜 둔 적이 있는데, Security Counselor가 "중간 위험"을 띄우기 전까지 잊고 있었다. 보안 인증 뉴스는 읽고 끝내면 의미가 없고, 내 장비에서 빨간 항목을 줄이는 데 써야 한다.

## 점검 순서

| 위치 | 확인할 것 | 내가 둔 기준 |
|---|---|---|
| Security Counselor | 보안 정책과 스캔 결과 | Intermediate 이상 |
| Security Center | 비정상 파일 활동 감시 | 켜기 |
| 권한 | admin 비활성화, 2단계 인증 | 관리자 계정만 필수 |
| 네트워크 | UPnP, 포트포워딩 | 꺼짐 |
| 알림 | 보안 권고 구독 | 이메일 또는 푸시 |

QTS에서 `Security Counselor`를 열고 보안 정책을 고른다. 처음부터 Advanced로 두면 경고가 너무 많아져서 실제로 고칠 항목을 놓치기 쉽다. 가족용 NAS라면 Intermediate로 시작하고, 외부 공개 서비스가 하나라도 있으면 Advanced까지 올려본다.

스캔이 끝나면 경고를 한 번에 고치지 않는다. 나는 아래 세 항목만 바로 처리했다.

- 기본 `admin` 계정 비활성화
- 약한 비밀번호 계정 변경
- 공유기 UPnP 자동 포트 매핑 해제

QTS 5.2의 `Security Center`는 파일 활동을 감시해 랜섬웨어처럼 대량 변경이 생길 때 보호 조치를 취하는 기능이다. 공식 설명처럼 백업을 대체하는 기능은 아니고, 이상 행동을 빨리 알아차리는 센서에 가깝다. 사진 원본, 문서, 백업 폴더처럼 변경 패턴이 비교적 일정한 공유 폴더부터 감시 대상으로 잡는다.

## 외부 접속은 따로 본다

여기서 헷갈렸던 건 Security Counselor 점수가 좋아도 공유기 설정은 따로 봐야 한다는 점이다. QNAP 안에서 UPnP를 꺼도 예전에 공유기에 수동으로 만든 포트포워딩 규칙은 남아 있을 수 있다. 공유기 관리자 화면에서 8080, 443, 5000, 5001 같은 NAS 관리 포트가 외부로 열렸는지 확인한다.

외부 접속이 필요하면 관리 UI를 공개하지 말고 VPN을 쓴다. 가족 사진 공유나 Jellyfin처럼 특정 서비스만 열어야 할 때도 관리자 계정과 같은 포트로 묶지 않는다. 모바일 앱에서 NAS를 만지는 계정은 관리자 권한을 빼고, 파일 접근 권한만 남기는 쪽이 사고가 작다.

## 보안 점검 기록 남기기

스캔 결과는 고친 뒤 사라지니 날짜를 남겨 둔다. NAS 안의 메모 앱보다 로컬 PC나 비밀번호 관리자 보안 메모에 적는 편이 낫다.

```text
2026-07-09 QNAP TS-464 QTS 5.2 보안 점검
- Security Counselor: Intermediate 통과
- Security Center: photos, documents 공유 폴더 감시
- UPnP: 꺼짐
- 포트포워딩: 없음
- 관리자 2FA: 켜짐
```

주의할 점은 두 가지다. 보안 점수가 100점이어도 백업이 없으면 랜섬웨어 복구는 어렵다. 또 Security Center가 파일 활동을 감시하더라도 동기화 앱이 대량으로 파일명을 바꾸는 날에는 오탐이 날 수 있다. 나는 사진 정리 작업을 하는 날에는 감시 알림을 더 자주 보고, 작업이 끝나면 스냅샷이 정상 생성됐는지 확인한다.

짧게 정리하면 이렇다. QNAP의 JC-STAR 소식은 홈 NAS에서도 보안 기준을 다시 잡는 계기로 쓰면 된다. Security Counselor로 계정과 포트를 줄이고, Security Center로 파일 활동 감시를 켜고, 공유기 포트포워딩까지 따로 확인한다. 인증 뉴스보다 내 장비의 열린 문을 줄이는 일이 더 급하다.

출처: [QNAP JC-STAR Level 1 보안 검증 발표](https://www.qnap.com/en-us/news/2026/qnap-nas-earns-japans-jc-star-level-1-security-validation-setting-a-new-benchmark-for-enterprise-and-government-procurement), [QNAP QTS 5.2 NAS 보안 문서](https://docs.qnap.com/operating-system/qts/5.2.x/en-us/securing-the-nas-5EAA5490.html)

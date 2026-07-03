---
layout: post
title: "Synology MailPlus Server 긴급 업데이트 - 4.0.1 버전 확인과 외부 노출 줄이기"
description: "Synology-SA-26:11 MailPlus Server Critical 권고를 기준으로 DSM 7.2.2와 7.3에서 패키지 버전, 포트 공개, 메일 로그를 점검하는 순서다."
date: 2026-07-04
tags: [Synology, DSM, NAS보안, MailPlus, NAS설정]
comments: true
share: true
---

![NAS 메일 서버와 보안 로그를 확인하는 작업 화면](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)
이 그림에서 볼 건 메일 서버가 NAS 안의 앱 하나가 아니라 인터넷에 직접 닿는 입구라는 점이다.

Synology MailPlus Server를 설치해 둔 NAS라면 2026년 7월 4일 기준으로 패키지 버전부터 확인해야 한다. Synology-SA-26:11은 2026년 6월 26일 공개된 Critical 권고이고, DSM 7.3은 `4.0.1-31663` 이상, DSM 7.2.2와 7.2.1은 `4.0.1-21663` 이상이 기준이다. 내가 헷갈렸던 건 DSM 업데이트와 패키지 업데이트가 따로 움직인다는 점이었다. DSM이 최신이어도 MailPlus Server가 낮으면 위험이 남는다.

내 기준 환경은 Synology DS923+, DSM 7.2.2-72806, MailPlus Server, 공유기 포트 포워딩 25/465/587/993이다. 개인 도메인으로 메일을 직접 받는 구성이라면 SMTP 포트를 완전히 닫을 수는 없다. 대신 버전, 포트, 로그를 나눠서 본다.

| 확인 항목 | DSM 7.3 | DSM 7.2.2/7.2.1 |
|---|---:|---:|
| 최소 MailPlus Server | 4.0.1-31663 | 4.0.1-21663 |
| 권고 등급 | Critical | Critical |
| 우회책 | 없음 | 없음 |
| 우선 조치 | 패키지 업데이트 | 패키지 업데이트 |

## 버전 확인

DSM에서 **패키지 센터 -> 설치됨 -> MailPlus Server**로 들어가 버전을 확인한다. 업데이트 버튼이 보이면 DSM 재부팅보다 패키지 업데이트가 우선이다. 메일 수신이 끊기면 곤란한 환경이면 10분 정도 점검 시간을 잡는다.

SSH를 켜둔 NAS라면 패키지 목록도 같이 확인한다.

```bash
synopkg list | grep -i MailPlus
```

여기서 버전이 기준보다 낮으면 업데이트부터 한다. Synology 권고에 별도 완화책이 없다고 나와 있어서 방화벽 규칙만 만지는 건 임시 대응에 가깝다.

## 포트 줄이기

테스트로 설치만 해둔 NAS라면 **패키지 센터 -> MailPlus Server -> 중지** 후 제거까지 한다. 실제 운영 중이면 공유기 포트 포워딩을 확인한다.

| 포트 | 용도 | 노출 판단 |
|---:|---|---|
| 25 | 서버 간 SMTP 수신 | 메일 수신 시 필요 |
| 465/587 | SMTP 제출 | 사용자 인증 필요 |
| 993 | IMAPS | 외부 메일 앱 사용 시 필요 |
| 5000/5001 | DSM 관리 | 외부 공개 비추천 |

여기서 5000/5001까지 같이 열려 있으면 닫는다. 관리 화면은 Tailscale VPN이나 내부망에서만 접근하는 쪽이 낫다. 나는 예전에 메일 앱 테스트하다가 5001까지 열어둔 걸 Security Advisor에서 보고 알았다.

## 로그 확인

업데이트 후에는 **MailPlus Server -> 로그**에서 로그인 실패, SMTP 인증 실패, 낯선 국가 IP를 본다. 정상 메일 서버도 스팸 시도는 계속 들어온다. 업데이트 직후 특정 계정으로 반복되면 비밀번호와 2단계 인증을 같이 손본다.

짧게 정리하면 이렇다. MailPlus Server를 쓰면 DSM 버전이 아니라 패키지 버전을 확인한다. 쓰지 않으면 중지하거나 제거한다. 운영 중이면 필요한 메일 포트만 남기고 DSM 관리 포트는 닫는다. 2026년 6월 26일 권고는 우회책이 없으니 "나중에 DSM 업데이트할 때 같이"로 미루면 안 된다.

출처: [Synology-SA-26:11 MailPlus Server 공식 권고](https://www.synology.com/security/advisory/Synology_SA_26_11)

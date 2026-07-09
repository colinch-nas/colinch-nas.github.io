---
layout: post
title: "QNAP Dirty Frag 점검 - Container Station과 SSH 권한부터 줄이기"
description: "2026년 7월 10일 QNAP QSA-26-17 Dirty Frag 권고 기준으로 QTS 5.2 홈 NAS에서 펌웨어, 컨테이너, SSH 권한을 점검한다."
date: 2026-07-10
tags: [QNAP, NAS보안, Docker, VPN]
comments: true
share: true
---

![QNAP NAS 보안 업데이트와 컨테이너 권한 점검](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)

이 그림에서는 NAS 펌웨어보다 컨테이너, 계정, 외부 접속 경로가 같이 공격면이 된다는 점을 보면 된다.

QNAP NAS를 QTS 5.2 계열로 쓰고 있다면 2026년 7월 10일 기준 `QSA-26-17 Dirty Frag` 권고를 펌웨어 업데이트 한 줄로만 넘기면 안 된다. QNAP은 CVE-2026-43284가 x86 기반 NAS, ARM64 기반 NAS, QuTS hero, QuTScloud에 영향을 준다고 공지했고, 패치된 최신 펌웨어 설치를 권고한다. 내 기준 환경은 QNAP TS-464, QTS 5.2, Container Station, 공유기 뒤 사설 IP, 외부 접속은 VPN만 허용한 구성이다.

헷갈렸던 건 이 취약점이 원격에서 바로 관리자 화면을 뚫는 종류가 아니라는 점이다. 인증된 로컬 일반 사용자가 루트 권한으로 올라갈 수 있는 취약점이다. 그래서 "우리 집 NAS는 가족만 로그인한다"로 끝내면 안 된다. NAS 안에서 도는 컨테이너, SSH 계정, 오래된 앱이 로컬 사용자처럼 발판이 될 수 있다.

| 점검 항목 | 볼 위치 | 내가 잡은 기준 |
|---|---|---|
| QTS 펌웨어 | 제어판 -> 펌웨어 업데이트 | 최신 버전 적용 |
| SSH/Telnet | 제어판 -> 네트워크 및 파일 서비스 | Telnet 끔, SSH 관리자만 |
| Container Station | 컨테이너 설정 | Privileged 모드 금지 |
| 외부 접속 | 공유기, myQNAPcloud, QuFirewall | 관리 UI 직접 공개 안 함 |

## 펌웨어부터 확인한다

QTS에서는 **제어판 -> 시스템 -> 펌웨어 업데이트**로 들어가 업데이트를 확인한다. 자동 확인이 늦게 뜨면 QNAP 다운로드 센터에서 모델명 `TS-464`를 직접 고른다. 업데이트 전에는 HBS 3 백업 작업이 최근 성공했는지 보고, 사진 원본이나 문서 폴더는 최소 한 번 외장 디스크나 다른 NAS에 복사해 둔다.

SSH를 잠깐 켜야 한다면 현재 커널과 아키텍처만 확인하고 다시 끈다.

```bash
uname -m
uname -r
```

여기서 x86_64나 aarch64 계열이면 권고의 영향 범위에 들어간다고 보고 패치 여부를 확인하는 게 맞다. 나는 처음에 "컨테이너만 돌리니까 괜찮겠지"라고 생각했는데, 오히려 컨테이너가 많을수록 권한 경계를 더 봐야 했다.

## 컨테이너 권한을 줄인다

Container Station에서 각 컨테이너의 고급 설정을 열어 `Privileged mode`가 켜져 있는지 본다. Jellyfin, Uptime Kuma, Vaultwarden 같은 일반 서비스는 보통 특권 모드가 필요 없다. USB 장치나 네트워크 장치를 직접 만지는 컨테이너만 예외로 두고, 그 경우에도 이미지를 공식 저장소나 검증된 저장소로 제한한다.

컨테이너 점검 순서는 이렇게 잡았다.

1. 쓰지 않는 컨테이너 중지 후 삭제
2. `Privileged` 켜진 컨테이너 목록 작성
3. 외부 포트가 열린 컨테이너만 따로 표시
4. 이미지 업데이트 전 compose 또는 설정 백업

## SSH와 외부 접속을 분리한다

QNAP 권고도 SSH/Telnet 권한 회수, 신뢰한 컨테이너만 사용, Web Server 같은 미사용 서비스 비활성화, VPN이나 QuFirewall을 통한 네트워크 제한을 말한다. 홈 NAS에서는 이 네 가지가 제일 현실적이다. 특히 Telnet은 켤 이유가 거의 없다. SSH도 평소에는 끄고, 필요할 때 관리자 계정으로만 잠깐 여는 편이 낫다.

외부에서 NAS를 만져야 한다면 관리자 UI를 포트포워딩하지 않는다. VPN으로 내부망에 들어온 뒤 QTS에 접속하는 구조가 관리하기 쉽다. myQNAPcloud를 쓰더라도 공개 범위를 파일 공유와 관리자 작업으로 섞지 않는다. 여기서 섞이면 모바일 앱 하나가 NAS 전체 권한처럼 동작한다.

주의할 점은 업데이트 후 "재부팅됐으니 끝"이 아니라는 것이다. Container Station 컨테이너가 자동으로 다시 떴는지, 외부 포트가 예상한 것만 열렸는지, QuFirewall 규칙이 유지됐는지 확인해야 한다. 나는 예전에 펌웨어 업데이트 뒤 테스트용 포트가 살아 있는 걸 뒤늦게 봤다. 보안 패치를 올렸는데 공격면은 그대로 둔 셈이었다.

짧게 정리하면 이렇다. QSA-26-17은 최신 펌웨어 적용이 출발점이다. 그 뒤 SSH/Telnet 권한을 줄이고, Container Station의 특권 모드를 걷어내고, QTS 관리자 화면은 VPN 뒤에 둔다. Dirty Frag 같은 로컬 권한 상승 취약점은 "누가 NAS 안에서 코드를 실행할 수 있는가"를 줄이는 쪽이 핵심이다.

출처: [QNAP QSA-26-17 Dirty Frag 보안 권고](https://www.qnap.com/en/security-advisory/qsa-26-17), [QNAP Security Advisories](https://www.qnap.com/en-us/security-advisories)

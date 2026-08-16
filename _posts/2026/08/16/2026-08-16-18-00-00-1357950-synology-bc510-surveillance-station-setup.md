---
layout: post
title: "Synology BC510·TC510 NAS 연결 설정 - Surveillance Station 녹화와 감지 알림"
description: "2026년 출시된 Synology BC510·TC510을 DSM 7.2 NAS와 Surveillance Station에 연결하는 방법이다. PoE 카메라 등록, 녹화 일정, 사람·차량 감지와 외부 접속 보안까지 정리한다."
date: 2026-08-16
tags: [Synology, NAS설정, 홈서버, DSM, NAS보안]
comments: true
share: true
---

![Synology BC510 TC510 NAS 보안 카메라 설정](https://images.unsplash.com/photo-1557597774-9d273605dfa9?w=1200&q=80)

Synology가 2026년 5월 공개한 BC510·TC510은 5MP(2880×1620) PoE 카메라다. DSM 7.2 NAS에 Surveillance Station을 설치하면 별도 NVR 없이 녹화하고, 사람·차량 감지 이벤트를 검색할 수 있다. 직접 연결해보니 카메라보다 중요한 건 고정 IP와 저장 공간, 그리고 외부 공개를 막는 설정이었다.

## BC510과 TC510 중 어떤 모델인가

두 모델의 영상 사양은 거의 같고 설치 형태가 다르다. BC510은 불릿형, TC510은 터렛형이라 현관 처마에는 BC510, 천장이나 벽면의 사각지대를 줄일 곳에는 TC510이 맞다. 둘 다 PoE(Power over Ethernet, 랜선 하나로 전원과 데이터를 함께 보내는 방식)를 사용하고 야간 시야는 최대 30m다.

| 항목 | BC510 | TC510 |
|---|---|---|
| 해상도 | 5MP, 2880×1620 | 5MP, 2880×1620 |
| 연결 | PoE | PoE |
| 보호 등급 | IP67/IP66 | IP67/IP66/IK10 |
| 추천 위치 | 벽면·마당 | 천장·현관 |

공식 사양 기준으로 사람·차량 감지, 침입·오디오·카메라 훼손 감지가 지원된다. 다만 Surveillance Station에서는 카메라 라이선스 상태를 별도로 확인해야 한다.

## NAS에 카메라 연결하기

테스트 환경은 DS925+와 DSM 7.2.x, Surveillance Station 9.x다. 카메라를 PoE 스위치에 연결하고 NAS와 같은 내부망에 둔다. 공유기의 DHCP 임대 목록에서 카메라 IP를 확인한 뒤, 공유기에서 해당 MAC 주소에 예약 IP를 지정했다. 카메라 IP가 바뀌면 녹화가 끊기는 일이 실제로 생긴다.

Surveillance Station에서 다음 순서로 등록한다.

1. **패키지 센터 → Surveillance Station** 설치
2. **IP 카메라 → 추가 → 카메라 추가** 선택
3. 제조사에서 Synology, 모델에서 BC510 또는 TC510 선택
4. 예약해 둔 IP, 카메라 관리자 계정과 비밀번호 입력
5. 연결 테스트 후 비디오 형식은 H.265, 스트림은 5MP로 저장

처음에는 H.265가 무조건 낫다고 생각했지만 오래된 모바일 앱이나 일부 브라우저에서 재생 호환성이 떨어졌다. 녹화는 H.265로 두고, 실시간 미리보기용 보조 스트림은 H.264 1280×720으로 설정하는 편이 안정적이었다.

## 녹화 일정과 이벤트 알림

**녹화 설정 → 녹화 일정**에서 상시 녹화와 이벤트 녹화를 나눌 수 있다. 현관은 상시 녹화, 실내는 움직임 이벤트만 저장하는 방식이 저장 공간을 덜 쓴다. 이벤트 설정에서 사람·차량 감지를 켜고, 현관 앞 도로까지 감지 영역에 넣지 않도록 마스크를 조정한다.

| 용도 | 설정 | 이유 |
|---|---|---|
| 현관 | 상시 녹화 + 사람 감지 | 짧은 움직임을 놓치지 않음 |
| 주차장 | 이벤트 녹화 + 차량 감지 | 저장 공간 절약 |
| 실내 | 움직임 이벤트만 | 불필요한 영상 감소 |

저장 경로는 전용 공유 폴더로 만들고 일반 가족 계정에는 접근 권한을 주지 않았다. 녹화 파일을 NAS 전체 백업 작업에 포함하면 백업 시간이 급격히 늘어나므로, 보존 기간을 14일로 정한 뒤 중요한 이벤트만 별도 보관하는 방식이 현실적이다.

## 외부 접속에서 삽질한 부분

카메라와 Surveillance Station을 인터넷에 직접 노출할 필요는 없다. 공유기에서 NAS의 5000, 5001 포트를 열어두는 대신 Tailscale VPN으로 NAS와 휴대폰을 같은 사설망처럼 묶었다. 외부에서 DS cam을 사용할 때도 VPN을 먼저 켜고 내부 주소로 접속한다.

관리자 계정은 카메라와 DSM에서 서로 다른 비밀번호를 쓰고, 카메라의 기본 계정은 등록 후 비활성화했다. 카메라 자체의 RTSP 포트를 인터넷으로 포워딩하는 설정은 하지 않았다. IP 카메라가 보안 장비인 만큼 영상보다 계정과 열린 포트를 먼저 줄여야 한다.

BC510·TC510은 최신 Synology 카메라를 NAS에 바로 묶고 싶은 경우 선택지가 된다. 다만 구매 전에 PoE 스위치, Surveillance Station 라이선스, 14일 이상 녹화할 저장 공간을 함께 계산해야 한다.

- [Synology BC510 공식 사양](https://www.synology.com/en-global/products/BC510)
- [Synology TC510 공식 사양](https://www.synology.com/en-ca/products/TC510)
- [Synology BC510·TC510 출시 안내](https://www.synology.com/en-ca/company/news/article/bc510_tc510/Synology%20Introduces%20BC510%20and%20TC510%2C%20new%20versatile%20AI-enabled%20bullet%20and%20turret%20cameras%20for%20smart%20surveillance)

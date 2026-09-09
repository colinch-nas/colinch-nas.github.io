---
layout: post
title: "UGREEN NASync DXP6800 Ultra 초기 설정 - 10GbE 홈서버로 쓰는 순서"
description: "UGREEN NASync DXP6800 Ultra의 6베이·듀얼 10GbE 구성을 홈서버로 활용하는 초기 설정 순서와 RAID, SSD, Docker 운영 시 확인할 조건을 정리한다."
date: 2026-09-09
tags: [UGREEN, NAS, 홈서버, Docker, HomeLab]
comments: true
share: true
---

![UGREEN NASync DXP6800 Ultra 10GbE 홈서버 구성](/assets/images/2026-09-09-ugreen-dxp6800-ultra-10gbe-home-server.png)

UGREEN NASync DXP6800 Ultra는 6개 SATA 베이와 2개 M.2 슬롯, 듀얼 10GbE 포트를 갖춘 2026년형 고성능 NAS다. 공개된 제품 사양 기준 Intel Core 5 120U, 기본 8GB DDR5 메모리, 최대 96GB 확장이 가능하다. 구매했다면 앱만 설치하기보다 저장소와 네트워크를 고정한 뒤 Docker를 올리는 순서가 덜 꼬인다.

## 사양에서 실제로 봐야 할 부분

| 항목 | DXP6800 Ultra | 홈서버에서의 의미 |
|---|---|---|
| 드라이브 | 6×SATA + 2×M.2 | 데이터와 캐시를 나눌 수 있음 |
| 네트워크 | 10GbE 2개 | PC·스위치까지 10GbE여야 효과가 남 |
| 메모리 | DDR5 8GB, 최대 96GB | Docker 여러 개면 증설 여지가 있음 |
| 출력·확장 | Thunderbolt 4, HDMI 2.1 | 로컬 편집·모니터 연결에 활용 |

여기서 가장 많이 착각하는 부분은 NAS에 10GbE 포트가 두 개 있다고 파일 복사가 자동으로 2배가 되지는 않는다는 점이다. PC, 케이블, 스위치가 모두 10GbE를 지원해야 한다. 처음에는 10GbE 한 포트만 메인 스위치에 연결하고, 다른 포트는 장애 대응용으로 남겨두는 편이 관리하기 쉽다.

## 초기 설정 순서

전원을 켜기 전에 공유기에서 NAS에 DHCP 예약을 걸었다. 예시는 `192.168.10.20`이다. IP가 바뀌면 Docker 서비스 주소와 백업 작업이 같이 흔들린다.

1. 관리자 계정을 기본 `admin`과 다른 이름으로 만든다.
2. 자동 업데이트는 바로 적용하지 말고, 설정 백업 후 유지보수 시간에 진행하도록 바꾼다.
3. 6개 디스크는 용도에 따라 RAID 6 또는 RAID 5로 만든다. 중요한 자료가 많고 디스크 교체 시간이 길다면 RAID 6이 마음 편하다.
4. 공유 폴더를 `data`, `docker`, `backup`으로 분리한다.
5. M.2는 처음부터 캐시로 묶지 않는다. 실제 사용량을 확인한 뒤 읽기 캐시부터 검토한다.

RAID는 백업이 아니다. 디스크 고장에 대한 여유 공간일 뿐이라서, `backup` 공유 폴더는 다른 NAS나 USB 디스크에도 다시 복사해야 한다.

## 10GbE와 Docker 운영 기준

PC와 NAS 사이를 10GbE 스위치로 연결한 뒤 파일 하나가 아니라 20GB 이상 단일 파일로 속도를 측정한다. 작은 파일이 많은 폴더는 디스크 지연과 SMB 메타데이터 처리 때문에 10GbE 회선 속도에 도달하지 않는다.

Docker 컨테이너는 전용 `docker` 공유 폴더 아래에 서비스별 디렉터리를 만든다. 예를 들어 Jellyfin과 Uptime Kuma를 올릴 때도 설정 파일과 미디어를 섞지 않는다. 컨테이너를 지워도 설정을 보존할 수 있고, 백업 대상도 명확해진다.

| 목적 | 권장 연결 | 피할 설정 |
|---|---|---|
| 관리자 화면 | Tailscale VPN | DSM 관리 포트 직접 공개 |
| 가족용 웹 서비스 | HTTPS 역방향 프록시 | HTTP 포트 그대로 포워딩 |
| 대용량 파일 편집 | 10GbE 유선 | Wi-Fi 속도로 성능 판단 |
| 백업 | 별도 장치·클라우드 | 같은 RAID 안에 복사만 하기 |

## 구매 직후 체크리스트

- [ ] 최신 UGOS Pro 펌웨어와 호환 디스크 목록 확인
- [ ] 관리자 2단계 인증과 로그인 알림 활성화
- [ ] 공유기에서 NAS 관리자·SMB 포트 외부 공개 여부 확인
- [ ] RAID 스크럽과 디스크 상태 검사 예약
- [ ] Docker 데이터와 설정의 별도 백업 테스트

내가 확인한 공식 사양만 보면 DXP6800 Ultra는 단순 파일 저장용보다 10GbE 작업 공간과 자체 호스팅을 함께 운영할 때 장점이 뚜렷하다. 다만 네트워크 전체가 10GbE가 아니면 체감은 일반 2.5GbE NAS와 크게 다르지 않다. 저장소를 먼저 안정화하고, 그 뒤에 Docker와 미디어 서비스를 추가하는 순서가 가장 안전하다.

참고 문서: [UGREEN NASync DXP6800 Ultra 공식 사양](https://ai.ugreen.com/products/ugreen-nasync-dxp6800-ultra-nas-storage)

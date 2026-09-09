---
layout: post
title: "Home Assistant 2026.9 네트워크 저장소 설정 — Synology NAS에 백업·미디어 연결"
description: "Home Assistant 2026.9.1에서 Synology NAS의 SMB 공유를 네트워크 저장소로 연결하는 방법을 정리한다. 백업·미디어·권한 설정과 Docker Container의 제한까지 확인한다."
date: 2026-09-10
tags: [HomeAssistant, Synology, NAS설정, 백업전략, 자체호스팅]
comments: true
share: true
---

![Home Assistant와 NAS 네트워크 저장소 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

이 그림에서 봐야 할 부분은 Home Assistant가 단독 저장장치가 아니라 NAS 공유 폴더를 붙여 쓸 수 있다는 점이다.

Home Assistant 2026.9.1 기준으로 Synology NAS의 SMB 공유를 `네트워크 저장소`로 등록하면 백업을 NAS에 바로 저장하거나 미디어 폴더로 사용할 수 있다. 나는 Home Assistant OS를 Synology VMM에 올린 환경에서 이 구성을 적용했다. 다만 Home Assistant Container(Docker)에는 같은 메뉴가 없으므로, 이 글의 절차는 Home Assistant OS 또는 Supervisor 환경에 해당한다. [공식 OS 문서](https://www.home-assistant.io/common-tasks/os/)에도 NFS와 Samba/CIFS 연결은 OS의 저장소 메뉴에서 설정한다고 안내돼 있다.

## 준비한 환경

| 항목 | 설정값 |
|---|---|
| Home Assistant | 2026.9.1, Home Assistant OS |
| NAS | Synology DSM 7.4, 고정 IP `192.168.0.10` |
| 프로토콜 | SMB/CIFS 2.1 이상 |
| 공유 폴더 | `ha-backup` |
| 전용 계정 | `ha-storage` |

Synology에서 `ha-storage` 계정을 만들고 `ha-backup` 공유 폴더에 읽기·쓰기 권한만 줬다. 관리자 계정을 입력해도 연결은 되지만, Home Assistant가 침해됐을 때 NAS 전체가 노출될 수 있어 전용 계정이 낫다. SMB 서비스는 `제어판 → 파일 서비스 → SMB`에서 활성화하고, 가능하면 최소 SMB 프로토콜을 2.1 이상으로 둔다.

## Home Assistant에 SMB 공유 연결

Home Assistant에서 `설정 → 시스템 → 저장소`로 이동해 `네트워크 저장소 추가`를 선택한다. 다음처럼 입력하면 된다.

| 입력란 | 값 |
|---|---|
| 이름 | `synology-ha-backup` |
| 사용 용도 | `백업` |
| 서버 | `192.168.0.10` |
| 프로토콜 | `Samba/CIFS` |
| 공유 | `ha-backup` |
| 사용자 이름·비밀번호 | Synology 전용 계정 정보 |

여기서 공유 항목에 `/volume1/ha-backup`을 넣으면 안 된다. CIFS는 NAS 주소 뒤의 공유 이름만 입력하는 방식이라 `ha-backup`이 맞다. `연결`을 누른 뒤 저장소 목록에 초록색 연결 상태가 표시되는지 확인한다.

## 백업 기본 위치를 NAS로 변경

저장소를 `백업` 용도로 처음 추가하면 기본 대상이 될 수 있지만, 기존 설정이 남아 있으면 수동으로 바꿔야 한다. `설정 → 시스템 → 백업`에서 `설정 및 기록`을 열고 우측 상단 메뉴의 `기본 작업 위치 변경`을 선택한다. 목록에서 `synology-ha-backup`을 고른다.

이제 수동 백업을 하나 만든 뒤 Synology File Station에서 실제 `.tar` 백업 파일이 생겼는지 확인한다. 화면에 백업 성공만 표시되고 NAS 폴더가 비어 있다면, `백업` 용도로 저장소를 추가했는지부터 다시 봐야 한다. `미디어`나 `공유`로 등록한 저장소는 백업 대상 목록에 나타나지 않는다.

## 미디어 공유로 쓸 때의 차이

사진·음악을 Home Assistant의 미디어 브라우저에서 읽게 하려면 같은 NAS 공유를 `미디어` 용도로 별도 등록한다. 등록 후 `/media/synology-media`처럼 연결되며, 앱이나 통합구성요소가 접근할 수 있다. 백업 폴더와 미디어 폴더를 하나로 합치면 권한과 삭제 실수가 섞이므로 공유 폴더를 분리하는 편이 안전하다.

| 목적 | NAS 공유 폴더 | Home Assistant 용도 |
|---|---|---|
| 설정 백업 | `ha-backup` | 백업 |
| 음악·사진 | `ha-media` | 미디어 |
| 앱이 읽는 파일 | `ha-share` | 공유 |

## 연결이 끊길 때 확인할 것

- NAS IP를 DHCP 예약으로 고정했는가
- Synology 방화벽에서 Home Assistant IP의 SMB 접근을 허용했는가
- 계정에 공유 폴더 읽기·쓰기 권한이 모두 있는가
- 공유 이름에 대소문자나 공백을 잘못 넣지 않았는가
- NAS 재부팅 뒤 저장소 상태가 다시 연결되는가

네트워크 저장소는 백업 사본을 하나 더 만드는 기능이지 3-2-1 백업을 완성하는 기능은 아니다. NAS와 Home Assistant가 같은 정전·랜섬웨어 영향권에 있으므로, Synology Hyper Backup으로 이 폴더를 다른 디스크나 오프사이트 대상으로 한 번 더 복사해야 한다. Home Assistant 2026.9.1에는 백업 수신 스트리밍 오류 수정도 포함됐지만, 실제 복원 테스트를 대신해 주지는 않는다. [2026.9 릴리스 노트](https://www.home-assistant.io/blog/2026/09/02/release-20269/)도 확인해 두는 편이 좋다.

짧게 정리하면 `Synology 전용 계정 생성 → SMB 공유 준비 → Home Assistant 네트워크 저장소 추가 → 백업 기본 위치 변경 → 실제 파일과 복원 가능성 확인` 순서다. Home Assistant Docker를 NAS에서 운영 중이라면 이 메뉴를 찾느라 시간을 쓰지 말고, 컨테이너의 `/config` 백업과 Synology 백업 작업을 별도로 구성해야 한다.

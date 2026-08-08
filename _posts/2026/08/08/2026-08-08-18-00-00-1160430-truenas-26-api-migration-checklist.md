---
layout: post
title: "TrueNAS 26-BETA.2 REST API 제거 - 홈서버 자동화 점검과 안전한 업그레이드 준비"
description: "TrueNAS 26-BETA.2에서 REST API가 제거되고 WebSocket API로 바뀐다. 홈서버에서 사용하는 스크립트와 앱을 찾고, 백업 후 안전하게 업그레이드를 준비하는 방법을 정리한다."
date: 2026-08-08
tags: [TrueNAS, HomeLab, 홈서버, NAS보안, 백업전략]
comments: true
share: true
---

![TrueNAS 홈서버와 네트워크 스토리지](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

TrueNAS 26-BETA.2를 테스트하기 전에 REST API를 사용하는 자동화부터 찾아야 한다. TrueNAS 26에서는 기존 REST API가 제거되고 JSON-RPC 2.0 WebSocket API로 바뀐다. 백업 스크립트나 Home Assistant 연동이 하나라도 남아 있으면 업그레이드 후 조용히 실패할 수 있다. 다만 2026년 8월 8일 기준 26 계열은 베타다. 중요한 파일을 보관하는 운영 NAS는 25.10.5에 두고, 별도 테스트 풀에서만 확인하는 게 맞다.

## 이번 변경에서 실제로 달라지는 점

TrueNAS 공식 버전 문서에 따르면 26-BETA.2에는 Linux Kernel 6.18, OpenZFS 2.4.1, Docker Engine 29.0.4가 포함된다. 눈에 띄는 기능보다 더 중요한 변화는 API다. 25.10에서 이미 사용 중단 예정이었던 REST API가 26에서 동작하지 않는다.

| 확인 항목 | TrueNAS 25.10 | TrueNAS 26 계열 |
|---|---|---|
| API | REST API 사용 가능 | REST API 제거 |
| 새 연동 방식 | 선택 사항 | JSON-RPC 2.0 WebSocket |
| 앱·컨테이너 | 기존 설정 유지 | 버전별 호환성 확인 필요 |
| 운영 권장 | 안정 버전 | 베타는 테스트 전용 |

내 NAS에서는 `curl`로 `/api/v2.0`을 호출하는 백업 알림 스크립트가 문제였다. 웹 UI만 쓰는 사용자는 영향이 작지만, 모니터링·백업·Home Assistant 자동화를 붙였다면 먼저 검색해야 한다.

## 업그레이드 전에 REST API 사용처 찾기

NAS 자체를 수정하는 명령이 아니라, 백업해 둔 스크립트 폴더에서 호출 흔적만 찾는다.

```bash
rg -n --hidden -g '*.sh' -g '*.py' -g '*.yaml' -g '*.yml' \
  'api/v2\.0|/api/|rest|truenas' /volume1/docker /volume1/scripts
```

TrueNAS 안에 저장한 앱 설정은 앱별 Compose 파일과 환경 변수도 확인한다. 특히 `TRUENAS_HOST`, API 키, `curl -k`가 함께 나오면 업그레이드 전에 별도 목록으로 적어 둔다. API 키를 코드에 직접 넣었다면 마이그레이션 후 새 키를 발급하고 기존 키는 폐기한다.

## 안전한 테스트 순서

먼저 System Settings > General > Manage Configuration에서 설정 파일을 다운로드한다. 이어서 Apps의 각 앱 설정과 데이터셋 스냅샷을 저장한다. 설정 파일은 데이터 백업이 아니므로, 이것만 받아 두고 안심하면 안 된다.

| 순서 | 작업 | 통과 기준 |
|---|---|---|
| 1 | 설정 파일 다운로드 | 복구용 파일을 다른 장치에도 보관 |
| 2 | 앱·VM 종료 및 스냅샷 | 실행 중인 쓰기가 없음 |
| 3 | 테스트 부팅 장치에 26-BETA.2 설치 | 웹 UI와 풀 import 확인 |
| 4 | SMB, 앱, 백업 작업 점검 | 실제 파일을 읽고 복원 |
| 5 | 자동화 API 교체 | 알림과 예약 작업 성공 |

공식 문서도 베타 버전은 중요한 작업에 사용하지 말라고 안내한다. 기존 부트 환경으로 되돌릴 수 있다고 생각했는데, 업그레이드 중 풀 기능 플래그와 앱 설정이 함께 바뀌면 단순 롤백만으로 원상 복구되지 않을 수 있다. 테스트가 끝나기 전에는 운영 풀의 ZFS 기능 플래그를 새 버전에 맞춰 업그레이드하지 않는 편이 안전하다.

## API를 쓰는 자동화는 어떻게 바꿀까

기존 REST 호출을 그대로 주소만 바꾸는 방식은 안 된다. 공식 마이그레이션 안내에 맞춰 WebSocket API 문서에서 같은 기능의 메서드를 찾고, 테스트 장비에서 인증·응답 형식·오류 처리를 모두 확인해야 한다. 새 연동은 `auth.login_ex`와 API 키 인증을 기준으로 작성하고, 외부에서 NAS 관리 포트를 공개하지 않는다. Tailscale 같은 VPN 안에서만 API를 호출하면 포트포워딩으로 관리 API를 인터넷에 노출하는 실수를 줄일 수 있다.

```text
운영 NAS 25.10.5
  ├─ 설정 파일 + 데이터셋 스냅샷 보관
  ├─ REST API 사용 스크립트 목록화
  └─ 테스트 NAS에서 WebSocket API 검증
          └─ 성공 후에만 운영 환경의 자동화 교체
```

이 그림에서 볼 부분은 업그레이드보다 자동화 검증이 앞선다는 점이다. UI에서 SMB 파일이 보인다고 백업과 알림까지 정상인 것은 아니다.

## 짧게 정리

- TrueNAS 26-BETA.2는 REST API가 제거된 테스트 버전이다.
- `curl`, Python, Home Assistant 연동에서 `/api/v2.0` 사용 여부를 먼저 찾는다.
- 설정 파일과 스냅샷을 모두 준비하고, 운영 풀은 25.10.5에 남긴다.
- 새 API는 JSON-RPC 2.0 WebSocket과 `auth.login_ex` 기준으로 별도 검증한다.

참고: [TrueNAS 26 Version Notes](https://www.truenas.com/docs/scale/26/gettingstarted/versionnotes/)

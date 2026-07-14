---
layout: post
title: "TrueNAS 26 REST API 제거 - 업그레이드 전 홈서버 자동화 점검법"
description: "TrueNAS 26에서 REST API가 제거된다. NAS 백업·모니터링 스크립트가 업그레이드 후 멈추지 않도록 API 사용 여부를 찾고 WebSocket API 전환을 준비하는 방법을 정리했다."
date: 2026-07-14
tags: [TrueNAS, 홈서버, HomeLab, NAS설정, NAS보안]
comments: true
share: true
---

![TrueNAS 홈서버와 API 자동화 점검](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)

TrueNAS 26으로 올리기 전에 백업 스크립트와 모니터링 도구가 REST API를 쓰는지 찾아야 한다. 공식 버전 노트 기준 TrueNAS 26에서는 기존 REST API가 제거되고 JSON-RPC 2.0 WebSocket API로 넘어간다. 웹 화면에서 파일을 읽고 쓰는 것만 하는 사용자는 영향이 적지만, 홈서버 자동화가 있다면 업데이트 후 조용히 실패할 수 있다.

## 이번 변경에서 걸리는 구성

| 구성 | TrueNAS 25.10 | TrueNAS 26 |
|---|---|---|
| 관리 API | REST API | JSON-RPC 2.0 over WebSocket |
| 자동화 요청 | `/api/v2.0/...` | WebSocket 메서드 호출 |
| 점검 대상 | 백업, 알림, Grafana, Ansible | API URL·인증 방식 전부 |

![NAS 자동화 구성에서 API 요청 경로를 확인하는 모습](https://images.unsplash.com/photo-1451187580094-2cfb76b7c9b7?w=800&q=80)
그림에서 볼 부분은 NAS UI가 아니라 NAS를 호출하는 별도 스크립트와 컨테이너다.

## 업그레이드 전에 호출 흔적 찾기

먼저 자동화 파일이 모여 있는 디렉터리에서 REST API 문자열을 검색한다. 아래 명령은 파일을 수정하지 않고 URL과 관련 라이브러리만 찾는다.

```bash
cd ~/homelab
rg -n "api/v2\.0|/api/|truenas|midclt|curl.*api" . \
  --glob '!**/node_modules/**' --glob '!**/.git/**'
```

Ansible이나 Python에서 `api/v2.0`을 찾았다면 해당 작업은 그대로 두고 업데이트하지 않는다. 특히 디스크 상태, 스냅샷, Cloud Sync를 주기적으로 읽는 스크립트는 성공 로그만 믿지 말고 실제 파일 생성까지 확인해야 한다.

## 안전한 점검 순서

1. **System Settings → General → Manage Configuration**에서 현재 설정을 파일로 내려받는다.
2. 각 데이터셋의 스냅샷과 별도 백업을 확인한다. 설정 파일만으로 풀 안의 데이터가 복구되지는 않는다.
3. API를 호출하는 컨테이너와 cron 작업을 잠시 멈추고, 테스트용 TrueNAS 26 VM에서 같은 호출이 실패하는지 본다.
4. REST 호출은 공식 API 문서의 해당 WebSocket 메서드로 바꾼다. TrueNAS 26의 새 인증 메서드는 `auth.login_ex`이며 API 키, OTP, SCRAM 등을 처리한다.
5. 업데이트 후에는 SMB 파일 복사뿐 아니라 스냅샷 생성, 백업 대상 업로드, 알림 발송까지 각각 시험한다.

![TrueNAS 업그레이드 전후 점검 항목](https://images.unsplash.com/photo-1544197150-880d4212a0c4?w=800&q=80)

내가 가장 쉽게 놓친 지점은 Docker 컨테이너 환경 변수였다. 호스트의 파일만 검색하면 `TRUENAS_URL`이나 API 경로가 Compose 파일에 남아 있는 걸 지나칠 수 있다. 실행 중인 컨테이너 설정도 같이 확인한다.

```bash
docker ps --format '{{.Names}}' | while read name; do
  echo "--- $name"
  docker inspect "$name" --format '{{range .Config.Env}}{{println .}}{{end}}' \
    | rg -i "truenas|api|token" || true
done
```

TrueNAS 26은 아직 Early Release(테스트·피드백용 초기 릴리스)로 안내되고 있어 중요한 파일 서버에 바로 적용할 단계는 아니다. 운영 장비는 현재 메이저 버전의 최신 유지보수판에 두고, 남는 SSD나 Proxmox VM에서 API와 백업 흐름을 먼저 검증하는 편이 안전하다. REST API를 쓰지 않는 구성이라도 설정 백업, 스냅샷, 복구 테스트는 업그레이드 전 체크리스트에서 빼지 않는다.

짧게 정리하면 `api/v2.0` 검색 → 설정·데이터 별도 백업 → 테스트 VM 검증 → WebSocket API 전환 → 실제 복구 테스트 순서다. TrueNAS 26의 새 기능보다 먼저 자동화 의존성을 확인해야 업그레이드가 홈서버 장애로 번지지 않는다.

- [TrueNAS 26 Version Notes](https://www.truenas.com/docs/scale/26/gettingstarted/versionnotes/)
- [TrueNAS API 문서](https://api.truenas.com/)

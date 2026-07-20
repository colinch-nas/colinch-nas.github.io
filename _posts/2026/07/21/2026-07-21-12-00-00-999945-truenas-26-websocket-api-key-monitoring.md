---
layout: post
title: "TrueNAS 26 API 키 설정 - REST API 대신 WebSocket으로 풀 상태 점검하기"
description: "TrueNAS 26에서 제거된 REST API 대신 사용자 API 키와 JSON-RPC WebSocket API를 설정하고, 외부 모니터링에서 풀 상태를 안전하게 조회하는 방법을 정리한다."
date: 2026-07-21
tags: [TrueNAS, HomeLab, NAS보안, 자체호스팅]
comments: true
share: true
---
![TrueNAS 26 WebSocket API와 NAS 상태 모니터링](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

TrueNAS 26에서 외부 모니터링을 붙일 때 예전 REST API 주소를 계속 쓰면 안 된다. REST API는 25.04부터 폐기 단계였고 TrueNAS 26에서 제거됐기 때문에, 이제는 사용자 API 키와 JSON-RPC 2.0 WebSocket API 조합으로 바꾸는 게 맞다. 여기서는 TrueNAS SCALE 26 테스트 서버에서 API 키를 만들고, 별도 PC에서 ZFS 풀 상태를 읽는 흐름을 정리한다.

이 그림에서 볼 부분은 NAS 관리 화면을 인터넷에 그대로 여는 게 아니라, 인증된 API 연결만 상태 조회에 사용하는 구조다.

## 환경과 선택 기준

내 환경은 TrueNAS SCALE 26, 관리 주소 `192.168.0.30`, ZFS 풀 이름 `tank`다. API를 Uptime Kuma나 간단한 알림 스크립트에 붙이려면 관리자 비밀번호를 저장하는 것보다 만료일이 있는 사용자 키를 따로 만드는 편이 낫다.

| 항목 | 기존 방식 | TrueNAS 26 기준 |
|---|---|---|
| 통신 | REST API | JSON-RPC 2.0 over WebSocket |
| 인증 | 계정·기존 키 혼용 | 사용자 연결 API 키 |
| 권한 | 관리자 계정으로 시작하기 쉬움 | 모니터링 전용 사용자 권장 |
| 주소 | `/api/v2.0/...` | `wss://NAS주소/api/current` |

## API 키 발급

웹 UI 오른쪽 위 계정 메뉴에서 **My API Keys → Add**를 선택한다. `truenas-monitor`처럼 용도를 드러내는 이름을 쓰고 만료일도 정한다. 키 문자열은 생성 직후 한 번만 표시되므로 바로 비밀번호 관리자에 보관한다.

모니터링 전용 사용자를 만들었다면 풀 조회에 필요한 읽기 권한만 부여한다. 처음부터 `root` 키를 복사해 쓰는 방식은 편하지만, 키가 유출되면 2단계 인증도 우회하는 비밀번호급 권한이 된다. TrueNAS 공식 문서도 API 키가 연결된 사용자의 2FA 설정 대상이 아니라고 경고한다.

## WebSocket 연결 테스트

모니터링 PC에 Python과 `websocket-client`를 설치한다. 코드 바로 아래의 `pool.query`는 풀 목록과 상태만 읽는 호출이라 삭제나 변경 작업을 하지 않는다.

```bash
python3 -m pip install websocket-client
```

아래 명령에서 API 키는 셸 히스토리에 남을 수 있으니 테스트 후 기록을 지운다. 운영 환경에서는 환경 변수보다 시크릿 저장소를 쓰는 편이 안전하다.

```bash
export TRUENAS_HOST="192.168.0.30"
export TRUENAS_API_KEY="생성한_API_키"
python3 - <<'PY'
import json
import os
import ssl
import websocket

host = os.environ["TRUENAS_HOST"]
key = os.environ["TRUENAS_API_KEY"]
ws = websocket.create_connection(
    f"wss://{host}/api/current",
    timeout=10,
    sslopt={"cert_reqs": ssl.CERT_NONE},  # 사설 인증서 테스트용
)

def call(request_id, method, params=None):
    ws.send(json.dumps({
        "msg": "method",
        "method": method,
        "params": params or [],
        "id": request_id,
    }))
    while True:
        response = json.loads(ws.recv())
        if response.get("id") == request_id:
            return response

print(call(1, "auth.login_with_api_key", [key]))
print(json.dumps(call(2, "pool.query"), indent=2))
ws.close()
PY
```

처음엔 `wss` 연결이 안 돼서 API가 막힌 줄 알았는데, 원인은 TrueNAS의 기본 자체 서명 인증서였다. 위 `CERT_NONE`은 내부 테스트에서만 쓰고, 실제 운영에서는 TrueNAS에 신뢰할 수 있는 인증서를 설치한 뒤 해당 줄을 제거해야 한다. HTTP로 API 키를 보내면 키가 취소될 수 있으므로 `ws://`로 낮추지 않는다.

## 점검 결과와 주의사항

출력에 `result` 배열과 풀의 `status`가 나오면 연결은 끝난다. 자동화에는 `status`가 `ONLINE`인지 읽는 정도만 붙이고, 스냅샷 삭제나 공유 권한 변경은 API 키에 맡기지 않는다.

- API 키는 생성 화면에서 한 번만 복사한다.
- 만료일이 지난 키와 사용하지 않는 키는 **My API Keys**에서 삭제한다.
- TrueNAS 26 업그레이드 전에 `/api/v2.0`를 호출하는 모니터링을 찾아 WebSocket 방식으로 바꾼다.
- 인증서 검증을 끈 코드는 홈 네트워크 밖으로 가져가지 않는다.
- `pool.query`가 정상이어도 디스크 상태까지 정상이라는 뜻은 아니므로 SMART와 스크럽 알림은 별도로 본다.

짧게 정리하면 `모니터링 전용 사용자 → 만료 API 키 → HTTPS WebSocket → 읽기 전용 호출` 순서다. TrueNAS 26에서는 REST API 호환보다 사용 중인 모니터링 도구의 JSON-RPC WebSocket 지원 여부를 우선 확인한다. 참고한 문서는 [TrueNAS 26 API 문서](https://www.truenas.com/docs/scale/api/)와 [TrueNAS API 키 화면 안내](https://www.truenas.com/docs/scale/26/toptoolbar/settings/apikeysscreen/)다.

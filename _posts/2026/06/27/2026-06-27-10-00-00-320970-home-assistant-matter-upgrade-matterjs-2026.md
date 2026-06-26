---
layout: post
title: "Home Assistant Matter 업그레이드 — matter.js 전환으로 카메라·도어벨 드디어 됩니다"
description: "Home Assistant가 2026년 6월 Matter 서버를 matter.js로 전환했다. 연결 안정성이 크게 개선됐고 Matter 1.5.1 기반으로 카메라·도어벨 지원이 시작됐다. 업데이트 방법과 실제 달라진 점 정리."
date: 2026-06-27
tags: [HomeAssistant, Matter, 스마트홈, 홈오토메이션, 스마트홈구축]
comments: true
share: true
---

# Home Assistant Matter 업그레이드 — matter.js 전환으로 카메라·도어벨 드디어 됩니다

![Home Assistant 스마트홈 기기 연결 허브 구성](https://images.unsplash.com/photo-1558618047-3c8c76ca7d13?w=800&q=80)

4개월 베타 끝에 Home Assistant가 Matter 서버를 **matter.js** 기반으로 공식 전환했다(2026.6.23). 기존 Python 서버에서 반복됐던 "기기 사용 불가" 오류가 상당 부분 해결됐고, Matter 1.5.1 규격 대응으로 카메라·도어벨 연동의 토대도 갖춰졌다.

## 환경 정보

| 항목 | 버전 |
|------|------|
| Home Assistant Core | 2026.6.x |
| Matter Server | matter.js 기반 7.x |
| 지원 Matter 규격 | Matter 1.5.1 (1.6 준비 중) |
| Thread 보더 라우터 | HA SkyConnect / Yellow |

## 왜 기존 Matter가 불안정했나

HA에 Matter를 처음 붙였을 때부터 골치였다. 연결은 되는데 하루에 한두 번 기기가 "사용 불가"로 바뀌고, HA 재시작 후 Matter 서버가 복구되는 데 2~3분이 걸렸다. Eve Energy나 Nanoleaf를 쓰는 사람들은 거의 다 이 현상을 겪었을 거다.

근본 원인은 Python 기반 Matter 서버의 네트워킹 스택이었다. 기기 발견, 재연결 로직 쪽 안정성이 부족했고, Matter 규격 자체가 버전이 올라갈수록 지원해야 할 기기 유형이 늘어나는데 대응 속도가 느렸다.

## matter.js 전환으로 실제로 달라진 것

**연결 안정성**이 핵심이다. matter.js는 TypeScript로 구현된 오픈소스 Matter 스택인데, 네트워킹 레이어를 완전히 새로 설계했다. 베타 기간 커뮤니티 반응을 보면 기기가 "사라지는" 빈도가 눈에 띄게 줄었다는 보고가 많다.

시작 속도도 빨라졌다. HA 재시작 후 Matter 기기들이 다시 붙는 시간이 이전 대비 절반 이하다.

그리고 이번 전환의 진짜 의미는 **Matter 1.5.1 대응**이다. Matter 1.5에서 카메라·도어벨 디바이스 타입이 공식 추가됐고, matter.js 기반 서버가 WebRTC, 스트리밍, 스냅샷, PTZ 제어를 처리할 수 있는 구조를 갖췄다. 아직 모든 기능이 완성된 건 아니고 개발이 진행 중이지만, 이전까지는 구조 자체가 없었다.

## Matter 서버 업데이트 방법

HA 애드온으로 Matter Server를 쓰고 있다면 업데이트만 하면 된다.

### 1. 애드온 버전 확인 및 업데이트

```
Settings → Add-ons → Matter Server → Update
```

버전이 7.x 이상이면 matter.js 기반이다. 6.x 이하면 업데이트가 필요하다.

### 2. 업데이트 후 통합 리로드

대부분은 자동으로 전환되지만, 간혹 Matter 통합이 초기화 오류를 낸다. 이럴 때:

```
Settings → Devices & Services → Matter (BETA) → 우측 메뉴 → Reload
```

그래도 안 되면 Matter 통합을 삭제하고 재설치해야 한다. **단, 기존에 페어링된 기기들을 Matter에서 먼저 공장초기화해야** 재페어링이 가능하다. 리셋 없이 재등록하면 "Already commissioned" 오류가 뜬다.

### 3. 네트워크 토폴로지 시각화

이번 업데이트에 새로 생긴 기능인데, Matter/Thread 기기들의 네트워크 연결 상태를 시각적으로 보여준다.

```
Settings → Devices & Services → Matter → Network topology
```

![Matter Thread 네트워크 연결 상태 시각화 대시보드](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=800&q=80)

Thread 메시 네트워크에서 어느 기기가 라우터 역할을 하고 어디서 신호가 약한지 한눈에 보여서 문제 진단이 훨씬 쉬워졌다.

## 2026년 6월 기준 Matter 기기 지원 현황

| 기기 유형 | 지원 상태 |
|----------|----------|
| 전구·조명 | ✅ 완전 지원 |
| 스위치·콘센트 | ✅ 완전 지원 |
| 온도·습도 센서 | ✅ 완전 지원 |
| 잠금장치 | ✅ 완전 지원 |
| 커튼·블라인드 | ✅ 완전 지원 |
| 에어컨·히터 | ✅ 완전 지원 |
| 카메라·도어벨 | ⚠️ 부분 지원 (스트리밍 개발 중) |
| 사이렌 | ✅ 이번 업데이트로 추가 |

Zigbee 기기를 이미 쓰고 있다면 굳이 Matter로 옮길 이유는 없다. [Zigbee2MQTT]({% post_url 2026-06-11-10-00-00-086415-zigbee2mqtt-setup-no-hub %})가 이미 안정적이다. Matter는 Zigbee 없이 새로 구매하는 기기를 추가할 때 특히 유리하다.

## 삽질했던 부분

**Thread 보더 라우터 펌웨어를 먼저 확인해라.** matter.js 전환 후 Thread 기기들이 불안정하게 연결된다면 보더 라우터 펌웨어 버전부터 체크해야 한다. HA SkyConnect 기준으로 2026년 초에 펌웨어 업데이트가 있었는데, 이걸 안 하면 matter.js와 충돌하는 케이스가 있다.

```
Settings → System → Hardware → SkyConnect → 펌웨어 업데이트 확인
```

**Eve, Nanoleaf 기기 재연결 필요.** 업데이트 직후 이 브랜드 기기들이 "사용 불가"로 바뀐 케이스가 커뮤니티에 여럿 보고됐다. 전원을 완전히 껐다 켜면 대부분 자동 재연결된다. 그래도 안 되면 HA에서 기기 삭제 → 기기 공장초기화 → 재페어링 순서로 진행하면 된다.

**카메라는 아직 안 된다고 생각하고 있어야 한다.** 구조는 갖춰졌지만 실제 스트리밍이 안정화되려면 몇 달 더 걸릴 것 같다. 공식 블로그에서도 "상당한 개발 노력이 필요하다"고 명시하고 있으니 바로 쓸 수 있다고 기대하면 실망한다.

## 한 줄 정리

matter.js로 전환 후 Matter 연동 안정성이 실질적으로 개선됐다 — 카메라·도어벨은 구조가 갖춰진 단계고 완전한 스트리밍은 추가 업데이트를 기다려야 한다.

---

다음엔 Beelink Mini PC를 홈서버로 쓰는 실측 비교 — 소비전력과 성능 사이의 선택을 다룬다.

**참고 링크**
- [Home Assistant 공식 블로그: The Matter upgrade you've been waiting for](https://www.home-assistant.io/blog/2026/06/23/the-matter-upgrade-youve-been-waiting-for/)
- [Home Assistant Matter 통합 공식 문서](https://www.home-assistant.io/integrations/matter/)
- [Home Assistant 에너지 모니터링 대시보드 구성]({% post_url 2026-06-26-14-00-00-296280-home-assistant-energy-monitoring-dashboard %})

# Colinch NAS 블로그 — 전용 하네스

> 공통 규칙(파일명·frontmatter·SEO·인간 글쓰기·트리거 프로토콜·git 워크플로)은 상위 폴더 `../CLAUDE.md` 참조
> 트리거 약어: `colinch-nas`

---

## 블로그 정보

| 항목 | 값 |
|------|----|
| 사이트 URL | https://colinch-nas.github.io |
| 저자 | Colinch NAS |
| 언어 | 한국어 |
| 대상 독자 | NAS 구매자·사용자, 홈서버 구축에 관심 있는 일반인·IT 종사자 |
| 핵심 방향 | NAS·홈서버·스마트홈 설정을 실제로 따라할 수 있게, 최신 제품·뉴스 반영 |

---

## 포스트 선정 방법 (매 포스팅 시 실행)

```
1. WebSearch로 최신 NAS·홈서버·스마트홈 뉴스 확인
   검색어 예시:
   - "Synology NAS 신제품 2026"
   - "QNAP 업데이트 2026"
   - "Home Assistant 새기능"
   - "Proxmox 업데이트"
   - "self-hosted app 추천 2026"
   - "홈서버 홈랩 최신"
   → 신제품 출시, 주요 펌웨어·앱 업데이트, 화제의 자체호스팅 앱이 있으면 최우선으로 다룬다
   → 실제 버전·모델명을 검색으로 확인하고 구체적으로 언급

2. 시리즈 현황에서 다음 미작성 주제 확인
   → 최신 뉴스와 연결되는 시리즈 주제를 우선 선택

3. 중복 확인
   grep -rl "주제키워드" _posts/ --include="*.md"
```

---

## 주제 방향과 예시

### 방향 1 — NAS 설정 가이드

Synology, QNAP, TrueNAS 등 특정 제품의 설정 방법. 처음 구매한 사람도 따라할 수 있을 만큼 구체적으로.

```
주제 예시:
- "Synology DS923+ 처음 설정 — DSM 7 기준 초기 설정 완전 정복"
- "QNAP TS-464 Docker 설치 방법 2026"
- "TrueNAS SCALE 설치부터 SMB 공유까지"
- "Synology NAS 외부 접속 설정 — DDNS와 Let's Encrypt 인증서"
- "QNAP NAS 백업 전략 — 3-2-1 규칙 실제 구현"
```

### 방향 2 — 홈서버·홈랩 구축

미니 PC, 구형 노트북 등으로 홈서버를 만드는 방법. Proxmox, Docker, Ubuntu Server 활용.

```
주제 예시:
- "Proxmox VE 8 설치 — 미니PC로 가상화 서버 만들기"
- "Docker Compose로 자체 호스팅 서비스 한 번에 관리하기"
- "Beelink Mini PC 홈서버로 쓰는 법 — 소비전력 vs 성능 비교"
- "Ubuntu Server 24.04 초기 설정 — SSH·방화벽·사용자 설정"
- "홈랩 네트워크 구성 — VLAN으로 IoT 기기 분리하기"
```

### 방향 3 — 자체 호스팅 서비스

Nextcloud, Jellyfin, Vaultwarden 등 직접 운영하는 서비스 설치·설정 가이드.

```
주제 예시:
- "Jellyfin 설치 설정 — Plex 대신 완전 무료로 미디어 서버 구축"
- "Nextcloud 설치 — 구글 드라이브 대신 내 서버에 클라우드 만들기"
- "Vaultwarden으로 비밀번호 관리 서버 직접 운영하기"
- "Immich — 구글 포토 대체 자체 호스팅 사진 관리"
- "Uptime Kuma로 내 서비스들 모니터링하기"
```

### 방향 4 — Home Assistant 스마트홈

Home Assistant 설치·자동화·기기 연동. 실제 써보면서 겪은 삽질 포함.

```
주제 예시:
- "Home Assistant 설치 방법 2026 — Raspberry Pi vs NAS vs VM 비교"
- "Home Assistant로 스마트 전구 자동화 — 조명 ON/OFF 자동화 만들기"
- "Zigbee2MQTT 설정 — 허브 없이 Zigbee 기기 연결하는 법"
- "Home Assistant 에너지 모니터링 — 실시간 전력 사용량 대시보드"
- "Matter 지원 기기 Home Assistant 연동 — 2026 현재 상황 정리"
```

### 방향 5 — NAS·홈서버 보안·네트워크

외부 공격, 랜섬웨어 대비, VPN, 역방향 프록시 설정.

```
주제 예시:
- "NAS 랜섬웨어 대비 — Synology 기준 보안 설정 체크리스트"
- "Tailscale VPN으로 어디서든 집 서버 안전하게 접속하기"
- "Nginx Proxy Manager로 도메인·HTTPS 한 번에 관리"
- "Cloudflare Tunnel — 포트 포워딩 없이 서비스 외부 공개하기"
- "NAS IP 차단 설정 — 무차별 로그인 시도 막는 방법"
```

---

## 글쓰기 스타일

### 언어 수준

- 처음 보는 개념은 괄호 안에 설명
  - "Reverse Proxy(외부 도메인 요청을 내부 서버로 전달하는 중계 서버)"
  - "DDNS(유동 IP에서도 고정 도메인처럼 쓰는 서비스)"
- 코드·명령어는 반드시 코드 블록으로 표기
- 실제 버전·모델명 명시 필수 ("DSM 7.2.2", "HA Core 2025.6", "Synology DS923+")

### 톤

- 직접 설정해보고 쓰는 느낌: "이렇게 하면 된다", "이건 삽질했다"
- 공식 문서에 없는 실제 설정 경험 중심
- 버전 업데이트나 인터페이스 변경으로 달라진 점 명시

### AI 감지 표현 금지 (이 블로그 추가)

```
- "스마트홈 시대가 도래했습니다"
- "다양한 기능을 활용할 수 있습니다"
- "편리한 생활을 누리세요"
- "혁신적인 기술"
- "강력한 성능을 제공합니다"
```

---

## 포스트 구조 템플릿

```markdown
---
layout: post
title: "[제품/서비스명] — [핵심 작업: 설치·설정·활용]"
description: "구체적 설명 50~160자. 제품명·버전 포함. 어떤 결과를 얻을 수 있는지 명시."
date: YYYY-MM-DD
tags: [메인태그, 태그2, 태그3]
comments: true
share: true
---

# 제목

![이미지](https://images.unsplash.com/photo-XXXXXXXX?w=800&q=80)

[첫 문장 = 결론. "X를 하면 Y가 된다" 또는 "X 버전부터 이렇게 바뀌었다".]

## 환경 정보

| 항목 | 버전/모델 |
|------|----------|
| NAS 모델 / OS | Synology DS923+ / DSM 7.2.2 |
| 관련 앱 버전 | Docker 24.0.x |

## 왜 이게 필요한가

[문제 상황 또는 이 설정이 필요한 이유.]

## 설정 방법

[단계별 설명. 명령어, 화면 경로 포함.]

```bash
# 실제 명령어
```

## 삽질했던 부분

[실제로 걸렸던 문제와 해결법. 없으면 생략.]

## 한 줄 정리

[이 설정으로 무엇을 얻었는지 한 문장.]

---

다음엔 [관련 주제]를 다룬다.
```

---

## SEO 키워드

| 그룹 | 키워드 |
|------|--------|
| Synology | `시놀로지 설정`, `Synology Docker`, `DSM 설정`, `시놀로지 외부접속` |
| QNAP | `QNAP 설정`, `QNAP Docker`, `QNAP NAS 활용` |
| 홈서버 | `홈서버 구축`, `Proxmox 설치`, `홈랩`, `미니PC 서버` |
| 스마트홈 | `Home Assistant`, `홈오토메이션`, `스마트홈 구축`, `Zigbee 설정` |
| 자체호스팅 | `Jellyfin 설치`, `Nextcloud 설치`, `자체호스팅`, `셀프호스팅` |
| 보안·네트워크 | `NAS 보안`, `Tailscale VPN`, `Cloudflare Tunnel`, `역방향 프록시` |

---

## 태그 목록

```
# NAS 브랜드·OS
Synology, QNAP, TrueNAS, TerraMaster, DSM

# 홈서버·가상화
Proxmox, Docker, Ubuntu, HomeLab, 홈서버, 미니PC

# 스마트홈
HomeAssistant, Zigbee, Matter, 스마트홈, 홈오토메이션, Zigbee2MQTT

# 자체호스팅 서비스
Jellyfin, Nextcloud, Vaultwarden, Immich, Plex, UptimeKuma, 자체호스팅

# 네트워크·보안
Tailscale, CloudflareTunnel, VPN, NginxProxyManager, NAS보안, DDNS

# 주제 분류
NAS설정, 홈서버구축, 미디어서버, 백업전략, 에너지모니터링
```

---

## 시리즈 현황

### 시리즈 1 — Synology NAS 완전 정복 (진행 예정)

| 순서 | 주제 | 상태 |
|------|------|------|
| 1 | DSM 초기 설정 및 보안 설정 | 완료 |
| 2 | Docker 설치 및 컨테이너 운영 | 완료 |
| 3 | Jellyfin 미디어 서버 구축 | 완료 |
| 4 | 외부 접속 (DDNS + Let's Encrypt) | 미작성 |
| 5 | 랜섬웨어 대비 백업 전략 | 미작성 |

### 시리즈 2 — Home Assistant 스마트홈 (진행 예정)

| 순서 | 주제 | 상태 |
|------|------|------|
| 1 | Home Assistant 설치 방법 비교 | 미작성 |
| 2 | Zigbee2MQTT 설정 | 미작성 |
| 3 | 조명·전력 자동화 만들기 | 미작성 |
| 4 | 대시보드 구성 | 미작성 |

### 시리즈 3 — 자체 호스팅 서비스 모음 (진행 예정)

| 순서 | 주제 | 상태 |
|------|------|------|
| 1 | Nextcloud 설치 — 내 서버에 클라우드 | 미작성 |
| 2 | Vaultwarden 비밀번호 서버 | 미작성 |
| 3 | Immich 사진 관리 서버 | 미작성 |
| 4 | Uptime Kuma 모니터링 | 미작성 |

마지막 포스트: `_posts/2026/06/29/2026-06-29-12-00-00-395040-paperless-ngx-docker-install-nas-document-management.md`
다음 일련번호: `407385`

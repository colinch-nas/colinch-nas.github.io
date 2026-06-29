---
layout: post
title: "ZimaCube 2 완전 정복 — $799짜리 NAS인데 로컬 AI까지 돌아간다"
description: "2026년 4월 출시된 ZimaCube 2 실사용 리뷰. 6베이 SATA에 PCIe 확장 슬롯 탑재로 GPU 장착 가능, ZimaOS 앱스토어로 Jellyfin·Nextcloud·Immich 원클릭 설치까지 정리."
date: 2026-06-28
tags: [홈서버, 자체호스팅, NAS설정, HomeLab, Docker]
comments: true
share: true
---

![ZimaCube 2 홈서버 NAS 데스크탑 서버](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=800&q=80)

ZimaCube 2는 Synology나 QNAP과 결정적으로 다른 게 하나 있다. PCIe 슬롯이 두 개 열려 있다. GPU를 꽂으면 된다. 로컬 LLM을 돌리거나 미디어 트랜스코딩 전용 카드를 달거나, 10GbE 카드를 추가하거나 — NAS인데 홈랩처럼 확장된다. 2026년 4월 IceWhale Technology가 출시했고, 지금 해외에서 리뷰가 쏟아지고 있는 제품이다.

## 환경 정보

| 항목 | 값 |
|------|----|
| 제품명 | ZimaCube 2 (IceWhale Technology) |
| 출시일 | 2026년 4월 29일 |
| CPU (기본) | Intel Core i3-1215U (6코어, 12세대) |
| CPU (Pro) | Intel Core i5-1235U (10코어, 12세대) |
| 베이 구성 | SATA 6베이 + M.2 NVMe 4슬롯 |
| 네트워크 | 듀얼 2.5GbE (기본) / 10GbE 온보드 (Pro) |
| 연결 | Thunderbolt 4 (USB4) 듀얼 포트 |
| PCIe | PCIe 4.0 × 1 + PCIe 3.0 × 1 (오픈 슬롯) |
| OS | ZimaOS (사전 설치, Docker 네이티브) |
| 가격 | $799 (기본) / $1,299 (Pro) / $2,499 (Creator) |

---

## 전통적인 NAS와 뭐가 다른가

Synology DS923+를 쓰다 보면 한계가 딱 하나 있다. DSM 밖으로 못 나간다. Proxmox를 올리고 싶어도 안 되고, 커스텀 Linux를 쓰고 싶어도 ARM 계열 CPU 특성상 제약이 크다.

ZimaCube 2는 x86 아키텍처라서 아무 OS나 올라간다. ZimaOS 말고도 TrueNAS, Proxmox, Unraid, Ubuntu Server, 심지어 Windows까지 설치 가능하다. 제조사가 "Open NAS"라고 부르는 이유가 여기 있다.

근데 처음 셋업은 ZimaOS로 시작하는 게 맞다. 학습 곡선이 거의 없고, 앱스토어에서 버튼 하나로 Jellyfin·Nextcloud·Immich·Home Assistant 다 올라간다.

## ZimaOS 앱스토어 — Docker를 몰라도 된다

ZimaOS의 핵심은 앱스토어다. 500개 이상의 Docker 앱을 GUI로 설치한다. 내부에서는 Docker 컨테이너가 뜨는 건데, 사용자는 명령줄을 칠 필요가 없다.

직접 써보면 이렇다:

1. 브라우저에서 ZimaOS 대시보드 접속 (로컬 IP 또는 `find.zimaspace.com`)
2. App Store → 검색 → Install
3. 15분 안에 Jellyfin 미디어 서버 뜸

익숙한 사람이라면 Custom App 기능으로 Docker Compose YAML을 붙여넣으면 된다. ZimaOS가 알아서 컨테이너를 구성한다.

```yaml
# Custom App 예시 — Vaultwarden 비밀번호 서버
version: "3"
services:
  vaultwarden:
    image: vaultwarden/server:latest
    ports:
      - "8080:80"
    volumes:
      - /DATA/vaultwarden:/data
    restart: unless-stopped
```

이걸 Custom App 창에 붙여넣으면 그냥 설치된다. 터미널 없어도 된다.

## 세 가지 모델 중 뭘 사야 하나

모델이 세 가지인데, 용도가 명확히 갈린다.

**기본 ($799)** — Core i3-1215U, 듀얼 2.5GbE  
파일 서버 + 미디어 서버 + 자체호스팅 앱 정도면 충분하다. 트랜스코딩 없는 Jellyfin (Direct Play 위주), Nextcloud, Vaultwarden, Immich — 전부 이 모델로 돌아간다.

**Pro ($1,299)** — Core i5-1235U, 10GbE 온보드  
10GbE가 온보드로 들어간다는 게 핵심이다. NAS끼리 데이터 이동이 많거나, 4K 영상 트랜스코딩을 자주 한다면 Pro를 선택해야 한다. 발열 제어도 기본 모델보다 나아졌다는 리뷰가 많다.

**Creator ($2,499)** — Pro + NVIDIA RTX PRO 2000 GPU 사전 설치  
이건 로컬 AI용이다. Ollama 올려서 Llama 3 돌리거나, Stable Diffusion 이미지 생성, 영상 AI 편집 — GPU 없이는 못 하는 작업들을 집에서 하고 싶다면 이 모델이다.

![PCIe 확장 슬롯 홈서버 GPU 장착](https://images.unsplash.com/photo-1518770660439-4636190af475?w=800&q=80)

## PCIe 확장 — 나중에 GPU 추가하는 법

Creator를 안 사도 나중에 GPU를 추가할 수 있다. PCIe 4.0 슬롯이 열려 있기 때문이다.

실제 절차:

```
1. ZimaOS 종료 후 전원 OFF
2. PCIe 4.0 슬롯에 GPU 삽입 (로우 프로파일 카드 확인 필수)
3. 전원 ON → ZimaOS 앱스토어에서 Ollama 설치
4. Ollama 설정에서 GPU 활성화
```

주의할 점이 있다. 케이스 내부 공간이 풀사이즈 GPU를 수용하지 못한다. 로우 프로파일(슬림) 카드만 들어간다. RTX 4060 로우 프로파일이나 Arc A380 로우 프로파일 같은 걸 알아봐야 한다. 일반 RTX 4090 넣는다고 생각하면 안 된다.

## 삽질했던 부분

**초기 설정 언어 문제**: ZimaOS가 영문 인터페이스다. 한국어 지원이 없다. 직관적이라서 크게 문제는 아닌데, 처음 접속 시 언어 선택지가 없어서 당황할 수 있다.

**로우 프로파일 GPU 호환성**: 공식 문서에 호환 카드 목록이 따로 없다. 커뮤니티 포럼에서 사례를 찾아야 한다. Arc A310 로우 프로파일은 동작 확인됐다는 글이 있는데, RTX 계열은 모델마다 다르다.

**Synology 대비 앱 생태계**: ZimaOS 앱스토어가 Docker 기반이라 앱 종류는 많은데, Synology DSM처럼 제조사가 관리하는 공식 앱이 아니라 커뮤니티 이미지다. 업데이트·보안 패치를 직접 챙겨야 한다는 점은 감안해야 한다.

**소음**: 1세대 ZimaCube보다 많이 조용해졌다는 리뷰가 많다. 기본 모델은 팬 소음이 거의 없다는 평. Pro는 10GbE 칩 발열 때문에 팬이 약간 돈다.

## 한 줄 정리

ZimaCube 2는 "NAS인데 홈서버이기도 하고, 원하면 AI 서버이기도 한" 물건이다. Synology DSM 생태계가 필요하거나 완전히 관리형 환경을 원하면 DS923+ 같은 전통 NAS가 맞고, 나중에 확장하거나 로컬 AI 실험까지 생각하고 있다면 ZimaCube 2가 현실적인 선택지다.

---

다음엔 ZimaCube 2에 ZimaOS 대신 Proxmox를 올리는 방법을 다룬다.

**참고 링크**
- [ZimaCube 2 공식 페이지](https://shop.zimaspace.com/products/zimacube-2-personal-cloud-nas)
- [ZimaCube 2 리뷰 — It's FOSS](https://itsfoss.com/zimacube-2-review/)
- [ZimaCube 2 리뷰 — NAS Compares](https://nascompares.com/review/zimacube-2-review/)
- [ZimaOS Docker 앱 가이드](https://shop.zimaspace.com/blogs/zima-campaign-hub/nas-101-first-docker-app-zimaos)

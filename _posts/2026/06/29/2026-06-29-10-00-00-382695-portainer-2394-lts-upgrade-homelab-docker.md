---
layout: post
title: "Portainer 2.39.4 LTS 업그레이드 — CVE 패치 포함 홈랩 Docker 관리 서버 최신화"
description: "Portainer CE 2.39.4 LTS가 2026년 6월 25일 출시됐다. CVE 보안 패치, Docker 28/29·Kubernetes 1.33~1.35 지원 포함. Docker Standalone 업그레이드 방법과 agent 동기화 주의사항을 정리했다."
date: 2026-06-29
tags: [Docker, 홈서버, 자체호스팅, HomeLab, 홈랩]
comments: true
share: true
---

# Portainer 2.39.4 LTS 업그레이드 — CVE 패치 포함 홈랩 Docker 관리 서버 최신화

![홈랩 서버 Docker 컨테이너 관리 화면](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=800&q=80)

Portainer CE 2.39.4 LTS가 2026년 6월 25일에 출시됐다. CVE 보안 취약점이 패치된 버전이라, 홈서버에서 Docker를 운영 중이라면 그냥 넘어가면 안 된다. Docker 이미지 교체로 5분이면 끝난다.

## 환경 정보

| 항목 | 버전 |
|------|------|
| Portainer CE | 2.39.4 LTS |
| Docker Engine | 28.5.1 / 29.5.2 |
| 지원 아키텍처 | x86_64, ARM64 |
| Kubernetes (지원) | 1.33, 1.34, 1.35 |
| Podman | 5.8.2 |

## 왜 지금 올려야 하는가

2.39.4는 LTS(Long Term Support) 버전이다. STS(Short Term Support) 계열의 실험적 기능은 없지만, 대신 여러 CVE가 패치됐다. CVE가 뭔지 몰라도 상관없다—보안 구멍이 막혔다는 뜻이다.

더 중요한 건 이번에 Docker 28과 29 버전을 공식 지원하기 시작했다는 거다. `docker --version`이 이미 28.x나 29.x인데 Portainer는 구버전으로 쓰고 있다면, 일부 컨테이너 기능이 정상 동작하지 않을 수 있다. 특히 Synology Container Manager 대신 Portainer를 설치해서 쓰는 경우라면 체크가 필요하다.

LTS라는 이름이 붙었다는 건, Portainer 측에서 "이 버전은 프로덕션 환경에서 써도 된다"고 보증하는 거다. 홈랩이라도 24시간 돌아가는 서비스라면 LTS 기준으로 맞추는 게 맞다.

## 업그레이드 전 백업

Portainer UI에서 Settings → Backup Portainer 메뉴가 있다. 파일 하나 받아두면 된다. 스택·환경·사용자 설정이 전부 포함된다. 2분이면 된다.

```bash
# 현재 버전 확인
docker exec portainer cat /portainer/portainer | strings | grep -i "version" | head -5
# 또는 UI에서 왼쪽 하단 Help → About Portainer
```

## Docker Standalone 업그레이드

3단계다. 기존 컨테이너 중지 → 이미지 교체 → 재시작.

```bash
# 1. 중지·삭제 (순서 중요: stop 먼저, rm 다음)
docker stop portainer
docker rm portainer

# 2. 최신 이미지 Pull
docker pull portainer/portainer-ce:2.39.4

# 3. 볼륨 유지하면서 재시작
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:2.39.4
```

`portainer_data` 볼륨을 그대로 유지하는 게 핵심이다. 이 볼륨에 스택, 환경, 사용자 설정이 전부 들어있다. 볼륨을 빠뜨리면 초기화되니 주의.

업그레이드 후 `https://NAS_IP:9443`으로 접속하면 기존 계정 그대로 로그인된다.

![Portainer 컨테이너 스택 관리 화면](https://images.unsplash.com/photo-1544197150-b99a580bb7a8?w=800&q=80)

## 삽질했던 부분

**Agent 버전도 같이 올려야 한다.** 이걸 빠뜨리면 에이전트 연결이 끊긴다.

Portainer를 여러 서버에 분산해서 Edge Agent 또는 Agent 모드로 쓰는 경우, 각 서버의 agent도 동일한 2.39.4로 올려야 한다. 공식 문서에서 "모든 agent가 동일 버전이어야 한다"고 명시하고 있다. 이걸 모르고 메인 서버만 올렸다가 연결 에러 보고 30분 낭비했다.

```bash
# agent 업데이트 (분산 환경에서만 해당)
docker stop portainer_agent
docker rm portainer_agent
docker pull portainer/agent:2.39.4
docker run -d \
  -p 9001:9001 \
  --name portainer_agent \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:2.39.4
```

단일 서버에서만 쓴다면 agent는 없어도 된다.

**`docker stop` 없이 `docker rm`만 하면 에러 난다.** 순서는 반드시 stop → rm이다. 아직 실행 중인 컨테이너는 삭제가 안 된다. 급하게 하다 보면 이 순서를 바꾸게 되는데, 그러면 `Error response from daemon: You cannot remove a running container` 메시지가 뜬다.

**포트 번호 확인.** 원래 9000번 포트(HTTP)로 접속하던 경우, 최근 버전부터 HTTPS 기본 포트가 9443으로 바뀌었다. 북마크가 9000으로 돼있으면 접속이 안 된다.

## 한 줄 정리

볼륨 유지하면서 이미지 교체하면 5분 만에 끝난다. agent 있으면 같이 올리는 거 잊지 말 것.

---

다음엔 Portainer 스택 기능으로 Docker Compose 서비스들을 체계적으로 관리하는 방법을 다룬다.

**참고 링크**
- [Portainer 2.39.4 LTS 릴리스 노트](https://github.com/portainer/portainer/releases/tag/2.39.4)
- [Portainer 공식 업그레이드 문서 (Docker Standalone)](https://docs.portainer.io/start/upgrade/docker)
- [Portainer 2.39 LTS 소개 블로그](https://www.portainer.io/blog/new-portainer-2-39-release)

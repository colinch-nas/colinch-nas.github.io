---
layout: post
title: "Synology NAS Docker 설치 — Container Manager로 컨테이너 운영하기"
description: "Synology DSM 7.2 기준 Container Manager(구 Docker) 설치 방법과 기본 컨테이너 운영 방법을 정리했다. docker-compose 파일로 서비스를 올리는 방법까지 포함."
date: 2026-06-26
tags: [Synology, Docker, NAS설정, 자체호스팅, ContainerManager]
comments: true
share: true
---

# Synology NAS Docker 설치 — Container Manager로 컨테이너 운영하기

![Synology NAS Docker 컨테이너 운영](https://images.unsplash.com/photo-1605745341112-85968b19335b?w=800&q=80)

Synology NAS에서 Docker를 쓸 수 있다는 게 처음엔 반신반의했다. NAS 박스에서 Nextcloud, Jellyfin, Vaultwarden을 돌린다는 게 가능한가 싶었는데 실제로 잘 된다. DSM 7.2부터는 Docker 앱 이름이 "Container Manager"로 바뀌었다. 설치와 기본 사용법을 정리한다.

## 환경 정보

| 항목 | 내용 |
|------|------|
| NAS 모델 | Synology DS923+ |
| DSM 버전 | DSM 7.2.2 |
| Container Manager | v20.10.x (DSM 패키지 버전) |
| 지원 모델 | x86_64 아키텍처 NAS (ARM 기반 일부 모델은 미지원) |

ARM 기반 NAS(J 시리즈 일부 등)는 Container Manager를 지원하지 않는다. DSM → 패키지 센터에서 Container Manager가 검색되면 지원 모델이다.

## Container Manager 설치

`DSM → 패키지 센터 → 검색: Container Manager → 설치`

설치 완료 후 메인 메뉴에 Container Manager 아이콘이 생긴다.

Docker Hub 이미지를 직접 받거나, docker-compose 파일로 스택을 올릴 수 있다.

## 기본 개념 — Container Manager UI 구성

- **컨테이너**: 실행 중인 인스턴스. 시작·중지·삭제 가능.
- **이미지**: 컨테이너를 만들기 위한 베이스. Docker Hub에서 다운로드.
- **레지스트리**: Docker Hub 등 이미지 저장소 연결.
- **프로젝트**: docker-compose.yml 기반으로 여러 컨테이너를 묶어 관리.

UI에서 할 수 있지만, 복잡한 설정은 SSH에서 명령어로 직접 하는 게 편하다.

## SSH로 Docker 사용하기

Container Manager보다 더 유연하게 쓰려면 SSH를 활성화하고 직접 명령어를 쓴다.

`제어판 → 터미널 및 SNMP → SSH 서비스 활성화`

```bash
# SSH 접속 후 docker 버전 확인
docker --version

# 이미지 검색
docker search nginx

# 이미지 다운로드
docker pull nginx:latest

# 컨테이너 실행
docker run -d -p 8080:80 --name my-nginx nginx

# 실행 중인 컨테이너 확인
docker ps

# 컨테이너 중지
docker stop my-nginx

# 컨테이너 삭제
docker rm my-nginx
```

## docker-compose로 서비스 올리기

단일 컨테이너로 끝나는 서비스는 드물다. 대부분 DB + 앱 + 프록시 조합으로 여러 컨테이너가 함께 실행된다. docker-compose를 쓰면 한 파일로 이 구성을 관리할 수 있다.

예시: Portainer(컨테이너 관리 웹 UI) 설치

```yaml
# /volume1/docker/portainer/docker-compose.yml
version: '3'
services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    ports:
      - "9000:9000"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /volume1/docker/portainer/data:/data
```

```bash
# 파일 위치로 이동 후 실행
cd /volume1/docker/portainer
docker-compose up -d

# 상태 확인
docker-compose ps

# 로그 확인
docker-compose logs -f
```

`http://[NAS IP]:9000`으로 접속하면 Portainer 관리 화면이 나온다. Container Manager UI보다 더 상세한 컨테이너 관리가 가능하다.

## 볼륨 경로 — 이게 중요하다

Synology NAS의 메인 스토리지는 `/volume1/`이다. 컨테이너에서 데이터를 영구적으로 보관하려면 호스트 경로를 `/volume1/` 아래로 마운트해야 한다.

```yaml
volumes:
  - /volume1/docker/[서비스명]/data:/[컨테이너 내 경로]
```

컨테이너가 삭제되어도 `/volume1/docker/` 아래의 데이터는 남아 있다. 이 경로를 백업 대상에도 포함시켜야 한다.

## 삽질했던 부분

메모리가 적은 NAS(RAM 4GB 이하)에서 컨테이너를 너무 많이 띄우면 DSM 자체가 느려진다. 처음엔 욕심 부려서 5~6개를 동시에 실행했다가 NAS가 버벅였다. 실제로 필요한 것만 상시 실행하고, 가끔 쓰는 건 필요할 때만 켜는 식으로 운영한다.

컨테이너 시간대(timezone) 설정을 안 하면 로그 시간이 UTC로 찍혀서 나중에 헷갈린다. 환경 변수에 `TZ: Asia/Seoul`을 꼭 넣는다.

```yaml
environment:
  - TZ=Asia/Seoul
```

## 한 줄 정리

Container Manager 설치하고 `/volume1/docker/` 아래에 서비스별 폴더 만들어서 docker-compose로 관리하면 된다.

---

다음엔 Jellyfin을 NAS에 설치해서 집에서 넷플릭스처럼 영상 스트리밍하는 방법을 다룬다.

**참고 링크**
- [Synology Container Manager 공식 문서](https://kb.synology.com/ko-kr/DSM/help/ContainerManager/docker_desc)
- [Docker Hub](https://hub.docker.com)
- [Portainer 공식 사이트](https://www.portainer.io)

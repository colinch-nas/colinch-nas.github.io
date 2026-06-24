---
layout: post
title: "Synology Docker 설치 및 컨테이너 운영 — Container Manager 실전 가이드"
description: "Synology DSM 7.2에서 Container Manager(구 Docker)를 설치하고 실제로 컨테이너를 띄우는 방법을 정리했다. 이미지 다운로드부터 볼륨 마운트, 네트워크 설정까지."
date: 2026-06-06
tags: [Synology, Docker, DSM, 홈서버]
comments: true
share: true
---

# Synology Docker 설치 및 컨테이너 운영 — Container Manager 실전 가이드

![Synology NAS Docker 컨테이너 운영](https://images.unsplash.com/photo-1605745341112-85968b19335b?w=800&q=80)

Synology NAS에 Docker를 올리면 NAS 용도가 완전히 달라진다. 단순 파일 서버에서 Jellyfin, Nextcloud, Vaultwarden 같은 서비스를 직접 돌리는 홈서버로 바뀐다. DSM 7.2부터는 Docker 앱 이름이 **Container Manager**로 바뀌었다.

이전 글에서 [DSM 초기 보안 설정을 다뤘다]({% post_url 2026-06-05-10-00-00-012345-synology-dsm-initial-setup-security %}). 보안 설정 먼저 하고 이 글을 따라하는 걸 권장한다.

## 환경 정보

| 항목 | 버전/모델 |
|------|----------|
| NAS 모델 | Synology DS923+ |
| DSM 버전 | DSM 7.2.2-72806 Update 2 |
| Container Manager | 20.10.23-1455 |
| CPU 아키텍처 | x86_64 (AMD Ryzen R1600) |

주의: Synology NAS 중 ARM 기반 모델(DS120j, DS220j 등)은 Docker를 지원하지 않는다. 패키지 센터에서 Container Manager가 보이지 않으면 ARM 모델일 가능성이 높다.

## Container Manager 설치

**패키지 센터 → 모두 → Container Manager** 검색 후 설치. 1분도 안 걸린다.

설치 후 DSM 메인 화면에 Container Manager 아이콘이 생긴다. 열면 Docker Hub에서 이미지를 검색하고 컨테이너를 만드는 GUI가 있다.

## 이미지 다운로드

**Container Manager → 레지스트리** 탭에서 Docker Hub 이미지를 검색할 수 있다. 예를 들어 `nginx`를 검색하면 공식 이미지가 나온다.

근데 솔직히 GUI보다 SSH로 직접 docker pull 하는 게 더 편하다. SSH 켜는 방법은 **제어판 → 터미널 및 SNMP → SSH 서비스 활성화**.

```bash
# SSH 접속 후 Docker 이미지 pull
sudo docker pull nginx:latest
sudo docker pull jellyfin/jellyfin:latest

# 이미지 목록 확인
sudo docker images
```

## 볼륨 마운트 구조 이해

Synology에서 Docker 볼륨 설정할 때 경로 매핑이 헷갈린다. NAS의 파일 시스템 경로와 컨테이너 내부 경로를 정확히 매핑해야 한다.

```bash
# NAS 기본 공유 폴더 경로
/volume1/    # 첫 번째 볼륨 (설치된 드라이브)
/volume1/docker/    # Docker 설정 파일 보관 권장 위치
/volume1/media/     # 미디어 파일 위치 예시
```

실제 nginx 컨테이너 실행 예시:

```bash
sudo docker run -d \
  --name=nginx \
  -p 8080:80 \
  -v /volume1/docker/nginx/html:/usr/share/nginx/html \
  -v /volume1/docker/nginx/conf:/etc/nginx/conf.d \
  --restart=unless-stopped \
  nginx:latest
```

`--restart=unless-stopped` 옵션을 빠뜨리면 NAS 재시작할 때마다 컨테이너를 수동으로 다시 켜야 한다. 처음엔 이걸 몰라서 재부팅 후에 왜 서비스가 안 되는지 한참 헤맸다.

## docker-compose 사용

여러 컨테이너를 함께 관리할 때는 docker-compose가 편하다. Container Manager에서 **프로젝트** 탭을 열면 docker-compose.yml을 붙여넣어 실행할 수 있다.

![Container Manager 프로젝트 탭](https://images.unsplash.com/photo-1544197150-880d4212a0c4?w=800&q=80)

```yaml
# /volume1/docker/nginx/docker-compose.yml
version: "3.9"
services:
  nginx:
    image: nginx:latest
    container_name: nginx
    ports:
      - "8080:80"
    volumes:
      - /volume1/docker/nginx/html:/usr/share/nginx/html
    restart: unless-stopped
```

SSH에서 직접 실행하는 방법:

```bash
cd /volume1/docker/nginx
sudo docker compose up -d

# 로그 확인
sudo docker compose logs -f

# 중지
sudo docker compose down
```

## 네트워크 설정

컨테이너 간 통신이 필요할 때는 같은 Docker 네트워크에 묶어야 한다.

```bash
# 커스텀 네트워크 생성
sudo docker network create mynetwork

# 컨테이너를 네트워크에 추가
sudo docker run -d --network=mynetwork --name=app1 nginx:latest
sudo docker run -d --network=mynetwork --name=app2 nginx:latest

# 같은 네트워크 안에서는 컨테이너명으로 통신 가능
# app1에서 app2로: http://app2:80
```

## 삽질했던 부분

**권한 문제**: NAS에서 Docker 볼륨으로 마운트한 폴더에 컨테이너가 쓰기를 못 하는 경우가 있다. 해결 방법은 폴더 권한을 확인하거나, 컨테이너 실행 시 `-e PUID=1026 -e PGID=100` 같은 환경변수로 사용자를 맞춰주는 것이다. PUID/PGID 값은 NAS에서 `id 사용자명` 명령으로 확인할 수 있다.

```bash
# NAS에서 현재 사용자 ID 확인
id admin
# 출력: uid=1026(admin) gid=100(users) ...
```

**메모리 부족**: NAS RAM이 4GB 이하라면 컨테이너 여러 개 동시 운영 시 스왑이 부족해서 느려진다. DS923+는 기본 8GB에 최대 32GB 확장 가능하니 여유롭지만, 저가형 모델은 조심해야 한다.

## 한 줄 정리

Container Manager 설치하고 볼륨 마운트와 `--restart=unless-stopped` 두 가지만 제대로 이해하면, 이후 어떤 서비스든 올리는 방법은 동일하다.

---
다음엔 [Synology NAS에 Jellyfin을 올려서 미디어 서버를 구축하는 법]({% post_url 2026-06-07-10-00-00-037035-synology-jellyfin-media-server-setup %})을 다룬다.

**참고 링크**
- [Synology Container Manager 공식 문서](https://kb.synology.com/ko-kr/DSM/help/ContainerManager/docker_desc)
- [Docker 공식 문서](https://docs.docker.com)
- [LinuxServer.io Docker 이미지 모음](https://fleet.linuxserver.io) — PUID/PGID 지원 이미지 표준

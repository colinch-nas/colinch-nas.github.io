---
layout: post
title: "Nextcloud AIO v13.3.1 Synology NAS 설치 - DSM에서 포트 충돌 없이 운영하기"
description: "Nextcloud AIO v13.3.1을 Synology DSM 7.2 Docker에 설치하고, APACHE_PORT 11000과 DSM 역방향 프록시로 외부 접속하는 설정을 정리한다."
date: 2026-08-25
tags: [Nextcloud, Synology, Docker, 자체호스팅, NAS설정, 역방향프록시]
comments: true
share: true
---

![Synology NAS에서 Nextcloud AIO를 운영하는 홈서버](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

Nextcloud를 Synology NAS에 새로 올린다면 개별 PHP·MariaDB 컨테이너를 따로 조립하기보다 Nextcloud AIO(All-in-One)를 쓰는 편이 관리가 덜 번거롭다. 2026년 8월 기준 AIO v13.3.1은 자동 도메인·DNS 설정 기능도 추가됐지만, DSM에서 443 포트를 이미 쓰고 있다면 `APACHE_PORT=11000`과 역방향 프록시를 먼저 정해야 한다.

## 확인한 환경

| 항목 | 기준 |
|---|---|
| NAS | Synology DS923+ |
| 운영체제 | DSM 7.2.x, Container Manager |
| AIO | v13.3.1 계열, `latest` 이미지 |
| 데이터 경로 | `/volume1/docker/nextcloud-data` |
| 외부 주소 | `cloud.example.com` |

Nextcloud는 2026년 8월에도 Hub 25 Autumn 32.0.14, 26 Winter 33.0.8, 26 Spring 34.0.3 유지보수 업데이트를 배포했다. 새 설치만 보고 끝내지 말고 AIO 화면의 백업과 업데이트 상태까지 확인해야 한다. 자동 업데이트를 믿고 NAS 스냅샷을 생략하면 복구 지점이 사라진다.

## 기존 컨테이너와 포트부터 확인

DSM의 Container Manager에서 `nextcloud`, `mariadb`, `nginx` 컨테이너를 먼저 확인한다. 기존 글처럼 이미 운영 중인 Nextcloud가 있다면 바로 AIO를 같은 폴더에 덮어쓰면 안 된다. 파일과 데이터베이스를 별도 백업한 뒤, 테스트용 공유 폴더에서 AIO를 먼저 띄우는 방식이 안전하다.

DSM에서 SSH를 켠 뒤, 8080은 AIO 관리 화면, 11000은 DSM 역방향 프록시가 연결할 내부 Apache 포트로 사용한다. 443을 AIO 컨테이너에 직접 매핑하지 않는 것이 핵심이다.

```bash
sudo docker run \
  --init \
  --sig-proxy=false \
  --name nextcloud-aio-mastercontainer \
  --restart always \
  --publish 8080:8080 \
  --env APACHE_PORT=11000 \
  --env APACHE_IP_BINDING=0.0.0.0 \
  --env NEXTCLOUD_DATADIR="/volume1/docker/nextcloud-data" \
  --volume nextcloud_aio_mastercontainer:/mnt/docker-aio-config \
  --volume /var/run/docker.sock:/var/run/docker.sock:ro \
  ghcr.io/nextcloud-releases/all-in-one:latest
```

명령어에서 `NEXTCLOUD_DATADIR`는 NAS의 실제 공유 폴더 경로로 바꾼다. 폴더를 미리 만들고 Docker 실행 계정이 접근할 수 있는지 확인한다. AIO가 이미 생성된 뒤 이 값을 바꾸면 기존 데이터가 자동으로 이동하지 않는다.

## AIO 초기 화면과 DSM 역방향 프록시

브라우저에서 `https://NAS주소:8443`이 아니라 우선 `https://NAS주소:8080`으로 AIO 관리 화면을 연다. 브라우저 경고가 나오면 인증서가 아직 없는 초기 화면이므로 비밀번호를 입력하고, AIO가 요구하는 도메인에는 `cloud.example.com`을 넣는다.

공유기에서 80·443을 NAS로 전달하고 DNS의 `cloud.example.com`이 집의 공인 IP를 가리키게 한다. 그다음 DSM 제어판의 로그인 포털 → 고급 → 역방향 프록시에서 다음 규칙을 만든다.

| 구분 | 값 |
|---|---|
| 소스 프로토콜/호스트/포트 | HTTPS / cloud.example.com / 443 |
| 대상 프로토콜/호스트/포트 | HTTP / 127.0.0.1 / 11000 |
| WebSocket | 활성화 |

DSM 인증서 메뉴에서 `cloud.example.com`용 Let's Encrypt 인증서를 발급하고, 역방향 프록시 규칙에 연결한다. 외부에서 `https://cloud.example.com`을 열어 로그인 화면이 나오면 성공이다. 11000 포트를 공유기에 직접 개방하면 HTTPS 보호와 DSM 프록시를 우회하므로 열지 않는다.

## 여기서 막혔던 지점

가장 흔한 오류는 443 포트 충돌이다. DSM 관리 화면이나 다른 서비스가 이미 443을 사용하면 AIO 도메인 검사에서 실패한다. 이때 AIO의 도메인 검사를 무조건 끄기보다, DNS가 현재 공인 IP를 가리키는지와 공유기의 443 전달 대상을 먼저 확인한다. 내부 Wi-Fi에서만 테스트하면 헤어핀 NAT 때문에 실패할 수 있으니 휴대전화 데이터로도 접속한다.

또 하나는 데이터 폴더를 `/volume1/docker/nextcloud-data`처럼 호스트 경로로 지정해 놓고 Hyper Backup 대상에서 빼는 실수다. AIO가 정상 실행돼도 백업되지 않으면 자체 호스팅의 장점이 줄어든다.

## 짧은 점검표

- AIO 관리 화면 비밀번호를 별도 보관했는가
- NAS 방화벽에서 8080·11000을 외부에 공개하지 않았는가
- DSM 인증서 만료일과 도메인 DNS를 확인했는가
- Nextcloud 데이터와 AIO 설정 볼륨을 함께 백업하는가
- Hub 유지보수 업데이트 후 웹·모바일 동기화를 한 번 테스트했는가

AIO는 설치 구성요소를 줄여 주지만 백업까지 대신해 주지는 않는다. Synology에서는 443을 DSM 역방향 프록시에 맡기고 AIO는 11000 내부 포트로 제한하는 구성이 관리와 보안 사이의 균형이 좋았다.

참고: [Nextcloud 2026년 8월 유지보수 업데이트](https://nextcloud.com/fr/blog/august-updates-for-nextcloud-hub-25-autumn-26-winter-26-spring/), [Nextcloud AIO 공식 저장소](https://github.com/nextcloud/all-in-one), [AIO 역방향 프록시 문서](https://github.com/nextcloud/all-in-one/blob/main/reverse-proxy.md)

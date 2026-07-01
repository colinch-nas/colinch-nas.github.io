---
layout: post
title: "Nextcloud 34 업그레이드 - NAS Docker에서 한 번에 올리지 않는 절차"
description: "2026년 6월 25일 공개된 Nextcloud 34.0.1 기준으로 NAS Docker 설치본을 안전하게 업그레이드하는 백업, 태그 변경, 검증 순서."
date: 2026-07-01
tags: [Nextcloud, Docker, 자체호스팅, NAS설정]
comments: true
share: true
---
![Nextcloud NAS Docker 업그레이드](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)

Nextcloud를 NAS Docker로 운영 중이면 `latest`로 바로 올리지 말고 현재 메이저 버전에서 한 단계씩 올리는 게 맞다. 2026년 6월 25일 기준 Nextcloud 최신 유지보수 릴리스는 34.0.1이다. 기존에 이 블로그에서 만든 [Nextcloud 설치 구성]({% post_url 2026-06-14-10-00-00-123450-nextcloud-install-self-hosted-cloud %})은 `nextcloud:30-apache` 기준이라 34로 바로 뛰면 앱 호환성, DB 마이그레이션, PHP 설정에서 삽질할 가능성이 크다.

문제는 Docker라서 더 쉽다고 착각하는 데서 생긴다. 이미지 태그만 `nextcloud:34-apache`로 바꾸고 `docker compose up -d`를 치면 컨테이너는 뜰 수 있다. 하지만 Nextcloud 내부 업그레이드는 웹 앱과 데이터베이스 스키마를 같이 바꾼다. 중간 메이저를 건너뛰면 복구 화면도 못 보고 유지보수 모드에 갇힐 수 있다.

기준 구성은 이렇게 잡았다.

| 항목 | 값 |
|---|---|
| NAS | Synology DS923+ |
| DSM | 7.2.2 또는 7.4 |
| 현재 Nextcloud | 30-apache 또는 33-apache |
| 목표 Nextcloud | 34.0.1-apache |
| DB | MariaDB 10.11 |

업그레이드 전에는 파일과 DB를 따로 백업한다. 이걸 빼먹으면 롤백이 아니라 재설치가 된다.

```bash
cd /volume1/docker/nextcloud
docker compose down
tar czf nextcloud-db-$(date +%Y%m%d).tgz db
tar czf nextcloud-html-$(date +%Y%m%d).tgz html
tar czf nextcloud-data-config-$(date +%Y%m%d).tgz /volume1/data/nextcloud
```

DB를 Docker 볼륨으로 만들었다면 볼륨 이름을 확인해 별도로 백업한다.

```bash
docker volume ls | grep nextcloud
docker run --rm -v nextcloud_db:/var/lib/mysql -v "$PWD":/backup alpine tar czf /backup/nextcloud-db-volume.tgz /var/lib/mysql
```

업그레이드는 태그를 고정해서 진행한다. `latest`는 편해 보이지만 어느 날 메이저 버전이 바뀌면 운영 서버가 테스트 서버가 된다.

```yaml
services:
  app:
    image: nextcloud:31-apache
```

30에서 시작했다면 31, 32, 33, 34 순서로 한 번씩 올린다. 각 단계마다 컨테이너를 올리고 웹 UI에 접속해 업그레이드 완료를 확인한다.

```bash
docker compose pull
docker compose up -d
docker logs -f nextcloud
```

업그레이드가 끝난 뒤에는 `occ`로 상태를 본다.

```bash
docker exec -u www-data nextcloud php occ status
docker exec -u www-data nextcloud php occ db:add-missing-indices
docker exec -u www-data nextcloud php occ maintenance:repair --include-expensive
```

여기서 삽질한 지점은 앱이다. Calendar, Contacts 같은 기본 앱은 대체로 따라오지만, 외부 저장소나 미리보기 생성 앱은 메이저 업그레이드 직후 비활성화되는 경우가 있다. 관리자 화면의 `앱` 메뉴에서 비활성화된 앱을 바로 켜기보다, 로그에 `not compatible`이 찍히는지 확인한다.

```bash
docker logs nextcloud --tail=200 | grep -i error
```

리버스 프록시를 쓰는 경우도 확인해야 한다. Nginx Proxy Manager나 Cloudflare Tunnel 뒤에 있으면 `trusted_domains`, `overwrite.cli.url`, `overwriteprotocol` 값이 업그레이드 후에도 남아 있는지 본다.

```bash
docker exec -u www-data nextcloud php occ config:system:get trusted_domains
docker exec -u www-data nextcloud php occ config:system:get overwrite.cli.url
```

주의할 점은 세 가지다. 자동 업데이트 컨테이너로 Nextcloud를 올리지 않는다. DB 백업 없이 메이저 버전을 넘기지 않는다. 사진 원본이나 가족 문서가 들어 있는 서버라면 34.0.1이 최신이라고 바로 운영 서버에 적용하지 말고, 동일한 Compose 파일을 복제해 테스트 컨테이너에서 한 번 마이그레이션을 돌린다.

짧게 정리하면 이렇다. 2026년 7월 1일 기준 Nextcloud 34.0.1은 최신 유지보수 릴리스다. NAS Docker에서는 `latest` 대신 `31-apache`, `32-apache`, `33-apache`, `34-apache`처럼 태그를 고정한다. 각 단계마다 웹 업그레이드 완료, `occ status`, 로그, 리버스 프록시 설정을 확인하면 복구 가능한 업그레이드가 된다.

출처: [Nextcloud server changelog](https://nextcloud.com/changelog/), [Nextcloud official Docker image](https://hub.docker.com/_/nextcloud/)

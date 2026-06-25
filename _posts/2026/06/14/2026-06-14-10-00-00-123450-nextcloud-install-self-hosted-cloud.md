---
layout: post
title: "Nextcloud 설치 — 구글 드라이브 대신 내 서버에 클라우드 만들기"
description: "Synology NAS 또는 VPS에 Docker로 Nextcloud를 설치하고 구글 드라이브를 대체하는 자체 호스팅 클라우드를 구축하는 방법. MariaDB 연동과 Nextcloud Hub 기능 설정 포함."
date: 2026-06-14
tags: [Nextcloud, 자체호스팅, Docker, 홈서버]
comments: true
share: true
---
![Nextcloud 자체 호스팅 클라우드 서버](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)

구글 드라이브 15GB 무료가 언제 꽉 찰지 모르고, 요금제 올리면 한 달에 몇 달러씩 나간다. NAS가 있으면 Nextcloud로 사실상 무제한 클라우드를 집에 만들 수 있다. 파일 동기화, 캘린더, 연락처, 사진 백업까지 된다.

## 환경 정보

| 항목 | 버전/모델 |
|------|----------|
| NAS 모델 | Synology DS923+ |
| Nextcloud 버전 | 30.x |
| Docker 이미지 | nextcloud:30-apache |
| 데이터베이스 | MariaDB 10.11 |

## docker-compose로 Nextcloud 설치

Nextcloud는 단독 컨테이너로도 돌릴 수 있지만 MariaDB를 별도 컨테이너로 쓰는 게 성능이 낫다. SQLite는 사용자가 많아지면 느려진다.

```yaml
# /volume1/docker/nextcloud/docker-compose.yml
version: "3.9"

services:
  db:
    image: mariadb:10.11
    container_name: nextcloud-db
    restart: unless-stopped
    command: --transaction-isolation=READ-COMMITTED --log-bin=binlog --binlog-format=ROW
    environment:
      - MYSQL_ROOT_PASSWORD=rootpassword
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_PASSWORD=nextcloudpassword
    volumes:
      - /volume1/docker/nextcloud/db:/var/lib/mysql

  app:
    image: nextcloud:30-apache
    container_name: nextcloud
    restart: unless-stopped
    ports:
      - "8080:80"
    environment:
      - MYSQL_HOST=db
      - MYSQL_DATABASE=nextcloud
      - MYSQL_USER=nextcloud
      - MYSQL_PASSWORD=nextcloudpassword
      - NEXTCLOUD_ADMIN_USER=admin
      - NEXTCLOUD_ADMIN_PASSWORD=adminpassword
      - NEXTCLOUD_TRUSTED_DOMAINS=192.168.1.100 mynextcloud.example.com
    volumes:
      - /volume1/docker/nextcloud/html:/var/www/html
      - /volume1/data/nextcloud:/var/www/html/data
    depends_on:
      - db
```

```bash
cd /volume1/docker/nextcloud
sudo docker compose up -d

# 로그 모니터링
sudo docker compose logs -f app
```

처음 실행 시 초기화에 2~3분 걸린다. 로그에서 `AH00558: apache2` 이후에 접속이 된다.

접속 주소: `http://NAS-IP:8080`

## 초기 설정 최적화

**PHP 메모리 제한 늘리기**:

```bash
# 컨테이너 내부 php.ini 수정
sudo docker exec -it nextcloud bash
echo "memory_limit=512M" >> /usr/local/etc/php/conf.d/nextcloud.ini
exit
sudo docker restart nextcloud
```

**캐시 설정 (Redis)**:

큰 파일이나 다수의 사용자를 위해 Redis 캐시를 추가한다.

```yaml
# docker-compose.yml에 추가
  redis:
    image: redis:alpine
    container_name: nextcloud-redis
    restart: unless-stopped

  # app 서비스에 추가
  app:
    # ...
    environment:
      - REDIS_HOST=redis
    depends_on:
      - db
      - redis
```

```php
# /volume1/docker/nextcloud/html/config/config.php에 추가
'memcache.local' => '\OC\Memcache\Redis',
'memcache.locking' => '\OC\Memcache\Redis',
'redis' => [
  'host' => 'redis',
  'port' => 6379,
],
```

## 파일 동기화 클라이언트

데스크탑과 모바일에 Nextcloud 클라이언트 앱을 설치하면 구글 드라이브처럼 자동 동기화된다.

- **PC**: [Nextcloud Desktop Client](https://nextcloud.com/install/#install-clients) 다운로드 설치
- **Android**: Play Store에서 Nextcloud 설치
- **iOS**: App Store에서 Nextcloud 설치

서버 주소 입력 시 HTTPS를 쓰려면 [외부 접속 설정]({% post_url 2026-06-08-10-00-00-049380-synology-ddns-letsencrypt-external-access %})이 먼저 돼 있어야 한다.

![Nextcloud 파일 관리 화면](https://images.unsplash.com/photo-1451187580094-2cfb76b7c9b7?w=800&q=80)

## 사진 자동 백업

Nextcloud 앱에서 카메라 업로드를 활성화하면 스마트폰 사진이 자동으로 Nextcloud로 백업된다. 구글 포토 대체로 쓸 수 있다.

단, 사진 뷰어가 구글 포토만큼 강력하지 않다. 얼굴 인식이나 장소 태깅 같은 기능은 없다. 순수 파일 저장 백업 용도로 생각하는 게 맞다. 사진 관리까지 원한다면 Immich가 낫다.

## 삽질했던 부분

**신뢰 도메인 설정**: 설치 후 외부 도메인으로 접속하면 `Access through untrusted domain` 오류가 난다. `NEXTCLOUD_TRUSTED_DOMAINS` 환경변수에 접속하려는 모든 도메인을 공백으로 구분해서 나열해야 한다.

```bash
# config.php를 직접 수정하는 방법
sudo docker exec -it nextcloud php /var/www/html/occ config:system:set trusted_domains 2 --value="mynextcloud.example.com"
```

**대용량 파일 업로드 실패**: 기본 PHP 설정은 업로드 파일 크기가 2MB로 제한돼 있다. `upload_max_filesize`와 `post_max_size`를 올려야 대용량 파일이 된다.

```bash
sudo docker exec -it nextcloud bash
echo "upload_max_filesize=10G" >> /usr/local/etc/php/conf.d/nextcloud.ini
echo "post_max_size=10G" >> /usr/local/etc/php/conf.d/nextcloud.ini
```

**데이터 폴더 권한**: `/var/www/html/data` 폴더의 소유자가 `www-data`(uid 33)여야 한다. NAS 볼륨에 마운트하면 권한이 맞지 않을 수 있다.

```bash
sudo chown -R 33:33 /volume1/data/nextcloud
```

## 한 줄 정리

MariaDB + Redis 조합으로 Nextcloud를 올리면 구글 드라이브 수준의 파일 동기화를 집 서버에서 운영할 수 있다.

---
다음엔 [Vaultwarden으로 비밀번호 관리 서버 직접 운영하기 — Bitwarden 자체 호스팅]({% post_url 2026-06-15-10-00-00-135795-vaultwarden-password-server-setup %})을 다룬다.

**참고 링크**
- [Nextcloud 공식 사이트](https://nextcloud.com)
- [Nextcloud Docker Hub](https://hub.docker.com/_/nextcloud)
- [Nextcloud 설정 파일 레퍼런스](https://docs.nextcloud.com/server/latest/admin_manual/configuration_server/config_sample_php_parameters.html)

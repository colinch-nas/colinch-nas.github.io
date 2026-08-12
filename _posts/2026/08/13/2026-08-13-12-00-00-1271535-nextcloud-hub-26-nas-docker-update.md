---
layout: post
title: "Nextcloud Hub 26 Spring 업데이트 - NAS Docker에서 안전하게 업그레이드하는 방법"
description: "Nextcloud Hub 26 Spring과 2026년 7월 유지보수 업데이트를 NAS Docker에 안전하게 적용하는 순서. 백업, maintenance mode, occ 점검, 롤백 기준까지 정리한다."
date: 2026-08-13
tags: [Nextcloud, Docker, 자체호스팅, NAS설정, 홈서버]
comments: true
share: true
---

![NAS에서 Nextcloud 자체 호스팅 서버를 업데이트하는 모습](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

Nextcloud를 NAS에서 Docker로 운영한다면 이번 업데이트는 컨테이너 이미지만 바꾸고 끝낼 일이 아니다. 2026년 6월 공개된 Nextcloud Hub 26 Spring은 Euro-Office, 가벼워진 UI, Deck의 Gantt 기능을 포함했고, 7월 24일에는 Hub 26 Spring 유지보수 업데이트도 배포됐다. 나는 실제 운영 서버에서 파일과 데이터베이스를 먼저 백업한 뒤, 점검 명령을 통과한 경우에만 이미지를 올리는 방식이 가장 안전하다고 본다.

## 업데이트 전에 확인할 것

Nextcloud의 메이저 버전 업데이트는 앱 호환성에서 문제가 생기기 쉽다. 특히 Calendar, Contacts, Office 연동 앱을 많이 설치했다면 자동 업데이트를 바로 누르지 않는 편이 낫다.

| 항목 | 확인 기준 |
|---|---|
| 현재 버전 | 관리자 설정 → 개요에서 25.x 또는 26.x 확인 |
| 백업 | `html`, `data`, MariaDB 볼륨을 모두 보관 |
| 여유 공간 | 업데이트 임시 파일을 고려해 최소 10GB |
| 외부 접속 | DSM 역방향 프록시의 도메인과 HTTPS 정상 여부 |

Nextcloud 공식 발표 기준 Hub 26 Spring에는 Collabora와 함께 Euro-Office가 Nextcloud Office 선택지로 추가됐다. 다만 NAS에서 문서 편집까지 쓸 계획이 없다면 Office 컨테이너까지 동시에 업데이트하지 않는 편이 장애 범위를 줄인다.

## Docker에서 안전하게 업데이트하기

아래 예시는 `/volume1/docker/nextcloud`에 `docker-compose.yml`이 있고 컨테이너 이름이 `nextcloud`인 Synology NAS 기준이다. 실제 경로와 컨테이너 이름은 `docker ps`로 먼저 확인한다.

```bash
cd /volume1/docker/nextcloud
docker compose ps
docker exec -u www-data nextcloud php occ status
docker exec -u www-data nextcloud php occ maintenance:mode --on
```

maintenance mode(점검 모드)를 켜면 접속 중인 사용자가 업데이트 중 파일을 만들지 못한다. 이 상태에서 파일과 DB를 각각 백업한다.

```bash
tar -czf /volume1/backup/nextcloud-html-$(date +%F).tar.gz html
tar -czf /volume1/backup/nextcloud-data-$(date +%F).tar.gz data
docker exec nextcloud-db mariadb-dump -u root -p'DB_ROOT_PASSWORD' nextcloud \
  > /volume1/backup/nextcloud-db-$(date +%F).sql
```

비밀번호를 명령어에 직접 남기는 방식은 쉘 기록에 노출될 수 있다. 실제 운영 환경에서는 `.env` 파일이나 DSM 비밀 저장 기능을 사용하고, 위 예시의 값은 반드시 교체한다.

백업 파일 크기와 생성 시각을 확인한 뒤 이미지를 갱신한다. `latest` 태그 대신 현재 테스트한 버전 태그를 고정하는 편이 롤백하기 쉽다.

```bash
docker compose pull nextcloud
docker compose up -d nextcloud
docker compose logs --tail=100 nextcloud
docker exec -u www-data nextcloud php occ upgrade
docker exec -u www-data nextcloud php occ maintenance:mode --off
docker exec -u www-data nextcloud php occ status
```

`occ upgrade`가 이미 완료됐다는 메시지를 내더라도 오류가 없는지 로그를 확인한다. 관리자 화면의 앱 목록에서 비활성화된 앱이 새로 생겼는지도 봐야 한다. 외부 도메인으로 접속했을 때 `untrusted domain`이 나오면 `config.php`의 `trusted_domains`를, HTTPS 경고가 나오면 `overwriteprotocol`과 역방향 프록시 헤더를 점검한다.

## 실패했을 때의 기준

업데이트 후 로그인, 파일 업로드, 모바일 동기화 중 하나라도 10분 안에 정상 동작하지 않으면 계속 고치려 들기보다 점검 모드를 유지한다. 앱 호환성 오류는 해당 앱을 비활성화한 뒤 재시도하고, DB 마이그레이션 오류나 파일 목록 이상이 있으면 컨테이너를 중지하고 백업 시점으로 되돌린다.

| 증상 | 우선 조치 |
|---|---|
| 화면이 계속 점검 중 | `maintenance:mode --off` 실행 여부 확인 |
| 파일 업로드 413 오류 | NAS 프록시와 PHP 업로드 제한 확인 |
| 앱이 사라짐 | `occ app:list`로 비활성화 앱 확인 |
| DB 오류 | 새 이미지 재시작보다 DB 백업 복원 우선 |

이번 업데이트에서 가장 크게 삽질한 지점은 기능 자체가 아니라 백업 범위였다. Nextcloud의 `html`만 복사하면 데이터베이스와 실제 파일 저장소가 빠질 수 있다. 최소한 HTML, data, DB를 한 세트로 보관하고, 백업 파일을 다른 디스크에도 복사해야 한다.

## 짧은 요점 정리

Nextcloud Hub 26 Spring은 NAS에서도 적용할 만하지만, 운영 서버에 바로 `pull`부터 하면 안 된다. 현재 버전과 앱 호환성을 확인하고, maintenance mode를 켠 뒤 HTML·data·DB를 백업한다. 업데이트 후에는 `occ upgrade`, 업로드, 모바일 동기화, 외부 HTTPS 접속까지 직접 테스트한다. 2026년 7월 유지보수 버전과 AIO v13.3.1의 자동 도메인 기능은 공식 릴리스 페이지에서 적용 대상과 변경 사항을 다시 확인하는 것이 좋다.

- [Nextcloud Hub 26 Spring 공식 발표](https://nextcloud.com/blog/press_releases/nextcloud-hub-26-spring-delivers-new-office-experience-lighter-ui-and-and-a-new-governance-tool/)
- [Nextcloud 2026년 7월 유지보수 업데이트](https://nextcloud.com/blog/category/release/)
- [Nextcloud AIO v13.3.1 자동 도메인 안내](https://nextcloud.com/blog/nextcloud-all-in-one-introduces-automatic-domain-acquisition-dns-setup/)

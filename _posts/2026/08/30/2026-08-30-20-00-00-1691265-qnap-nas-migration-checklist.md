---
layout: post
title: "QNAP NAS 마이그레이션 체크리스트 - Synology에서 옮길 때 공유폴더·Docker·백업 검증하기"
description: "Synology에서 QNAP NAS로 옮길 때 파일 복사만 하지 않고 계정 권한, Docker 볼륨, 백업과 복구까지 검증하는 실전 마이그레이션 순서를 정리한다."
date: 2026-08-30
tags: [QNAP, NAS설정, 홈서버, Docker, 백업전략]
comments: true
share: true
---
![QNAP NAS 마이그레이션과 백업 검증 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

QNAP NAS로 옮길 때 가장 위험한 실수는 파일 복사가 끝난 순간 마이그레이션도 끝났다고 생각하는 것이다. 2026년 8월 QNAP 공식 안내도 저장공간과 서비스 제한을 확인한 뒤 이전하라고 설명한다. 내가 실제로 옮긴다면 기존 NAS를 바로 끄지 않고, 새 QNAP에 파일·권한·컨테이너·복구 경로를 각각 확인한 뒤 DNS만 바꾼다.

## 환경과 이전 범위

아래 조건을 기준으로 잡으면 모델이 달라도 순서를 적용하기 쉽다.

| 항목 | 예시 |
|---|---|
| 기존 장비 | Synology DSM 7.2.2, 4베이 SHR |
| 새 장비 | QNAP QTS 5.2, 4베이 RAID 5 |
| 데이터 | 문서 1.2TB, 사진 2.8TB, 미디어 5TB |
| 서비스 | Container Station, Jellyfin, Uptime Kuma |
| 네트워크 | 기존 NAS `192.168.1.20`, 새 NAS `192.168.1.30` |

RAID는 백업이 아니다. 디스크 장애를 견디는 방식일 뿐이므로, 이전 작업 전에 외장 디스크나 다른 NAS에 원본 백업을 하나 남긴다. 파일 수가 많은 사진 폴더는 용량보다 파일 개수 때문에 검증 시간이 길어진다.

## 1. 새 QNAP에서 저장공간과 계정부터 만든다

QTS에서 스토리지 풀과 볼륨을 만든 뒤 바로 데이터를 넣지 않는다. `Control Panel → Privilege → Users`에서 기존 NAS와 같은 사용자 이름을 만들고, 공유폴더도 다음처럼 나눈다.

```text
/share/Docs
/share/Photos
/share/Media
/share/AppData
```

`AppData`는 일반 사용자에게 공유하지 않는다. Docker 컨테이너 설정과 데이터베이스가 들어갈 공간이라 권한을 넓게 주면 서비스 계정의 실수가 전체 파일로 번질 수 있다. SMB 공유폴더 이름과 실제 경로를 다르게 만들면 컨테이너 compose 파일을 고칠 때 헷갈리므로 이름을 유지하는 편이 낫다.

## 2. 파일은 SMB보다 동기화 작업으로 옮긴다

수 TB를 노트북으로 내려받았다가 다시 올리면 중단 지점을 관리하기 어렵다. QNAP의 HBS 3(Hybrid Backup Sync) 또는 rsync를 사용해 NAS 간 복사 작업을 만든다. 첫 복사는 밤새 돌리고, 이후 변경분만 한 번 더 동기화한다.

복사 후에는 폴더 용량만 비교하지 말고 파일 개수와 샘플 해시를 확인한다. Linux 셸에서 원본과 대상에 각각 실행할 수 있다.

```bash
find /volume1/Photos -type f | wc -l
find /share/Photos -type f | wc -l
sha256sum "Photos/2026/08/family.jpg"
```

파일 개수가 다르면 숨김 파일, 휴지통, 권한 오류 로그부터 확인한다. 숫자가 같아도 파일 내용이 같다는 뜻은 아니므로 사진·문서·대용량 영상에서 각 2~3개씩 해시를 비교한다.

## 3. Docker는 이미지가 아니라 볼륨을 이전한다

컨테이너를 새 QNAP에서 다시 받는 것만으로는 Jellyfin 라이브러리와 Uptime Kuma 알림 설정이 돌아오지 않는다. 기존 compose 파일과 `AppData`를 함께 복사하고, QNAP 경로에 맞게 왼쪽 경로만 수정한다.

```yaml
services:
  jellyfin:
    image: jellyfin/jellyfin:latest
    volumes:
      - /share/AppData/jellyfin:/config
      - /share/Media:/media:ro
    ports:
      - "8096:8096"
```

`/config`를 비워둔 채 컨테이너만 실행하면 새 서버로 인식한다. 기존 설정이 꼭 필요하면 컨테이너를 멈춘 상태에서 복사하고, 실행 후 로그에서 데이터베이스 마이그레이션 오류가 없는지 확인한다. `latest` 태그를 그대로 쓰기보다 이전 장비에서 사용하던 이미지 버전을 먼저 고정하는 쪽이 안전하다.

## 4. 외부 접속과 백업을 마지막에 전환한다

새 QNAP 내부 IP로 SMB, Jellyfin, 관리자 화면을 확인한 뒤 공유기의 포트포워딩과 DDNS 대상만 바꾼다. QTS 관리자 포트나 SMB 포트를 인터넷에 직접 공개하지 않고, 외부 접속은 Tailscale VPN 또는 HTTPS 역방향 프록시로 제한한다.

마지막으로 백업 작업의 목적지를 확인한다. HBS 3 작업이 성공으로 표시돼도 실제 복구가 되는지는 별도 문제다. 문서 하나를 다른 폴더로 복원하고, 사진 한 장을 열고, compose 파일로 컨테이너를 재생성해본다.

### 이전 완료 체크리스트

- [ ] 원본 NAS를 최소 7일 보존
- [ ] 공유폴더별 파일 개수와 샘플 해시 비교
- [ ] 사용자·그룹·관리자 MFA 확인
- [ ] Docker compose와 `AppData` 복사
- [ ] 내부망에서 SMB·미디어 서버 접속 확인
- [ ] 외부 포트는 VPN/HTTPS만 허용
- [ ] HBS 3 백업 1회 실행 및 파일 복원

QNAP으로의 마이그레이션은 디스크를 옮기는 작업이 아니라 주소, 권한, 서비스 데이터를 새 환경에 재현하는 작업이다. 기존 NAS를 바로 초기화하지 않고 `복사 → 내부 검증 → 외부 전환 → 복구 테스트` 순서를 지키면 문제가 생겨도 돌아갈 지점이 남는다.

참고: [QNAP NAS Migration Guide 공식 안내](https://blog.qnap.com/en/nas-migration-guide-switching-to-qnap-to-escape-drive-lock-in-and-storage-limits/)

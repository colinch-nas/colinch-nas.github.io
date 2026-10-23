---
layout: post
title: "Immich 3.x SSD·HDD 저장소 구성 - 사용자 수별 호환성과 총비용"
description: "Immich 3.x를 NAS·홈서버에 운영할 때 PostgreSQL·썸네일은 SSD, 원본 사진은 HDD에 두는 기준과 1·4·10인 규모별 저장장치·백업 총비용을 정리한다."
date: 2026-10-23
tags: [Immich, Docker, NAS, SSD, HDD, HomeLab]
comments: true
share: true
---
![Immich 사진 서버의 SSD·HDD 저장소 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)
<small>사진 서버에서 빠른 영역과 큰 용량 영역을 분리하는 구성을 보여주는 이미지다.</small>

Immich 3.x를 한두 명이 쓰면 SSD를 추가로 여러 장 살 필요가 없다. **PostgreSQL 데이터베이스는 로컬 SSD**, 원본 사진과 영상은 대용량 HDD, 백업은 별도 디스크에 두는 구성이 비용과 복구 난이도의 균형이 좋다. 반대로 데이터베이스까지 NAS의 NFS·SMB 공유 폴더에 두는 구성은 공식 문서가 피하라고 하므로 적용하지 않는 편이 안전하다. 여기서 말하는 버전은 2026년 10월 23일 확인한 Immich 공식 3.x 문서 기준이며, 실제 이미지 태그와 요구사항은 배포 시 다시 확인해야 한다.

## 저장소를 나눠야 하는 이유

Immich는 원본만 저장하지 않는다. 사용자 메타데이터와 경로는 PostgreSQL에, 썸네일·미리보기·인코딩 영상은 별도 폴더에 쓴다. 공식 요구사항은 PostgreSQL을 로컬 SSD에 두고, 썸네일과 트랜스코딩 결과가 사진 라이브러리 용량의 평균 10~20%를 더 사용할 수 있다고 설명한다.

| 영역 | 권장 위치 | 이유 | SSD가 없어도 되는 경우 |
|---|---|---|---|
| PostgreSQL `DB_DATA_LOCATION` | 로컬 SSD | 검색·앨범·메타데이터의 작은 읽기/쓰기 | 사진이 거의 없고 테스트만 할 때 |
| 원본 `UPLOAD_LOCATION/library` | NAS HDD 또는 RAID | 용량과 원본 보존이 우선 | 1TB 이하이고 빠른 접근이 더 중요한 경우 |
| `thumbs`, 인코딩 결과 | SSD 또는 HDD | 탐색 반응성과 용량의 절충 | 사용자 1명, 모바일 백업 위주 |
| 백업 | 다른 디스크·다른 장비 | 같은 디스크 고장에 대비 | 원본을 잃어도 되는 임시 자료뿐일 때 |

## 사용자 수별 구매 판단

아래 금액은 판매가가 아니라 **가상 계산**이다. 1TB SSD 10만 원, 8TB NAS용 HDD 18만 원, 별도 8TB 백업 HDD 18만 원, 전력 단가 150원/kWh로 가정했다. 사진 증가량과 전기요금은 집마다 달라 실제 견적이 아니다.

| 규모 | 구성 | 저장장치 초기비용 | 구매 판단 |
|---|---|---:|---|
| 1인·사진 1TB 이하 | 기존 SSD DB + HDD 원본 + USB 백업 | 약 36만 원 | 새 SSD를 사지 말고 DB만 로컬에 둔다 |
| 4인·사진 3~5TB | 1TB SSD DB/썸네일 + 8TB HDD 원본 + 8TB 백업 | 약 46만 원 | 원본과 백업 디스크를 분리한다 |
| 10인·영상 포함 | 1TB 이상 SSD DB/썸네일 + 2개 이상 HDD + 별도 백업 | 64만 원 이상 | HDD 한 장만으로 운영하지 않는다 |

SSD와 HDD를 같은 디렉터리처럼 섞는 것이 핵심이 아니다. Docker Compose의 `DB_DATA_LOCATION`만 로컬 SSD 경로로 지정하고, `UPLOAD_LOCATION`은 NAS 볼륨으로 지정한다. SSD가 고장 나면 원본만으로는 앨범·얼굴 인식·사용자 메타데이터가 완전히 돌아오지 않으므로 DB 덤프도 같은 백업 정책에 포함해야 한다.

## 복원 체크리스트

공식 백업 문서는 DB와 미디어 파일을 함께 복원해야 한다고 안내한다. 백업 성공 알림만 확인하지 말고 아래 순서로 검증한다.

- [ ] `DB_DATA_LOCATION`의 PostgreSQL 덤프와 설정 파일을 별도 디스크에 복사한다.
- [ ] `UPLOAD_LOCATION`의 `library`, `thumbs`, `profile`, `encoded-video` 경로를 함께 보관한다.
- [ ] 테스트 컨테이너에서 DB를 복원하고 원본 사진 1개, 영상 1개, 앨범 1개를 연다.
- [ ] 썸네일이 없으면 재생성 작업이 필요하다는 점을 복구 시간에 포함한다.
- [ ] Immich 버전을 바꿔 복원할 때는 DB 마이그레이션 가능 여부를 확인한다.

구매하지 않아도 되는 조건도 분명하다. 사용자가 1명이고 원본이 다른 클라우드에도 남아 있다면 SSD 캐시나 10GbE보다 현재 서버의 DB를 로컬 디스크에 두는 것이 우선이다. 반대로 여러 사용자가 동시에 검색하고 영상 미리보기를 만든다면 SSD 용량을 줄여 HDD만 늘리는 선택이 오히려 체감 속도와 복구 시간을 악화시킬 수 있다.

짧게 정리하면 **DB는 로컬 SSD, 원본은 용량 중심 HDD, 백업은 별도 위치**가 기본선이다. 저장장치 가격만 비교하지 말고 썸네일 10~20% 추가 공간, DB 복구, 전력과 백업 디스크까지 합산해야 Immich의 실제 운영비를 판단할 수 있다.

참고한 공식 문서:

- [Immich 요구사항](https://docs.immich.app/install/requirements/)
- [Immich 백업과 복원](https://docs.immich.app/administration/backup-and-restore/)
- [Immich 시스템 설정](https://docs.immich.app/administration/system-settings/)

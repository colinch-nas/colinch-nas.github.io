---
layout: post
title: "TrueNAS SCALE 25.10 Cloud Sync 복원 - S3·2·4·8베이 총비용 판단"
description: "TrueNAS SCALE 25.10 Cloud Sync를 S3 백업과 복원에 사용할 때 2·4·8베이 장비별 호환성, ACL 위험, 저장·전송비용과 복원 점검표를 정리한다."
date: 2026-10-17
tags: [TrueNAS, TrueNAS보안, HomeLab, 백업, S3]
comments: true
share: true
---

![TrueNAS SCALE Cloud Sync 백업과 복원 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

이 글의 기준은 **TrueNAS SCALE 25.10에서 2·4·8베이 NAS의 중요한 데이터만 S3 호환 저장소에 백업하고, 실제 복원 경로까지 확인하는 경우**다. 사진·문서·앱 설정을 별도 장소에 두려면 Cloud Sync가 유용하지만, ZFS 풀 전체를 그대로 되살리는 기능은 아니다. 디스크 고장 뒤 빠른 전체 복구가 목표라면 로컬 스냅샷·복제와 S3를 함께 계산해야 한다.

## 베이 수보다 복원 대상이 비용을 결정한다

TrueNAS 25.10의 Cloud Sync는 Amazon S3, Backblaze B2, Google Cloud Storage 등과 단방향 또는 반복 전송을 구성할 수 있다. `PUSH`로 올린 작업에서 메뉴의 `Restore`를 선택하면 전송 방향을 뒤집은 복원 작업을 만들 수 있다. 다만 복원 목적지를 원본 dataset으로 잡으면 파일 소유자가 `root`로 바뀔 수 있어 ACL 확인이 필요하다. [공식 Cloud Sync 문서](https://www.truenas.com/docs/scale/25.10/scaletutorials/dataprotection/cloudsynctasks/) 기준으로 확인한 내용이다.

| 운영 규모 | S3에 보낼 대상 | 권장 복구 조합 | 구매를 미뤄도 되는 조건 |
|---|---|---|---|
| 2베이·1인 | 문서, 사진, 설정 파일 | S3 + 로컬 스냅샷 | 미디어 원본을 다시 구할 수 있고 복구 시간이 하루 이상이어도 될 때 |
| 4베이·가족/소규모 팀 | 업무 폴더, DB 덤프, 컨테이너 설정 | S3 + USB 또는 다른 NAS | 장애 때 임시 공유 폴더로 버틸 수 있으면 고성능 백업 NAS는 보류 |
| 8베이·다중 서비스 | 중요 dataset, 앱별 백업, 복구용 메타데이터 | 원격 복제 + S3 오프사이트 | 서비스 중단 허용 시간이 길고 원격 복제 비용이 더 큰 경우 |

## 설정할 때 빠뜨리기 쉬운 조건

Cloud Sync 작업에는 `Credentials > Backup Credentials > Cloud Credentials`의 전용 자격 증명을 사용한다. S3 키에는 해당 bucket에 필요한 권한만 주고, NAS 관리자 키를 재사용하지 않는다. `Remote Encryption`을 켜면 S3에 암호화된 파일을 저장할 수 있지만 암호와 salt를 잃으면 복원이 막힌다. 파일명 암호화는 긴 파일명과 복원 호환성 문제가 있어 신규 작업에 무심코 켜지 않는 편이 안전하다.

비용은 아래처럼 계산하면 된다. 가격은 공급자와 지역에 따라 바뀌므로 2026년 10월 17일의 고정 금액으로 단정하지 않는다.

```text
월 총비용 = S3 저장용량 × 저장단가
          + 월간 업로드·다운로드 GB × 전송단가
          + 복원 시간 × 장애 중 인건비
          + 백업용 HDD·UPS의 월 환산 비용
```

예를 들어 가상으로 6TB를 S3에 보관하고 매월 300GB가 바뀐다면, 저장비만 보지 말고 전체 6TB를 한 번 내려받는 복구 전송비와 복구에 걸리는 시간을 별도 항목으로 적는다. S3 Glacier 계열처럼 즉시 꺼낼 수 없는 저장 등급은 복원 전에 해제 시간이 필요하고, TrueNAS 문서도 Glacier 객체는 S3 콘솔 등 다른 방법으로 먼저 복원하라고 안내한다.

## 발행 전 복원 체크리스트

- [ ] 실제 운영 dataset과 별도의 `restore-test` dataset을 만들었다.
- [ ] 작은 파일, 권한이 있는 파일, 컨테이너 설정, DB 덤프를 각각 복원했다.
- [ ] 파일 소유자·ACL·심볼릭 링크가 원래 기대와 같은지 확인했다.
- [ ] 암호화 키, S3 endpoint, bucket 이름, 자격 증명 보관 위치를 기록했다.
- [ ] `Dry Run`과 실제 `Restore` 로그를 저장했다.
- [ ] S3가 아닌 USB 또는 원격 복제만으로도 임시 서비스를 띄울 수 있는지 계산했다.

Cloud Sync의 성공 로그는 백업 사본이 만들어졌다는 뜻이지 애플리케이션이 정상 기동한다는 뜻은 아니다. 2베이는 S3와 USB 외장 HDD만으로도 시작할 수 있고, 4베이부터는 복구 시간 때문에 원격 복제를 추가할 이유가 생긴다. 8베이는 디스크를 더 사는 것보다 어떤 dataset을 몇 시간 안에 복구해야 하는지 정한 뒤 저장·전송·운영 중단 비용을 함께 비교하는 편이 정확하다.

참고: [TrueNAS SCALE 25.10 데이터 보호 문서](https://www.truenas.com/docs/scale/25.10/scaletutorials/dataprotection/), [25.10 릴리스 노트](https://www.truenas.com/docs/scale/25.10/gettingstarted/scalereleasenotes/)

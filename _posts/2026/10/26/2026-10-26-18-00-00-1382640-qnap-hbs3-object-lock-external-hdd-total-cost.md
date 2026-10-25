---
layout: post
title: "QNAP QTS 5.2 HBS 3 Object Lock - 외장 HDD와 2·4베이 총비용 판단"
description: "QNAP QTS 5.2의 HBS 3에서 Object Lock과 외장 HDD 로테이션을 비교하고, 2·4베이 NAS의 복원 조건과 가상 총비용을 계산한다."
date: 2026-10-26
tags: [QNAP, QTS, HBS3, NAS백업, NAS보안, ObjectLock]
comments: true
share: true
---

![QNAP HBS 3 오프사이트 백업 구성](https://images.unsplash.com/photo-1544197150-880d4212a0c4?w=1200&q=80)

QNAP NAS를 1~2명이 쓰고 외장 디스크를 교대 보관할 수 있다면 Object Lock보다 외장 HDD가 저렴하다. 4베이 NAS를 여러 사람이 쓰거나 백업 디스크를 계속 연결해 둬야 한다면 QTS 5.2의 HBS 3와 S3 Object Lock이 랜섬웨어 삭제 위험을 줄인다. 다만 Object Lock은 백업 자체가 아니므로 복원 테스트가 필요하다.

## 외장 HDD와 Object Lock의 차이

QNAP 공식 문서 기준 HBS 3는 Amazon S3, Backblaze B2, Wasabi 등 일부 저장소에서 Object Lock을 설정한다. 지정 기간 동안 객체를 지우거나 덮어쓰지 못하게 하는 WORM 방식이며, 기본 불변 기간은 30일이다. 변경 파일은 새 객체와 새 보존 기간으로 처리된다. [QNAP Object Lock 설정 문서](https://www.qnap.com/en/how-to/faq/article/how-do-i-enable-and-configure-object-lock-settings-in-an-hbs-job)

| 구성 | 조건 | 비용이 커지는 지점 | 맞는 경우 |
|---|---|---|---|
| 외장 HDD 2개 교대 | USB·파일시스템·복원 도구 확인 | 디스크 2개와 보관 장소 | 2베이, 1~2명 |
| S3 Object Lock | HBS 3·지원 클라우드·버킷 정책 확인 | 저장료·요청·복원 전송료 | 4베이, 원격 보관 |
| 둘 다 사용 | 두 경로를 각각 검증 | 초기·월 운영비 모두 발생 | 업무 원본, 짧은 복구 시간 |

## 가상 총비용 계산

실제 판매가가 아닌 가상 사례다. 보호 데이터 4TB인 2베이는 외장 HDD 6TB 2개, 12TB인 4베이는 외장 HDD 16TB 2개 또는 S3 12TB 이상으로 계산한다.

`외장 HDD 총비용 = 디스크 2개 + USB 케이블·보관함 + 교대에 드는 시간`

`Object Lock 총비용 = 월 저장료 + 업로드·요청료 + 복원 전송료 + 암호화 키 관리 비용`

외장 HDD는 월 요금이 없지만 NAS 옆에 계속 꽂아 두면 오프사이트 사본이 아니다. Object Lock도 30일 뒤 비용이 자동으로 줄지는 않는다. 변경 객체가 보존 기간 안에 남기 때문이다. 가격은 실제 견적을 식에 대입한다.

## HBS 3 적용 전 체크리스트

- [ ] QTS·HBS 3 버전과 NAS 모델이 대상 저장소를 지원하는가
- [ ] S3 버킷 생성 때 Versioning과 Object Lock을 함께 켰는가
- [ ] 불변·연장 보존 기간을 데이터 변경 주기보다 길게 잡았는가
- [ ] HBS 3 암호화 비밀번호와 클라우드 접근 키를 별도 보관했는가
- [ ] 복원 메뉴에서 한글 파일·권한·삭제 파일을 확인했는가
- [ ] Object Lock이 끝나기 전 삭제·정리 작업을 실행하지 않았는가

QNAP은 3-2-1-1-0의 마지막 0을 복구 오류 0건과 검증으로 설명한다. HBS 3 성공 알림만으로 완료 처리하면 안 된다. 실제 장비 테스트가 아닌 공식 문서와 가상 비용에 따른 판단이다. 2베이는 외장 HDD 교대만으로 충분할 수 있고, 4베이·다중 사용자는 Object Lock과 복원 시간을 예산에 넣는 편이 안전하다.

참고: [QNAP 3-2-1-1-0 백업 규칙](https://www.qnap.com/en-ca/solution/3-2-1-1-0-rule), [HBS 3 공식 페이지](https://www.qnap.com/en-us/software/hybrid-backup-sync?source=post_page)

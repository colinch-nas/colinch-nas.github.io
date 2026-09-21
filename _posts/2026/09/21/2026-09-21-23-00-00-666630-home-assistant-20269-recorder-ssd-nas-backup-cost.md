---
layout: post
title: "Home Assistant 2026.9.2 Recorder 저장장치 선택 - 센서 수별 SSD·NAS 백업 총비용"
description: "Home Assistant 2026.9.2 Recorder를 운영할 때 센서 수와 보존 기간별로 SQLite 저장장치, NAS 백업, 복원 여유 공간과 구매 비용을 계산하는 기준이다."
date: 2026-09-21
tags: [HomeAssistant, Synology, Docker, SSD, 백업, 홈서버]
comments: true
share: true
---

![Home Assistant Recorder 데이터가 SSD와 NAS 백업으로 흐르는 구조](/assets/images/2026/09/home-assistant-recorder-storage-backup.png)

이 그림에서 볼 부분은 센서 데이터가 먼저 운영 장치의 SQLite에 쌓이고, 복구용 사본은 NAS로 분리된다는 점이다.

Home Assistant 2026.9.2를 1~2명이 쓰고 센서가 50개 안팎이면 별도 DB 서버를 사지 말고, 로컬 SQLite를 SSD에 두고 NAS에는 백업만 저장하는 구성이 비용 대비 낫다. 센서가 200개를 넘거나 기록 기간을 1년 이상 유지한다면 SSD 여유 공간과 NAS 백업 용량을 함께 계산해야 한다. 반대로 단순 조명 자동화만 쓰고 이력 10일이면 고성능 NVMe나 MariaDB를 추가할 이유가 거의 없다.

## 공식 기준에서 놓치기 쉬운 숫자

Home Assistant 공식 Recorder 문서 기준으로 기본 DB는 `/config/home-assistant_v2.db`인 SQLite다. 기본 보존 기간은 10일이며, DB를 바꾸는 마이그레이션은 지원되지 않는다. 업그레이드나 정리 과정에서 DB 크기만큼의 임시 공간이 필요하고, SQLite가 손상돼 새 DB를 만드는 최후 복구 상황에는 DB 크기의 2.5배 여유 공간이 권장된다. 이 글의 기준일은 2026년 9월 21일이며, [Recorder 공식 문서](https://www.home-assistant.io/integrations/recorder/)를 확인했다.

| 환경 | 권장 운영 저장장치 | 보존 시작값 | NAS 백업 판단 |
|---|---|---:|---|
| 1~2명, 센서 50개 이하 | 기존 SSD 또는 eMMC | 10일 | 매일 전체 백업 |
| 2~4명, 센서 50~200개 | SATA SSD 이상 | 30일 | `/config` 백업을 NAS에 저장 |
| 센서 200개 이상, 장기 이력 | SSD + 필터링 | 90일 이하부터 검토 | DB 크기와 복원 시간을 함께 측정 |

센서 수만 세면 부족하다. 전력·온도 센서처럼 값이 자주 바뀌는 엔티티가 20개인지, 하루 한 번 상태가 바뀌는 문 센서가 100개인지에 따라 DB 증가량이 달라진다. 따라서 아래처럼 기록 대상을 줄이는 것이 디스크를 사는 것보다 먼저다.

```yaml
# 기록량이 큰 엔티티를 제외해 SQLite 증가 속도를 낮추는 예시다.
recorder:
  purge_keep_days: 30
  auto_purge: true
  exclude:
    entity_globs:
      - sensor.*_linkquality
      - sensor.*_rssi
    domains:
      - automation
```

위 설정은 예시다. 실제 엔티티 이름을 확인하지 않고 붙여 넣으면 필요한 통계까지 사라질 수 있다. `설정 → 시스템 → 복구 → 시스템 정보`에서 Estimated Database Size를 기록한 뒤, 7일 후 증가량을 다시 확인하는 방식이 안전하다.

## SSD와 NAS 비용을 같이 계산하는 법

다음은 가격을 고정한 추천표가 아니라, 구매 전 계산을 위한 가상 사례다. SSD 500GB 50,000원, NAS에 둘 HDD 4TB 120,000원, 전력 단가 170원/kWh, Home Assistant 미니 PC 소비전력 8W, NAS 평균 25W로 가정했다.

| 구성 | 초기 저장장치 비용 | 연간 전기료 계산 | 어울리는 경우 |
|---|---:|---:|---|
| 기존 로컬 디스크 + NAS 백업 | 120,000원 | `(8+25)W×24×365÷1000×170` ≈ 49,000원 | 대부분의 1~4인 가정 |
| 새 500GB SSD + NAS 백업 | 170,000원 | 약 49,000원 | DB 쓰기가 잦거나 기존 디스크가 불안정한 경우 |
| SSD + 별도 MariaDB 서버 + NAS 백업 | 200,000원 이상 | 서버 전력 추가 | DB 관리와 장애 대응을 직접 할 사람 |

실제 전기료와 장비 가격은 지역·요금제·부하에 따라 달라진다. NAS가 이미 24시간 켜져 있다면 Recorder 때문에 추가되는 전기료는 거의 0원에 가깝다. 반대로 Home Assistant만 쓰려고 NAS를 새로 사는 것은 DB 하나를 위해 운영 장비와 백업 장비를 동시에 늘리는 선택이 된다.

활성 SQLite DB를 SMB/NFS 공유 폴더에 직접 두는 구성은 네트워크 단절과 파일 잠금이 복구 변수가 된다. 공식 문서가 기본 DB를 로컬 `/config`에 두도록 설명하는 이유도 이 운영 단순성에 있다. NAS는 [Home Assistant 백업 파일을 보관하는 네트워크 저장소](https://www.home-assistant.io/common-tasks/general/#backups)로 쓰고, DB 자체는 로컬 디스크에 두는 편이 일반적인 단일 홈서버에 맞다.

## 백업 성공보다 복원 확인이 먼저다

백업 파일이 만들어졌다는 알림만으로는 Recorder 복구를 증명할 수 없다. 월 1회 테스트 폴더에서 다음 순서로 확인한다.

- [ ] Home Assistant 전체 백업을 NAS 공유 폴더에 복사한다.
- [ ] 백업 파일의 날짜와 파일 크기가 이전 실행과 같은지 확인한다.
- [ ] 테스트 컨테이너 또는 예비 장치에 복원한다.
- [ ] 자동화 목록, 대시보드, 장치 연결, History 화면을 각각 연다.
- [ ] DB 크기가 큰 경우 복구 장치에 원본 DB의 2.5배 이상 여유 공간을 남긴다.

핵심은 센서 수가 아니라 “복구할 이력이 정말 필요한가”다. 현재 상태와 자동화만 중요하면 10일 보존과 NAS 백업으로 충분할 수 있다. 전력 사용량을 1년 비교해야 한다면 보존 기간을 늘리되, 기록량이 큰 엔티티를 먼저 제외하고 SSD·NAS 용량을 다시 산정하는 순서가 맞다.

짧게 정리하면 `SQLite는 로컬 SSD`, `백업은 NAS`, `보존 기간은 센서별로 제한`, `월 1회 실제 복원`이 기본선이다. MariaDB나 새 NAS 구매는 센서 수와 복원 요구를 측정한 뒤에도 병목이 남을 때 결정하면 된다.

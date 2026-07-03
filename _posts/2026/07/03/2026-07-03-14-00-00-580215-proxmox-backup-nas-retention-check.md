---
layout: post
title: "Proxmox Backup Server - NAS 저장소 보존 정책을 7일만 믿으면 안 되는 이유"
description: "Proxmox Backup Server를 NAS 저장소에 붙일 때 daily·weekly·monthly 보존 정책과 prune, verify 순서를 실전 기준으로 정리했다."
date: 2026-07-03
tags: [Proxmox, NAS, HomeLab, 백업, Docker]
comments: true
share: true
---
![Proxmox Backup Server NAS 보존 정책](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)

이 그림에서는 백업 장비가 있다는 사실보다 복구 가능한 시점이 몇 개 남아 있는지가 더 중요하다는 점을 봐야 한다.

Proxmox Backup Server를 NAS에 붙일 때 `keep-daily 7`만 설정하면 마음은 편하지만 실제 복구 시나리오는 빈약하다. VM 업데이트 실패는 당일 알 수 있지만, 설정 파일이 조용히 깨진 문제는 2~3주 뒤에 발견되기도 한다. 나는 처음에 7일 보존만 믿었다가 Home Assistant DB 손상을 늦게 알아차려 쓸 수 있는 백업이 없었다.

내 기준 장비는 미니 PC Proxmox VE, 별도 NAS의 PBS datastore, 2.5GbE 내부망이다. VM 6개, LXC 4개 기준으로 daily만 남기면 저장공간은 적게 쓰지만 장애 발견 지연에 약했다.

| 보존 단위 | 예시 | 잡는 문제 |
|---|---:|---|
| daily | 7개 | 업데이트 직후 실패 |
| weekly | 4개 | 2~4주 뒤 발견한 설정 꼬임 |
| monthly | 3개 | 장기 데이터 비교 |
| manual | 작업 전 1개 | DSM, HA, DB 큰 변경 전 롤백 |

나는 아래처럼 시작하는 편이다.

```text
keep-last: 3
keep-daily: 7
keep-weekly: 4
keep-monthly: 3
```

여기서 중요한 건 prune만 돌리는 게 아니라 verify까지 묶는 것이다. 백업 목록에는 있는데 실제 chunk가 깨져 있으면 복구 버튼을 누르는 순간 알게 된다. NAS 디스크가 SMR이거나 절전 모드에서 자주 깨어나는 환경이면 verify 시간이 예상보다 길 수 있다.

설정 순서는 이렇게 잡았다.

| 단계 | 작업 | 확인할 것 |
|---|---|---|
| 1 | datastore 생성 | NAS 공유 권한, 여유 공간 |
| 2 | backup job 등록 | VM별 제외 디스크 |
| 3 | prune schedule 설정 | daily/weekly/monthly 균형 |
| 4 | verify schedule 설정 | 주 1회 이상 실제 검증 |
| 5 | 복구 테스트 | 새 VM으로 부팅 확인 |

삽질한 지점은 백업 성공 알림만 보고 끝낸 것이다. 실제로는 VM 하나를 새 ID로 복구해서 부팅해봐야 네트워크 MAC, 고정 IP, Docker volume까지 감이 온다. 특히 Home Assistant, Vaultwarden, Immich처럼 DB가 중요한 서비스는 앱 화면까지 열어봐야 한다.

짧게 정리하면 이렇다. NAS에 백업이 있다는 말은 충분하지 않다. Proxmox Backup Server에서는 보존 정책, verify, 복구 테스트까지 한 묶음으로 봐야 한다. 7일 daily만 남기는 설정은 깔끔하지만, 늦게 발견되는 장애에는 생각보다 약하다.

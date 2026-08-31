---
layout: post
title: "TrueNAS 26-BETA.3 업데이트 체크리스트 - ZFS 풀과 앱을 안전하게 검증하는 순서"
description: "2026년 8월 20일 공개된 TrueNAS 26-BETA.3 업데이트 전후에 설정 백업, ZFS 풀, SMB, 앱, REST API를 점검하는 실제 홈서버 체크리스트다."
date: 2026-09-01
tags: [TrueNAS, NAS설정, 홈서버구축, HomeLab, NAS보안, 백업전략]
comments: true
share: true
---

![TrueNAS 홈서버 스토리지 화면](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

TrueNAS 26-BETA.3는 2026년 8월 20일 공개됐다. Linux 커널 6.18.42와 OpenZFS 2.4.3으로 올라갔지만, 공식 문서도 early release를 테스트 용도로만 안내한다. 사진과 백업을 맡긴 홈서버라면 지금 바로 올리기보다, 별도 부팅 디스크나 테스트 장비에서 검증하는 쪽이 맞다. 그래도 업그레이드한다면 아래 순서대로 확인하면 된다.

## 이번 버전에서 실제로 신경 쓸 부분

| 점검 대상 | TrueNAS 26에서 달라진 점 | 판단 기준 |
|---|---|---|
| 풀 기능 | OpenZFS 2.4 계열, 새 기능 플래그 | 롤백 가능성을 남기려면 풀 업그레이드 보류 |
| 앱·컨테이너 | 컨테이너와 가상머신 관련 수정 | 앱별 마운트 경로와 권한 재확인 |
| API | 기존 REST API 제거 | 자동화 스크립트가 WebSocket API를 쓰는지 확인 |
| SMB | 보조 파라미터 변경 가능성 | auxiliary parameters를 비우고 공유 테스트 |

그림에서 볼 부분은 버전 숫자보다 저장 풀과 앱 데이터가 같은 위치에 안전하게 남아 있는지다.

## 업데이트 전 백업과 상태 저장

환경은 TrueNAS 25.10.6, 4베이 ZFS RAIDZ1, SMB 공유 3개, Apps에 Jellyfin과 Syncthing을 올린 상태로 잡았다. 설정 파일만 저장하고 끝내면 앱 데이터와 스냅샷은 보호되지 않는다.

`System Settings → General → Manage Configuration`에서 설정 파일을 다운로드하고, 암호화된 시크릿을 포함했다면 안전한 외부 저장소에도 복사한다. `Storage → Snapshots`에서는 중요한 데이터셋의 최신 스냅샷을 확인한 뒤 다른 디스크나 원격 TrueNAS로 복제한다.

셸에서는 풀 상태와 최근 오류를 기록한다. 아래 명령은 상태 확인용이며, 출력 결과를 업데이트 전 파일로 보관하면 비교가 쉽다.

```bash
zpool status -v
zpool list
zfs list -o name,used,avail,refer,mountpoint
dmesg | tail -n 80
```

`DEGRADED`, resilvering, checksum 오류가 하나라도 있으면 업데이트를 미룬다. 디스크 문제가 있는 상태에서 운영체제까지 바꾸면 원인 분리가 어려워진다.

## UI에서 안전하게 업데이트하기

`System Settings → System → Update`에서 현재 브랜치의 최신 유지보수 버전을 확인한다. 공식 경로는 웹 UI의 업데이트 기능이며, SSH에서 `apt upgrade`로 처리하면 안 된다. 베타를 선택했다면 `Developer` 성격의 테스트 장비로만 적용한다.

업데이트 전에 `System Settings → Advanced → Syslog`나 서비스별 고급 설정에 넣어둔 auxiliary parameters를 기록하고 제거한다. 버전이 바뀌며 이 값이 SMB를 깨뜨릴 수 있다. 웹 UI에서 재부팅이 끝나면 브라우저 캐시를 `Ctrl+F5`로 지운 뒤 접속한다.

풀 업그레이드 알림이 나와도 바로 승인하지 않는다. 시스템 업데이트와 풀 업그레이드는 별개다. 풀 기능 플래그를 올리면 이전 TrueNAS 버전에서 부팅해도 새 기능을 읽지 못할 수 있어, 베타를 시험하는 동안에는 보류하는 편이 안전하다.

## 재부팅 후 확인 순서

| 순서 | 확인할 화면·기능 | 통과 조건 |
|---|---|---|
| 1 | Dashboard, Alerts | 디스크·부트 풀·메모리 경고 없음 |
| 2 | Storage → Pools | 풀 ONLINE, 스냅샷 목록 정상 |
| 3 | Shares → SMB | PC에서 읽기·쓰기·삭제 모두 성공 |
| 4 | Apps | Jellyfin/Syncthing 실행, 마운트 경로 유지 |
| 5 | Tasks | 스냅샷·복제·클라우드 백업 수동 실행 성공 |

자동화 스크립트가 TrueNAS REST API 주소를 호출한다면 특히 주의해야 한다. TrueNAS 26에서는 REST API가 제거됐으므로 JSON-RPC 2.0 WebSocket API로 바꿔야 한다. API 키는 `User Settings → API Keys`에서 새로 만들고, 인터넷에 직접 노출하지 않는다.

베타 테스트를 끝내고 이전 버전으로 돌아갈 때는 `System Settings → Boot → Boot Environments`에서 기존 부팅 환경을 선택한다. 단, 풀을 업그레이드했다면 단순 부팅 환경 롤백만으로 원상 복구되지 않을 수 있다.

정리하면, TrueNAS 26-BETA.3는 새 커널과 OpenZFS를 시험할 만한 버전이지만 일반 홈서버의 안정 업데이트로 보기는 이르다. 설정 파일 백업, 복제본 확인, auxiliary parameters 제거, 앱·SMB 테스트, 풀 업그레이드 보류까지 끝났을 때만 적용하는 것이 현실적인 기준이다.

참고 문서: [TrueNAS 26 Version Notes](https://www.truenas.com/docs/scale/26/gettingstarted/versionnotes/), [TrueNAS 26 BETA.2 안내](https://www.truenas.com/blog/truenas-26-beta2-next-step/)

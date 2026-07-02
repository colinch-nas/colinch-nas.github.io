---
layout: post
title: "Synology DSM 보안 업데이트 - 7.2.2 Update 7 이상 올리기 전 점검 순서"
description: "Synology-SA-26:06 DSM 보안 권고를 기준으로 DSM 7.2.2 Update 7 이상 적용 전 스냅샷, 패키지, 외부 접속을 점검하는 절차다."
date: 2026-07-02
tags: [Synology, DSM, NAS보안, NAS설정, 백업전략]
comments: true
share: true
---
![Synology NAS 보안 업데이트를 점검하는 책상](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)

Synology NAS를 DSM 7.2.2로 쓰고 있다면 2026년 7월 2일 기준 `7.2.2-72806 Update 7` 이상인지 확인하면 된다. Synology-SA-26:06 DSM 권고는 DSM 7.3, 7.2.2, 7.2.1에 걸친 파일 읽기·쓰기, 서비스 거부, 정보 노출 취약점을 다룬다. 외부 접속을 열어 둔 DS923+나 DS224+라면 "언젠가 자동으로 되겠지"보다 직접 점검하는 쪽이 낫다.

## 업데이트 전에 확인할 것

문제는 DSM 업데이트가 재부팅을 동반한다는 점이다. NAS에서 Synology Drive, Photos, Container Manager, Hyper Backup을 같이 돌리면 재부팅 타이밍 하나로 동기화 충돌이 생긴다. 나는 Drive 동기화가 끝나기 전에 올렸다가 노트북 쪽에서 동일 파일 충돌본이 여러 개 생긴 적이 있다.

DSM 화면에서 아래 순서로 본다.

```text
제어판 -> 업데이트 및 복구 -> DSM 업데이트
현재 버전 확인
DSM 7.2.2-72806 Update 7 이상이면 권고 기준 통과
낮은 버전이면 수동 업데이트 또는 중요 업데이트 적용
```

업데이트 버튼을 누르기 전에 공유 폴더 스냅샷부터 만든다. Snapshot Replication을 쓰고 있다면 `스냅샷 -> 생성`에서 이름을 이렇게 남긴다.

```text
before-dsm-722-update7-20260702
```

스냅샷은 DSM 자체 롤백이 아니라 파일 단위 복구용이다. 그래도 업데이트 직후 Drive나 Photos 인덱싱이 꼬였을 때 원본 파일을 확인할 기준점이 된다.

## 패키지와 컨테이너를 잠깐 멈춘다

패키지 센터에서 `Synology Drive Server`, `Synology Photos`, `Hyper Backup` 상태를 본다. 백업 작업이 도는 시간에는 업데이트하지 않는다. Container Manager에서 Home Assistant, Jellyfin, Vaultwarden 같은 컨테이너를 운영 중이면 compose 파일 위치와 볼륨 경로를 적어 둔다.

SSH를 켜 둔 NAS라면 재부팅 전 디스크 사용량도 확인한다.

```bash
df -h
```

시스템 파티션이나 `/volume1`이 꽉 찬 상태에서 업데이트하면 패키지 복구가 더 지저분해진다. 여유 공간이 10% 아래라면 휴지통 비우기와 오래된 Hyper Backup 버전 정리부터 한다.

## 외부 접속 NAS는 포트부터 줄인다

이번 권고에는 원격 공격자가 제한적 파일 읽기·쓰기나 서비스 거부를 일으킬 수 있는 항목이 포함됐다. DSM을 인터넷에 직접 열어 둔 상태라면 업데이트 전후로 공유기 포트포워딩을 확인한다. DSM 관리 포트 `5000`, `5001`을 그대로 외부에 열어 둔 구성은 피하고, 가능하면 Tailscale VPN이나 QuickConnect로만 관리 화면에 들어간다.

업데이트 후에는 세 가지만 확인한다.

```text
1. 제어판 -> 보안 -> 인증서 만료일
2. 패키지 센터 -> 모든 패키지 실행 상태
3. 파일 스테이션에서 테스트 파일 업로드와 삭제
```

DSM 7.2.2 Update 7 이상은 새 기능보다 보안 보수에 가깝다. 그래서 업데이트 성공 화면만 보고 끝내면 안 된다. 스냅샷을 만들고, 동기화·백업 패키지를 잠깐 멈추고, 재부팅 후 실제 파일 쓰기까지 해보면 된다.

참고한 공식 자료는 아래다.

- [Synology-SA-26:06 DSM 보안 권고](https://www.synology.com/en-br/security/advisory/Synology_SA_26_06)
- [Synology DSM 릴리스 노트](https://www.synology.com/releaseNote/DSM)

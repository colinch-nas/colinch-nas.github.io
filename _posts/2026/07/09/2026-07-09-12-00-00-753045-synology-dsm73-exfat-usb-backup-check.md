---
layout: post
title: "Synology DSM 7.3 exFAT 백업 - 외장 SSD를 꽂기 전에 확인할 것"
description: "Synology DSM 7.3의 native exFAT 지원을 기준으로 USB 외장 SSD 백업 디스크를 인식, 복사, 복구 테스트하는 순서다."
date: 2026-07-09
tags: [Synology, DSM, NAS설정, 백업전략, NAS보안]
comments: true
share: true
---
![Synology NAS 외장 SSD 백업 점검](https://images.unsplash.com/photo-1597852074816-d933c7d2b988?w=800&q=80)
이 그림에서는 NAS 본체보다 옆에 꽂힌 외장 디스크를 봐야 한다. DSM 업데이트 뒤 백업 매체가 그대로 읽히는지 확인하는 게 핵심이다.

Synology DSM 7.3으로 올렸다면 exFAT 외장 SSD는 "패키지 설치 후 인식"이 아니라 DSM 기본 기능으로 봐야 한다. 2026년 7월 9일 기준 Synology DSM 릴리스 노트는 외부 장치의 native exFAT 지원과 exFAT Access 패키지 중단을 명시한다. 그래서 업데이트 직후 할 일은 새 기능 구경이 아니라, 기존 USB 백업 디스크가 File Station과 Hyper Backup에서 같은 이름으로 보이는지 확인하는 것이다.

내 기준 환경은 DS923+, DSM 7.3 계열, USB 3.2 외장 SSD 1TB, 백업 대상은 `/volume1/docker`와 사진 보관 폴더다. 처음엔 exFAT가 기본 지원이면 그냥 꽂으면 끝이라고 생각했다. 해보니 디스크 이름, 권한, Hyper Backup 대상 경로가 바뀌는 쪽이 더 문제였다.

## 업데이트 직후 확인 순서

| 순서 | DSM 화면 | 통과 기준 |
|---|---|---|
| 1 | 제어판 -> 외부 장치 | exFAT SSD가 정상 상태로 표시 |
| 2 | File Station | `usbshare1` 같은 공유 폴더가 열림 |
| 3 | Hyper Backup | 기존 백업 작업의 대상 경로가 깨지지 않음 |
| 4 | 로그 센터 | 강제 분리, 파일 시스템 오류 로그 없음 |
| 5 | PC 또는 Mac | 백업 파일 일부를 외부 컴퓨터에서 읽을 수 있음 |

여기서 우선 볼 것은 속도가 아니다. 외장 SSD가 `usbshare1`이었다가 `usbshare2`로 잡히면 백업 작업은 정상처럼 보여도 실제 대상이 달라질 수 있다. DSM에서 외장 장치를 제거한 뒤 같은 포트에 다시 꽂고 이름이 유지되는지 본다.

## 실제 백업 테스트

Hyper Backup 전체 작업을 바로 돌리기 전에 작은 폴더 하나로 복사 테스트를 한다. 목적은 SSD 쓰기 속도 측정이 아니라 권한과 파일명 호환성 확인이다.

```text
테스트 폴더: /volume1/docker/homeassistant
복사 대상: /usbshare1/test-restore/homeassistant
확인 파일: configuration.yaml, automations.yaml, backups/*.tar
```

한글 파일명, 긴 파일명, 숨김 파일이 섞인 폴더를 골라야 한다. 사진 폴더만 복사하면 설정 파일 누락을 못 잡는다. 나는 처음에 `docker-compose.yml`을 빼먹고 백업이 된 줄 알았다. 복구 때 컨테이너 경로를 다시 맞추느라 20분을 버렸다.

## exFAT를 계속 쓸지 판단한다

| 선택 | 맞는 경우 | 아쉬운 점 |
|---|---|---|
| exFAT | Windows, macOS, DSM을 오가며 직접 읽어야 함 | 권한과 스냅샷 관리가 단순하다 |
| ext4 | NAS 전용 백업 디스크로만 씀 | 일반 PC에서 바로 열기 어렵다 |
| Hyper Backup 저장소 | 버전 보관과 암호화가 필요함 | Explorer 앱 없이는 구조가 낯설다 |

외장 SSD를 가족 PC와 같이 읽어야 하면 exFAT가 편하다. 다만 NAS 백업만 목적이면 파일 시스템보다 복구 절차가 더 중요하다. 백업 파일을 하나 만들고 끝내지 말고, 다른 컴퓨터에서 열어 보고, DSM에서 다시 복사해 보고, Home Assistant나 Jellyfin 설정 파일이 실제로 들어 있는지 확인한다.

주의할 점은 세 가지다. DSM 7.3 설치 후에는 이전 DSM으로 쉽게 되돌린다고 생각하면 안 된다. exFAT Access 패키지는 더 이상 중심 기능이 아니므로 패키지 상태보다 DSM 버전을 본다. 외장 SSD는 상시 연결보다 백업 시간에만 연결하는 쪽이 랜섬웨어 피해 범위를 줄인다.

짧게 정리하면 이렇다. DSM 7.3의 native exFAT 지원은 외장 SSD 백업을 쓰기 편하게 만들지만, 백업 성공을 보장하지는 않는다. 업데이트 직후에는 외장 장치 이름, Hyper Backup 대상 경로, 실제 복구 가능 파일 3가지를 확인한다. 출처는 [Synology DSM 릴리스 노트](https://www.synology.com/en-us/releaseNote/DSM)와 [Synology exFAT Access 릴리스 노트](https://www.synology.com/en-us/releaseNote/exFAT-Free)다.

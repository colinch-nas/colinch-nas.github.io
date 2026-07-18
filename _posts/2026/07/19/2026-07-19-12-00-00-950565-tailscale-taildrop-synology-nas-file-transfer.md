---
layout: post
title: "Synology NAS 파일 전송 - Tailscale Taildrop으로 외부 포트 없이 보내기"
description: "DSM 7.2.2 Synology NAS에 Tailscale Taildrop을 설정해 휴대폰·노트북과 파일을 주고받는 방법. 포트포워딩 없이 공유 폴더 권한과 실제 주의점을 정리한다."
date: 2026-07-19
tags: [Synology, Tailscale, NAS설정, 자체호스팅, NAS보안]
comments: true
share: true
---
![Tailscale Taildrop으로 Synology NAS와 노트북·스마트폰 사이 파일을 전송하는 구성](/assets/images/2026/07/tailscale-taildrop-synology-nas.png)

Tailscale Taildrop을 쓰면 Synology NAS와 내 휴대폰·노트북 사이에 파일을 보낼 때 포트포워딩이나 WebDAV를 열지 않아도 된다. 2026년 1월 5일 기준 Tailscale 공식 문서에서 Synology NAS용 설정이 검증돼 있고, DSM 7.2.2에서 사진 한 장을 외부 네트워크의 휴대폰으로 보내는 흐름을 기준으로 정리했다.

이 그림에서 봐야 할 부분은 NAS의 공유 폴더를 인터넷에 공개하지 않고, 승인된 기기끼리만 파일 경로를 만드는 구성이다.

## 이 방법을 고른 이유

처음에는 NAS에 외부 포트를 열고 File Station 링크를 만드는 게 가장 빠르다고 생각했다. 그런데 가족 휴대폰에 링크를 전달할 때 만료 시간과 계정 권한을 매번 확인해야 했고, 공유 링크가 남는 문제도 있었다. Taildrop은 Tailscale 네트워크(tailnet)에 등록된 기기 사이에서 파일을 넘기는 기능이라 접근 대상을 기기 단위로 줄일 수 있다.

| 방식 | 외부 포트 | 받는 사람 관리 | 적합한 용도 |
|---|---:|---|---|
| Taildrop | 필요 없음 | tailnet 기기·사용자 | 내 기기 간 전송 |
| DSM 공유 링크 | 보통 필요 | 링크·계정 | 잠깐 외부 공유 |
| WebDAV | 필요 | 계정·권한 | 클라이언트 동기화 |

단, Taildrop은 백업이 아니다. 파일을 NAS에 보관하는 기능이 아니라 전송 기능에 가깝기 때문에, 원본과 NAS가 동시에 손상될 상황은 별도 백업으로 대비해야 한다.

## Synology NAS에 Tailscale 설치

DSM 7.2.2에서 관리자 계정으로 패키지 센터를 열고 `Tailscale`을 검색해 설치한다. 앱을 실행한 뒤 로그인하면 NAS가 tailnet의 기기로 등록된다. 휴대폰과 노트북에도 Tailscale 앱을 설치하고 같은 계정으로 로그인한다.

여기서 실패하기 쉬운 지점은 NAS만 로그인하고 클라이언트에는 Tailscale을 설치하지 않는 경우다. Taildrop은 일반 인터넷 파일 업로드가 아니어서, 보내는 기기와 받는 기기가 모두 tailnet에 있어야 한다.

## Taildrop 전용 공유 폴더 만들기

공식 Synology 절차처럼 File Station에서 새 공유 폴더를 만들고 이름을 `Taildrop`으로 지정한다. 폴더 권한은 관리자 전체 권한으로 시작하지 말고 실제 전송에 사용할 계정에만 읽기·쓰기 권한을 준다.

권한을 만든 뒤에는 작은 테스트 파일 하나로 확인한다. NAS 전체 공유 폴더를 Taildrop 대상으로 쓰면 실수로 개인 문서까지 전송할 수 있으므로, 사진 임시 전송이나 모바일 업로드처럼 목적을 좁힌 폴더가 관리하기 쉽다.

## 실제 전송 순서

Tailscale 앱에서 NAS와 휴대폰이 모두 `Connected` 상태인지 확인한다. 휴대폰에서 Tailscale의 파일 전송 메뉴를 열고 NAS를 선택한 뒤 전송할 파일을 고른다. NAS에서는 File Station의 `Taildrop` 폴더에 파일이 도착했는지 확인한다.

전송이 안 될 때는 아래 순서로 범위를 줄이면 된다.

1. 두 기기가 같은 tailnet에 로그인돼 있는지 확인한다.
2. NAS Tailscale 앱에서 기기가 승인 상태인지 확인한다.
3. `Taildrop` 공유 폴더의 사용자 권한을 다시 확인한다.
4. 1GB 영상 대신 1MB 테스트 파일로 재시도한다.

## 운영할 때 주의할 점

Taildrop은 현재 공식 문서에서도 alpha 기능으로 표시돼 있다. 따라서 중요한 원본을 유일한 전송 수단으로 맡기지 않고, 전송 후 파일 크기와 열림 여부를 확인하는 습관이 필요하다. 휴대폰을 분실했다면 Tailscale 관리 화면에서 해당 기기를 비활성화해야 한다. NAS에서 Tailscale을 삭제·재설치하거나 DSM을 크게 업그레이드할 때는 기기 인증이 끊길 수 있으니, 작업 전에 NAS 로컬 접속 방법을 확보해 둔다.

| 점검 항목 | 기준 |
|---|---|
| 인터넷 공개 | 공유기 포트포워딩 없음 |
| 계정 | 사용하지 않는 tailnet 기기 제거 |
| 폴더 | 전송 전용 폴더와 최소 권한 |
| 검증 | 파일 크기·해시·열림 여부 확인 |
| 복구 | 별도 Hyper Backup 또는 외장 디스크 보유 |

Tailscale을 이미 NAS에 설치해 두었다면 Taildrop은 별도 역방향 프록시 없이 구성할 수 있는 짧은 파일 전송 경로다. 다만 alpha 기능이라는 범위를 기억하고, 임시 전송 폴더와 정식 백업 영역을 섞지 않는 것이 실제 운영에서 가장 덜 고생하는 방법이다.

출처: [Tailscale Synology 공식 설정 문서](https://tailscale.com/docs/integrations/synology), [Tailscale NAS 연동 문서](https://tailscale.com/docs/integrations/nas), [Taildrop Synology 설정 문서](https://tailscale.com/docs/features/taildrop/how-to/set-up-taildrop-with-nas)

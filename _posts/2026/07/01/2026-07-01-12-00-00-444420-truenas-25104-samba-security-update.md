---
layout: post
title: "TrueNAS 25.10.4 업데이트 - Samba 보안 패치 전 SMB 공유 점검법"
description: "TrueNAS 25.10.4의 Linux kernel과 Samba 보안 업데이트를 홈서버에 적용하기 전 스냅샷, SMB 세션 정리, 업데이트 후 검증 순서를 정리했다."
date: 2026-07-01
tags: [TrueNAS, NAS보안, 홈서버, SMB, 백업전략]
comments: true
share: true
---
![TrueNAS 홈서버 SMB 공유 보안 업데이트 점검](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=800&q=80)

TrueNAS를 파일 서버로 쓰고 있다면 25.10.4 업데이트는 미루기보다 점검 후 적용하는 쪽이 낫다. 2026년 7월 1일 기준 공식 TrueNAS 25.10.4 공지에는 Linux kernel 6.12 LTS 계열과 Samba 4.22.10 업데이트가 포함됐고, 커널 권한 상승 취약점과 Samba 보안 취약점 수정이 같이 들어갔다.

문제는 홈서버에서 SMB(Samba, Windows 파일 공유 프로토콜)를 가족 PC, 맥북, 미디어 서버가 동시에 물고 있다는 점이다. 그냥 재부팅하면 복사 중이던 파일이 깨지거나, macOS Finder가 이전 세션을 붙잡고 인증 오류를 내는 경우가 있다. [TrueNAS SCALE 설치와 SMB 공유 설정]({% post_url 2026-06-22-10-00-00-222210-truenas-scale-install-smb-share %})까지 마쳤다면 업데이트 전 확인은 더 중요하다.

## 업데이트 전 스냅샷을 남긴다

웹 UI에서 `Datasets`로 들어가 SMB 공유에 쓰는 데이터셋을 고른다. `Create Snapshot`을 누르고 이름은 날짜가 보이게 잡는다.

```text
before-25-10-4-20260701
```

앱 데이터셋과 사용자 파일 데이터셋을 분리해뒀다면 둘 다 찍는다. 이걸 빼먹고 업데이트했다가 SMB 권한이 꼬이면 원인 확인보다 복구가 더 피곤해진다.

## SMB 접속을 끊고 업데이트한다

TrueNAS Shell에서 현재 SMB 세션을 확인한다.

```bash
smbstatus
```

복사 중인 사용자가 있으면 멈춘 뒤 진행한다. 혼자 쓰는 NAS라도 Windows 탐색기 미리보기, Jellyfin 라이브러리 스캔, Paperless-ngx consume 폴더 감시가 세션을 잡고 있을 수 있다. 나는 이걸 안 보고 재부팅했다가 스캔 중이던 PDF 하나가 0바이트로 남은 적이 있다.

업데이트는 `System Settings -> Update`에서 `TrueNAS 25.10.4`를 선택해 적용한다. 이 업데이트는 커널이 바뀌므로 재부팅을 전제로 봐야 한다.

## 재부팅 후 SMB와 권한을 검증한다

업데이트 후 바로 파일 복사를 시작하지 말고 서비스 상태부터 본다.

```bash
systemctl status smbd
smbstatus
```

Windows에서는 기존 네트워크 드라이브를 끊었다가 다시 붙인다.

```cmd
net use Z: /delete
net use Z: \\truenas\share /user:사용자명
```

macOS는 Finder에서 서버 연결을 다시 열고, 같은 파일을 만들고 지우는 테스트를 한다. 읽기만 되는 상태를 정상으로 착각하면 나중에 백업 작업에서 실패한다.

## 주의할 점

TrueNAS 앱이나 VM을 같이 돌리는 서버라면 업데이트 시간을 더 길게 잡아야 한다. 25.10 계열은 NAS 기능만 쓰는 경우보다 컨테이너, VM, iSCSI까지 엮인 구성에서 확인할 게 많다. 특히 SMB 공유를 앱 컨테이너에 마운트해 쓰는 구조라면 앱을 멈춘 뒤 업데이트하는 게 낫다.

롤백은 부팅 환경에서 처리한다. `System Settings -> Boot`에서 이전 boot environment가 남아 있는지 확인해둔다. 업데이트 직후 SMB가 이상하면 설정을 만지기 전에 이전 부팅 환경으로 돌아가는 게 더 깔끔하다.

TrueNAS 25.10.4는 새 기능보다 보안 보수 성격이 강하다. 홈서버에서는 업데이트 버튼보다 스냅샷, SMB 세션 정리, 재부팅 후 쓰기 테스트가 핵심이다.

참고:
- [TrueNAS 25.10.4 공식 공지](https://forums.truenas.com/t/truenas-25-10-4-is-now-available/66244)
- [TrueNAS 25.10 버전 노트](https://www.truenas.com/docs/scale/25.10/gettingstarted/scalereleasenotes/)

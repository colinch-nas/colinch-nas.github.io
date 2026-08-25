---
layout: post
title: "Synology Tailscale 원격 접속 - DSM 7.2.2에서 포트포워딩 없이 NAS 쓰기"
description: "Synology DSM 7.2.2에 Tailscale을 설치하고 포트포워딩 없이 외부에서 NAS와 내부 서비스를 접속하는 설정 순서와 보안 체크 포인트를 정리한다."
date: 2026-08-25
tags: [Synology, DSM, Tailscale, NAS보안, VPN]
comments: true
share: true
---

![Synology NAS 원격 접속을 위한 Tailscale 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

Synology NAS를 외부에서 쓸 때 공유기 5000, 5001 포트를 그대로 열 필요는 없다. DSM 7.2.2 기준으로 Package Center에서 Tailscale을 설치하면 NAS와 노트북·휴대폰을 같은 tailnet(VPN으로 묶인 개인 기기 네트워크)에 넣을 수 있다. 실제로 써보니 DDNS보다 초기 설정이 짧고, 공유기에서 포트포워딩을 지우는 것까지가 핵심이었다.

## 이번 구성

| 항목 | 설정값 |
|---|---|
| NAS | Synology DS923+ |
| 운영체제 | DSM 7.2.2 |
| 원격 접속 | Tailscale 공식 패키지 |
| 공개 포트 | 없음 |
| 접속 대상 | DSM, SMB, Docker 서비스 |

그림에서 봐야 할 부분은 NAS가 인터넷에 포트를 직접 노출하지 않고, 인증된 기기끼리 암호화된 경로를 만든다는 점이다.

## Tailscale 설치와 인증

DSM에서 `패키지 센터 → 유틸리티`로 이동해 Tailscale을 설치한다. 실행 후 `Log in`을 누르고 Google, Microsoft 또는 GitHub 계정으로 인증한다. NAS가 관리자 콘솔의 Machines 목록에 나타나면 연결은 끝난다.

이후 외부 기기에도 Tailscale 앱을 설치하고 같은 계정으로 로그인한다. NAS의 Tailscale IP는 `100.x.x.x` 형태로 표시된다. 집 밖에서 브라우저 주소창에 다음처럼 입력하면 DSM에 접속할 수 있다.

```text
https://100.x.x.x:5001
```

처음에는 인증서 경고가 나올 수 있다. 이는 DSM 인증서가 Tailscale 주소와 일치하지 않아서다. 단순히 DSM 관리 화면과 SMB만 쓸 때는 기기 간 Tailscale 연결 자체가 암호화되므로, 브라우저 경고를 무시하기보다 Tailscale 주소를 북마크하고 관리자 계정에 2단계 인증을 켜는 편이 낫다.

## 내부 서비스까지 접속하기

Tailscale이 설치된 NAS에서는 DSM뿐 아니라 SMB와 Docker 컨테이너도 같은 Tailscale IP로 접근할 수 있다. 예를 들어 NAS의 SMB 주소는 다음처럼 쓴다.

```text
\\100.x.x.x\homes
```

Jellyfin이 NAS의 8096 포트에서 실행 중이면 외부에서도 아래 주소로 접속한다.

```text
http://100.x.x.x:8096
```

휴대폰에서 집 안의 Home Assistant나 프린터까지 접근해야 한다면 NAS를 subnet router(집 안 사설망의 주소를 Tailscale 기기에 전달하는 게이트웨이)로 구성할 수 있다. 다만 이 기능은 NAS에서 라우팅 권한을 추가로 설정해야 하므로, DSM과 NAS 서비스만 필요하면 먼저 사용하지 않는 게 안전하다.

## HTTPS 주소가 필요할 때

Tailscale 관리 콘솔에서 HTTPS certificates를 활성화하면 `tailscale serve`로 tailnet 내부에 HTTPS 주소를 만들 수 있다. NAS의 로컬 8123 포트에서 실행 중인 Home Assistant를 예로 들면 SSH로 NAS에 접속한 뒤 다음 명령을 실행한다.

```bash
sudo tailscale serve --https=443 http://127.0.0.1:8123
```

설정 상태는 다음 명령으로 확인한다.

```bash
sudo tailscale serve status
```

이 방식의 HTTPS는 공개 웹사이트가 아니라 내 tailnet 기기에서만 접근하는 용도다. 불특정 다수가 접속해야 하는 서비스라면 Serve가 아니라 별도 Reverse Proxy(외부 요청을 내부 서비스로 넘기는 중계 서버)와 접근 제어를 검토해야 한다.

## 설정 후 꼭 확인할 것

- 공유기에서 DSM 5000·5001 포트포워딩을 삭제한다.
- DSM 방화벽에서 Tailscale 인터페이스와 필요한 서비스 포트만 허용한다.
- DSM 관리자 계정에 2단계 인증과 자동 차단을 켠다.
- Tailscale 관리자 콘솔에서 사용하지 않는 기기를 폐기한다.
- NAS가 꺼지거나 Tailscale 서비스가 중단되면 원격 접속도 끊긴다. 중요한 파일은 별도 백업을 유지한다.

처음엔 DDNS와 Let's Encrypt 인증서를 세팅하는 게 정석이라고 생각했는데, NAS 관리 화면과 개인용 서비스만 원격에서 쓸 때는 공개 포트를 열지 않는 구성이 관리 부담이 훨씬 적었다. 외부 공개 웹서비스가 아니라 가족 기기 몇 대에서 접속하는 목적이라면 Tailscale부터 적용할 만하다.

관련 문서: [Tailscale의 Synology 원격 접속 안내](https://tailscale.com/docs/integrations/synology), [Tailscale Serve 명령어](https://tailscale.com/docs/reference/tailscale-cli/serve)

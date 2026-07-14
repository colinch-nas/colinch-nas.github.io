---
layout: post
title: "QNAP QuMagie 보안 점검 - License Center까지 같이 업데이트한다"
description: "2026년 6월 QNAP QSA-26-35 권고 기준으로 QuMagie와 License Center를 업데이트하고 외부 공개, 앨범 공유, 관리자 계정을 점검하는 순서."
date: 2026-07-14
tags: [QNAP, NAS보안, QNAP설정, 미디어서버]
comments: true
share: true
---

![QNAP NAS 사진 앱 보안 점검 화면](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?w=800&q=80)

이 그림에서 봐야 할 건 사진 앱 자체보다 외부 접속과 앱 권한이 같은 NAS 안에 묶여 있다는 점이다.

QNAP에서 QuMagie를 쓰고 있다면 오늘 기준으로 App Center에서 QuMagie와 License Center 업데이트부터 해야 한다. 2026년 6월 17일 공개된 QSA-26-35 권고는 QuMagie 2.8.2, 2.9.0과 License Center 1.8.56을 영향 버전으로 적고 있다. 특히 QuMagie 쪽은 로그인 전 미디어 파일, 얼굴 인식 썸네일, 앨범 아카이브가 노출될 수 있는 유형이라 집 사진을 NAS에 모아둔 사람에겐 가볍지 않다.

내 기준 환경은 QNAP TS-464, QTS 5.2.x, QuMagie 2.x, myQNAPcloud는 꺼둔 내부망 NAS다. 처음엔 "사진 앱을 외부에 안 열었으니 괜찮겠지"라고 생각했다. 그런데 공유기에서 예전에 만든 포트포워딩, myQNAPcloud Link, 역방향 프록시 설정이 남아 있으면 앱 업데이트만으로 끝낼 문제가 아니다.

| 점검 항목 | 확인 위치 | 기준 |
|---|---|---|
| QuMagie 버전 | App Center -> QuMagie | 2.9.1 또는 2.10.0 이상 |
| License Center 버전 | App Center -> License Center | 2.0.42 이상 |
| 외부 공개 | myQNAPcloud, 공유기 NAT | QuMagie 직접 공개 없음 |
| 앨범 공유 | QuMagie 공유 링크 | 만료일 없는 링크 삭제 |
| 관리자 계정 | Control Panel -> Users | 사진 앱용 계정과 분리 |

작업 순서는 단순하게 잡는다. 관리자 계정으로 QTS에 들어가서 `App Center`를 열고 `QuMagie`를 검색한 뒤 업데이트한다. 같은 방식으로 `License Center`도 업데이트한다. QNAP 권고에는 QuMagie 2.8.2는 2.9.1, QuMagie 2.9.0은 2.10.0, License Center 1.8.56은 2.0.42에서 수정됐다고 적혀 있다. 업데이트 버튼이 안 보이면 이미 최신이거나 모델 지원이 끊긴 경우라 `Control Panel -> System -> Firmware Update`와 제품 지원 상태를 같이 확인한다.

업데이트 뒤에는 외부 접속을 끊어야 한다. QuMagie를 가족에게 보여주려고 8080, 443, 임의 포트를 열어둔 적이 있다면 공유기 NAT부터 본다. 나는 사진 앱은 포트포워딩하지 않고 Tailscale VPN으로만 접근하게 둔다. 외부 공유가 꼭 필요하면 QuMagie 직접 공개 대신 만료일 있는 앨범 링크를 짧게 만들고, 공유가 끝나면 바로 지운다.

공유기에서 열린 포트를 빠르게 확인하려면 NAS 내부에서 목록을 적어두고 대조한다.

```bash
# QNAP SSH에서 현재 NAS가 듣는 포트 확인
netstat -tulpn | grep LISTEN
```

여기서 `8080`, `443`, `5000` 같은 관리 포트가 보이는 것 자체는 정상일 수 있다. 문제는 이 포트가 공유기에서 인터넷으로 그대로 열려 있는지다. 공유기 관리자 화면의 포트포워딩 목록과 myQNAPcloud 자동 라우터 설정을 같이 본다. UPnP를 켜둔 경우 앱이 포트를 다시 열 수 있어 이 부분에서 한 번 삽질했다.

QuMagie 안에서는 공유 앨범을 정리한다. `Shared Albums`나 공유 링크 목록에서 만료일이 없거나 목적이 끝난 링크를 삭제한다. 얼굴 인식 썸네일은 원본 사진보다 덜 민감해 보이지만, 가족 구성과 장소를 추정할 수 있어 노출되면 꽤 곤란하다. 사진 폴더 권한도 `everyone` 읽기 허용으로 두지 않는다.

주의할 점은 세 가지다. App Center 업데이트 뒤에도 QTS 자체가 오래되면 다른 취약점이 남는다. 관리자 계정으로 QuMagie를 평소 사용하면 License Center 같은 시스템 앱 문제가 같이 커진다. 외부 접속을 VPN으로 바꿨더라도 휴대폰 앱에서 기존 클라우드 연결이 살아 있는지 확인해야 한다.

짧게 정리하면 이렇다. QuMagie는 2.9.1 또는 2.10.0 이상, License Center는 2.0.42 이상으로 올린다. 포트포워딩과 myQNAPcloud 공개를 끊고, 사진 공유 링크는 만료일 있는 것만 남긴다. QNAP NAS 보안은 펌웨어 업데이트보다 "사진 앱이 밖에서 보이는가"를 같이 봐야 실제 위험이 줄어든다.

확인 출처: [QNAP QSA-26-35 보안 권고](https://www.qnap.com/en-us/security-advisory/qsa-26-35)

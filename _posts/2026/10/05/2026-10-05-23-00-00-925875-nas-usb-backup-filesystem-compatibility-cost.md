---
layout: post
title: "Synology DSM 7·QNAP QTS 5·TrueNAS SCALE USB 백업 파일시스템 - exFAT·NTFS·ext4 복원 호환성과 총비용"
description: "Synology DSM 7, QNAP QTS 5, TrueNAS SCALE에서 USB 백업 디스크를 exFAT·NTFS·ext4 중 무엇으로 골라야 하는지 복원 호환성, 제한, 라이선스와 총비용으로 판단한다."
date: 2026-10-05
tags: [Synology, QNAP, TrueNAS, DSM, NAS보안]
comments: true
share: true
---

![NAS USB 백업 디스크와 파일시스템 선택](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

NAS와 분리 가능한 USB 디스크를 함께 두고, 파일시스템 선택이 복원 경로에 어떤 차이를 만드는지 보여주는 이미지다.

Windows·macOS에서도 USB 백업 디스크를 읽어야 하면 exFAT, NAS에 계속 연결해 백업하고 권한·링크를 보존해야 하면 ext4 계열을 우선 검토하는 편이 낫다. 다만 TrueNAS의 USB 디스크 백업은 공식 문서에서도 복원 문제를 경고하므로, 주 백업을 USB 한 장에만 두는 구성은 피해야 한다. 아래 내용은 2026년 10월 5일 공식 문서 기준이며, 특정 장비를 직접 연결해 시험한 결과는 아니다.

## 파일시스템 선택표

| 선택지 | 장점 | 걸리는 조건 | 적합한 경우 |
|---|---|---|---|
| exFAT | Windows·macOS와 교차 사용, 4GB 초과 파일 | Synology는 exFAT Access 패키지, 구형 QNAP은 라이선스 필요 | 사진·영상 원본을 PC에서도 읽을 때 |
| NTFS | Windows에서 바로 읽고 복구 도구가 많음 | NAS에서 권한·메타데이터가 제한될 수 있음 | Windows PC로 최종 복구할 때 |
| ext4 | Linux·NAS 백업에 자연스럽고 비용 없음 | macOS·Windows에서 바로 읽기 어려움 | USB를 NAS 전용 백업 디스크로 둘 때 |
| FAT32 | 폭넓은 호환성 | 단일 파일 4GB 제한 | 작은 문서만 옮길 때 |

Synology DSM 7.2는 USB에서 Btrfs, ext3, ext4, FAT32, exFAT, HFS Plus, NTFS를 인식하지만 exFAT 데이터 접근에는 `exFAT Access` 패키지가 필요할 수 있다. QNAP은 QTS 5.0.1 이상 ARM 모델과 QTS 5.0 이상 x86 모델에서 exFAT를 기본 지원한다고 안내하지만, 구형 버전은 라이선스가 필요하고 그 라이선스는 다른 NAS로 옮길 수 없다. 파일시스템 이름만 보고 “어느 NAS에서나 복원된다”고 가정하면 안 되는 이유다.

## 용도별 판단

가상 사례로 사용 데이터가 6TB이고 USB 백업 디스크가 NAS에 연결된 상태라고 하자. Windows PC에서도 가끔 파일을 꺼내야 한다면 8TB exFAT 디스크가 접근성은 좋다. 반면 매일 버전 백업을 쌓는다면 8TB가 곧 8TB의 백업 공간이라는 뜻은 아니다. 변경분, 보존 버전, 암호화 컨테이너가 추가되므로 최소 30% 이상의 여유 공간을 두고 보존 정책을 먼저 정해야 한다.

| 운영 규모 | 권장 구성 | 구매하지 않아도 되는 것 |
|---|---|---|
| 1인·문서 중심 | 8TB 외장 HDD 1개 + 주 1회 분리 보관 | NAS 전용 백업 앱이 없어도 단순 파일 복사 가능 |
| 2~3인·사진/영상 | NAS 전용 ext4 또는 앱 백업 + 별도 exFAT 이동 디스크 | 모든 백업을 SSD로 살 필요 없음 |
| 4인 이상·컨테이너 운영 | NAS 백업과 별도 NAS·클라우드 중 하나 추가 | USB 한 장만으로 3-2-1을 완성했다고 볼 수 없음 |

총비용은 `디스크 가격 + USB 케이스/전원 + exFAT 라이선스(필요한 경우) + 두 번째 복사본의 월 비용 + 복구에 걸리는 시간`으로 계산한다. exFAT의 교차 호환성 때문에 추가 디스크를 사지 않아도 되는 환경이면 비용이 줄지만, NAS 고장 때 전용 백업 형식이나 암호화 키를 잃으면 복구 작업 자체가 가장 큰 비용이 된다.

## 복원 전 체크리스트

- [ ] 백업 디스크의 파일시스템과 백업 앱 형식을 기록한다.
- [ ] 복원할 NAS 모델·DSM/QTS 버전에서 해당 파일시스템을 실제로 인식하는지 확인한다.
- [ ] 암호화 백업이면 비밀번호·키 파일을 NAS와 다른 장소에 보관한다.
- [ ] 1GB 파일 1개가 아니라 문서, 사진, 권한이 있는 폴더를 각각 복원한다.
- [ ] 복원한 파일의 개수·해시·열림 여부를 원본과 대조한다.
- [ ] TrueNAS라면 USB 디스크를 주 저장소로 쓰지 않고, 복사 후 분리 보관한다.

결정 기준은 간단하다. **PC에서 바로 읽는 백업**이면 exFAT 또는 NTFS, **NAS 전용 버전 백업**이면 ext4와 백업 앱, **TrueNAS의 장기 보관**이면 USB 한 장보다 다른 ZFS 시스템이나 클라우드 복제까지 포함해야 한다. 싼 파일시스템보다 실제 복원 경로가 짧은 구성이 총비용을 낮춘다.

## 공식 근거

- [Synology DSM 7.2 사용자 가이드 — 외부 장치](https://global.download.synology.com/download/Document/Software/UserGuide/Os/DSM/7.2/krn/Syno_UsersGuide_NAServer_7_2_krn.pdf)
- [QNAP exFAT 공식 안내](https://www.qnap.com/ko-kr/software/exfat)
- [QNAP QTS 5.2 외부 저장장치 포맷 안내](https://docs.qnap.com/operating-system/qts/5.2.x/ko-kr/GUID-04D7A1AF-9275-4FC9-8C32-607B649E9708.html)
- [TrueNAS Hardware Guide — USB 저장장치 백업 주의사항](https://www.truenas.com/docs/scale/gettingstarted/tnhardwareguide/)

---
layout: post
title: "QNAP QuTS hero h6.0 High Availability 설정 - 두 대 NAS 장애 전환 전 점검"
description: "2026년 5월 정식 출시된 QNAP QuTS hero h6.0의 High Availability를 실제 홈서버에 적용하기 전, 동일 장비·디스크·네트워크 조건과 설정 순서를 정리한다."
date: 2026-07-20
tags: [QNAP, NAS설정, NAS보안, 홈서버구축, HomeLab]
comments: true
share: true
---

![QNAP QuTS hero h6.0 이중 NAS 홈서버 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)
이 그림에서 볼 것은 NAS 한 대를 빠르게 만드는 구성이 아니라, 두 장비 사이에 데이터를 계속 맞추는 구조라는 점이다.

QNAP이 2026년 5월 29일 QuTS hero h6.0을 정식 출시하면서 High Availability(HA, 한 대가 멈추면 다른 한 대가 서비스를 이어받는 구성)를 전면에 내세웠다. 홈서버에서도 매력적으로 보이지만, NAS 두 대만 사서 켜면 끝나는 기능은 아니다. 공식 조건상 QuTS hero 모델 두 대의 CPU, 메모리, 디스크 수와 용량, 네트워크 인터페이스까지 맞아야 한다.

## 내 기준 구성과 먼저 확인할 것

실제 적용 전 가상으로 잡은 기준은 다음과 같다.

| 항목 | 기준 |
|---|---|
| 운영체제 | QuTS hero h6.0.x |
| 장비 | 같은 QNAP QuTS hero 지원 모델 2대 |
| 디스크 | 같은 개수·용량, 같은 슬롯 배치 |
| 케이블 | 하트비트 1개 + 클러스터 2개 |
| 스위치 | 관리형 스위치, 고정 IP 3개 이상 |

여기서 가장 많이 하는 착각은 TS-464 한 대와 다른 모델 한 대를 묶을 수 있다고 생각하는 것이다. QTS와 QuTS hero를 섞을 수도 없다. 두 노드의 펌웨어 버전도 먼저 맞추고, 기존 데이터가 있는 장비를 바로 클러스터에 넣지 않는 편이 안전하다. 초기 구성 과정에서 디스크가 초기화될 수 있기 때문이다.

## High Availability 설정 순서

두 NAS를 같은 스위치에 연결하되 관리 트래픽과 HA 트래픽을 구분한다. 하트비트는 두 장비의 상태를 확인하고 동기화하는 전용 연결이다. 클러스터 연결은 외부에서 공유 폴더와 서비스에 접근할 때 사용한다.

1. 두 NAS의 `Control Panel → System → Firmware Update`에서 QuTS hero h6.0.x와 릴리스 노트를 확인한다.
2. 각 NAS에 관리용 고정 IP를 지정한다. DHCP 예약만 믿지 말고 NAS 화면에서도 주소를 확인한다.
3. `App Center`에서 **High Availability Manager**를 설치하고 두 장비 모두 재부팅한다.
4. 주 노드에서 `High Availability Manager → Create HA Cluster`를 연다.
5. 상대 NAS의 관리 IP와 관리자 권한이 있는 별도 계정을 입력한다. 기본 `admin` 계정을 계속 쓰지는 않는다.
6. 하트비트 인터페이스와 클러스터 인터페이스를 각각 선택하고 사전 검사 결과를 확인한다.
7. 디스크 배치, 메모리, 네트워크 인터페이스 경고가 모두 사라진 뒤 클러스터를 생성한다.

설정 후에는 공유 폴더 하나만 만들어 파일을 복사하고, 활성 노드의 네트워크를 잠시 끊어 장애 전환을 확인한다. 브라우저 세션이 끊기는 것과 데이터가 사라지는 것은 다른 문제라서, SMB 복사·Docker 컨테이너·예약 백업을 각각 테스트해야 한다.

## 이 구성에서 삽질하기 쉬운 부분

HA는 백업이 아니다. 두 노드의 삭제나 암호화도 동기화될 수 있으므로 HBS 3, Snapshot Replica, 외장 디스크 중 하나를 별도로 둬야 한다. 또한 두 노드의 네트워크 인터페이스 구성이 다르면 패시브 노드가 끊겼을 때 게이트웨이·VLAN·포트 트렁킹 변경이 막힐 수 있다.

가정용이라면 HA 비용이 과하다. NAS 한 대와 외장 HDD 백업으로도 정전·디스크 고장에는 충분할 수 있다. 반대로 CCTV나 홈오토메이션처럼 몇 분의 중단도 곤란하고 동일 장비를 이미 보유했다면, 먼저 테스트용 공유 폴더로 장애 전환을 검증한 뒤 운영 데이터를 옮기는 순서가 맞다.

짧게 정리하면 QuTS hero h6.0 HA의 핵심은 기능을 켜는 버튼이 아니라 두 노드의 동일성, 세 개의 네트워크 연결, 별도 백업이다. 이 세 가지를 준비하지 못했다면 HA보다 3-2-1 백업을 먼저 완성하는 편이 낫다.

출처: [QNAP QuTS hero h6.0 정식 출시 안내](https://www.qnap.com/en-us/news/2026/qnap-officially-releases-quts-hero-h6-0-official-featuring-dual-nas-high-availability-immutable-snapshots-and-more), [QNAP High Availability 요구사항](https://docs.qnap.com/application/ha-manager/1.x/en-us/ha-requirements-202A6F9A.html), [QuTS hero h6.0 네트워크 요구사항](https://docs.qnap.com/operating-system/quts-hero/6.0.x/en-us/about-network-amp-virtual-switch-AF9EF96D.html)

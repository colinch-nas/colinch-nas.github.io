---
layout: post
title: "TrueNAS Home Assistant 앱 종료 예고 - VM으로 옮기는 백업·복구 순서"
description: "TrueNAS Apps의 Home Assistant 2026.8.3 앱이 2026년 11월 26일 제거될 예정이다. 기존 설정과 PostgreSQL 이력을 보존하면서 Home Assistant OS VM으로 이전하는 실제 점검 순서를 정리한다."
date: 2026-08-28
tags: [TrueNAS, HomeAssistant, 홈서버, 가상머신, NAS설정, 백업전략]
comments: true
share: true
---

![TrueNAS 홈서버에서 Home Assistant OS 가상머신으로 이전하는 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

그림에서 봐야 할 부분은 저장소와 스마트홈 제어기가 한 장비에 있어도 Home Assistant를 별도 VM으로 분리하면 네트워크와 업데이트를 관리하기 쉽다는 점이다.

TrueNAS Apps의 **Home Assistant 앱은 2026년 8월 26일 deprecated(지원 중단 예고)**로 표시됐고, 공식 카탈로그의 제거 예정일은 **2026년 11월 26일**이다. 현재 앱 버전은 2026.8.3이지만 새 설치라면 Home Assistant OS를 KVM 가상머신으로 올리는 편이 낫다. 기존 사용자는 백업만 만들고 앱을 지우면 안 된다. 앱의 PostgreSQL 기록과 장기 통계는 일반 Home Assistant 백업에 포함되지 않는다.

## 이전 구성을 잡는 기준

TrueNAS SCALE 25.10.6, ZFS 풀, RAM 8GB 이상인 미니PC를 기준으로 했다. VM은 CPU 2코어, 메모리 4GB, 디스크 32GB로 시작하면 조명·센서 자동화에 충분하다. 카메라 기록까지 많으면 6GB를 배정한다.

| 항목 | 기존 앱 | Home Assistant OS VM |
|---|---|---|
| 설치 | Apps 카탈로그 | KVM `.qcow2` 이미지 |
| 애드온 | 카탈로그 의존 | Supervisor에서 관리 |
| 검색·연결 | mDNS·Matter가 네트워크 영향을 받음 | 브리지 네트워크·USB 패스스루 |
| 데이터 | HA 설정 + PostgreSQL | HA 백업 + VM 백업 |

TrueNAS는 VM 전환 이유로 mDNS, SSDP, Bluetooth, HomeKit, Thread/Matter 검색과 Zigbee·Z-Wave USB 패스스루를 안내한다. 컨테이너 네트워크를 바꾼 뒤 ESPHome 장치 검색이 늦어졌던 환경이라면 특히 차이가 난다.

## 앱 삭제 전 백업

Home Assistant에서 **설정 → 시스템 → 백업**으로 전체 백업을 만들고 PC나 다른 풀에 내려받는다. 이어 앱의 PostgreSQL을 별도로 덤프한다. 컨테이너 이름과 DB 사용자는 설치 환경마다 다르므로 TrueNAS 앱 상세 화면에서 먼저 확인한다.

```bash
# 실제 컨테이너 이름을 확인한다.
docker ps --format '{{.Names}}'

# 아래 이름과 DB 값은 실제 환경에 맞게 바꾼다.
docker exec -t home-assistant-postgres \
  pg_dump -U home-assistant -d home-assistant \
  > /mnt/tank/backup/ha-postgres-2026-08-28.sql
```

TrueNAS Apps에서 Docker 명령이 바로 실행되지 않으면 앱의 **Shell** 메뉴나 카탈로그 문서를 이용한다. 이름을 추측해 덤프하면 0바이트 파일만 남는 경우가 있다. 백업 파일은 TrueNAS 풀 한 곳에만 두지 않는다.

## Home Assistant OS VM 만들기

Home Assistant 공식 Linux 설치 페이지에서 KVM용 `.qcow2` 이미지를 내려받는다. TrueNAS의 **Virtualization → Virtual Machines → Add**에서 아래처럼 설정한다.

| 설정 | 값 |
|---|---|
| 펌웨어 | UEFI |
| CPU·RAM | 2 vCPU·4096MB |
| 디스크 | 내려받은 qcow2 이미지 |
| 네트워크 | 브리지 + DHCP |
| VNC | 설치할 때만 활성화 |

VM 디스크는 `tank/vm/home-assistant`처럼 별도 데이터셋에 둔다. VM을 시작하고 공유기 DHCP 목록에서 IP를 확인한 뒤 `http://할당된-IP:8123`으로 접속한다. 온보딩 화면에서 기존 전체 백업을 복원하면 된다.

Zigbee 코디네이터는 VM 편집 화면의 USB 장치에서 하나만 선택한다. TrueNAS 앱이나 Zigbee2MQTT 컨테이너가 같은 동글을 동시에 잡으면 `adapter not found`가 발생한다. 복원 후 **설정 → 시스템 → 하드웨어**에서 동글을 확인하고, 센서 이벤트와 자동화 한 개를 실제로 실행한다.

## 삭제 시점과 점검표

VM에서 장치, 자동화, Matter·ESPHome 검색이 모두 정상일 때 기존 앱을 삭제한다. 다른 VLAN의 장치가 안 보이면 Home Assistant 문제가 아니라 방화벽과 mDNS 반사 설정일 수 있다. VM 스냅샷만 백업으로 믿지 말고, 복원된 VM에서 새 HA 백업을 만든 뒤 다른 위치에 복사한다.

- [ ] 전체 HA 백업을 PC 또는 다른 풀에 저장
- [ ] PostgreSQL 덤프 파일 크기 확인
- [ ] VM 브리지 IP와 `8123` 접속 확인
- [ ] Zigbee 동글을 VM 하나에만 연결
- [ ] 자동화·장치 검색·기록 화면 테스트

TrueNAS의 Home Assistant 앱은 2026년 11월 26일 제거 예정이므로 지금 즉시 멈추는 것은 아니다. 새 설치는 HAOS VM으로 시작하고, 기존 사용자는 `전체 백업 → PostgreSQL 덤프 → VM 복원 → Zigbee·Matter 테스트 → 앱 삭제` 순서로 옮기면 된다. 참고 자료는 [TrueNAS Home Assistant 카탈로그](https://apps.truenas.com/catalog/home-assistant_stable/)와 [Home Assistant Linux/VM 설치 문서](https://www.home-assistant.io/installation/linux/)다.

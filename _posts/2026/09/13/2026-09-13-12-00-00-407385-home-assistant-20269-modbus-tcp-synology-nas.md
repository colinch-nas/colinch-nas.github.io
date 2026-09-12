---
layout: post
title: "Home Assistant 2026.9 Modbus TCP 설정 - Synology NAS에서 전력계 연결하기"
description: "Home Assistant 2026.9.2의 Modbus 변경점을 기준으로 Synology NAS Container에서 Modbus TCP 전력계를 연결하고, 값이 틀릴 때 점검하는 방법을 정리했다."
date: 2026-09-13
tags: [HomeAssistant, Synology, NAS설정, 에너지모니터링, Docker]
comments: true
share: true
---

![Synology NAS와 Modbus TCP 전력계 연결 구성](/assets/images/2026/09/home-assistant-modbus-tcp-synology-nas.png)

Home Assistant 2026.9부터 Modbus 장치를 무조건 긴 YAML로만 붙여야 한다는 부담이 조금 줄었다. Fronius, Sofar 같은 제조사 통합은 장치가 프로토콜을 알아서 처리하고, 직접 연결해야 하는 일반 전력계는 기존 `modbus` 설정을 그대로 쓸 수 있다. Synology NAS에서 Home Assistant Container를 운영 중이라면 이번 글의 핵심은 **NAS가 전력계를 직접 읽는 것이 아니라, 같은 네트워크의 TCP 502 포트로 읽게 만드는 것**이다.

그림에서 볼 부분은 NAS와 전력계가 같은 스위치에 연결되어 있고, Home Assistant가 전력 데이터를 대시보드로 모으는 구조다.

## 2026.9에서 달라진 점

2026.9 릴리스의 Modbus 개선은 새로운 센서 등록 화면 하나가 생겼다는 뜻보다는, 여러 통합이 하나의 Modbus 연결을 공유할 수 있는 기반이 마련됐다는 의미에 가깝다. 9월 11일 공개된 2026.9.2는 버그 수정 패치이므로, 아래 설정은 `ghcr.io/home-assistant/home-assistant:2026.9.2` 기준으로 고정했다.

공식 문서도 전용 제조사 통합이 있으면 수동 Modbus보다 그것을 먼저 찾으라고 안내한다. 레지스터 주소를 직접 해석해야 하는 전력계나 태양광 인버터만 일반 Modbus 설정으로 들어가는 편이 유지보수하기 쉽다.

## 준비물과 네트워크 확인

| 항목 | 확인 기준 |
|---|---|
| Synology | DSM 7.2 이상, Container Manager 설치 |
| Home Assistant | Container 2026.9.2, `network_mode: host` |
| 전력계 | Modbus TCP 지원, 장치 IP 고정, 포트 502 |
| 설정값 | Unit ID, 레지스터 주소, 데이터 타입, 배율 |

전력계의 IP를 공유기에서 DHCP 예약으로 고정한다. NAS에서 포트가 열렸는지는 다음처럼 확인할 수 있다.

```bash
nc -vz 192.168.10.50 502
```

`succeeded`가 나오지 않으면 Home Assistant YAML을 고쳐도 해결되지 않는다. 전력계의 TCP 기능이 켜져 있는지, NAS와 장치가 VLAN(네트워크를 논리적으로 나눈 구역) 사이에서 통신 가능한지부터 확인한다.

## Synology Container Manager에 컨테이너 만들기

Container Manager의 프로젝트에서 아래 Compose를 사용했다. `/volume1/docker/homeassistant`는 실제 NAS 공유 폴더 경로에 맞춰 바꾼다.

```yaml
services:
  homeassistant:
    container_name: homeassistant
    image: ghcr.io/home-assistant/home-assistant:2026.9.2
    network_mode: host
    volumes:
      - /volume1/docker/homeassistant:/config
      - /etc/localtime:/etc/localtime:ro
    environment:
      TZ: Asia/Seoul
    restart: unless-stopped
```

`host` 네트워크를 쓴 이유는 Modbus TCP 때문만은 아니다. Home Assistant의 일부 자동 검색과 로컬 장치 탐색이 브리지 네트워크에서 누락되는 경우가 있어, NAS에서 단독으로 운영할 때는 이 구성이 덜 삽질한다. 단, 컨테이너 방식은 Home Assistant OS처럼 앱(Add-on)과 원클릭 백업을 제공하지 않으므로 NAS Hypervisor에 HAOS 가상 머신을 두는 선택지도 남겨둬야 한다.

## Modbus TCP 센서 등록

전력계 제조사 매뉴얼에서 예시로 `40001`이라고 적힌 주소가 Home Assistant의 `0`인지 `1`인지가 가장 자주 틀리는 부분이다. 아래는 유효전력 레지스터가 `40001`, 32비트 signed 값, 10배 스케일이라는 가정의 예시다.

```yaml
modbus:
  - name: energy_meter
    type: tcp
    host: 192.168.10.50
    port: 502
    timeout: 5
    sensors:
      - name: house_power
        slave: 1
        address: 0
        input_type: holding
        data_type: int32
        swap: word
        scale: 0.1
        unit_of_measurement: W
        device_class: power
        state_class: measurement
```

설정 후 개발자 도구에서 YAML 구성 검사를 실행하고 Home Assistant를 재시작한다. 값이 `6553.5W`처럼 비정상적으로 크면 `scale`, `data_type`, `swap` 순서로 확인한다. 음수와 양수가 뒤집히면 signed/unsigned를 잘못 선택했을 가능성이 높다.

## 실패했던 지점과 보안 기준

처음에는 NAS 방화벽에서 502 포트를 인터넷에 열면 외부에서도 전력계를 볼 수 있을 거라 생각하기 쉽다. 그럴 필요가 없다. Modbus TCP는 인증이 없는 장치가 많으므로 공유기 포트포워딩을 하지 말고, 외부 접속은 이미 구성한 Tailscale VPN이나 역방향 프록시 인증 뒤에서 Home Assistant만 노출한다. 전력계는 내부 VLAN에 두고 NAS IP에서만 502를 허용하는 편이 안전하다.

또 하나는 센서 이름을 바꾼 뒤 통계가 끊기는 문제다. 에너지 대시보드에 넣을 센서는 처음부터 `device_class`, `state_class`, 단위를 정확히 지정하고, 누적 kWh 레지스터에는 `state_class: total_increasing`을 사용한다. 순간 전력 W를 누적값으로 등록하면 그래프가 그럴듯해 보여도 비용 계산은 틀어진다.

정리하면, 제조사 통합이 있으면 그것을 사용하고, 일반 계측기는 고정 IP·502 포트·레지스터 표를 먼저 확보한 뒤 NAS Container에서 Modbus TCP를 붙이면 된다. 2026.9.2에서는 여러 Modbus 통합이 같은 연결을 공유하는 방향이 강화됐지만, 실제 값의 정확성은 여전히 장치 매뉴얼과 레지스터 오프셋에 달려 있다.

참고 자료: [Home Assistant 2026.9 릴리스 노트](https://www.home-assistant.io/blog/2026/09/02/release-20269/), [Modbus 통합 공식 문서](https://www.home-assistant.io/integrations/modbus/), [Home Assistant 설치 방식](https://www.home-assistant.io/installation/)

---
layout: post
title: "Home Assistant 2026.9 Modbus 설정 - Synology NAS에서 전력계 연결하기"
description: "Home Assistant 2026.9.1의 Modbus 개선점을 기준으로 Synology NAS Docker 환경에서 태양광 인버터와 전력계를 연결하고 에너지 대시보드에 표시하는 방법을 정리한다."
date: 2026-09-11
tags: [HomeAssistant, Synology, Docker, 홈서버, 에너지모니터링]
comments: true
share: true
---
![Home Assistant 전력 모니터링 대시보드](https://images.unsplash.com/photo-1473341304170-971dccb5ac1e?w=1200&q=80)

Home Assistant 2026.9.1을 Synology NAS의 Docker에서 운영 중이라면 Modbus 장비를 붙여 전력 사용량과 태양광 발전량을 한 화면에서 볼 수 있다. 이번 버전은 Modbus 연결을 여러 통합이 공유할 수 있게 정리했고, Fronius와 Sofar 같은 장비는 YAML 레지스터를 직접 외우지 않고 UI에서 추가하는 흐름이 생겼다.

이 글은 DSM 7.4, Home Assistant Container 2026.9.1, 같은 LAN에 있는 Modbus TCP 전력계라는 조건으로 작성했다. RS-485 USB 동글을 NAS에 꽂는 경우는 컨테이너 장치 권한까지 따로 확인해야 한다.

그림에서 볼 부분은 NAS가 데이터를 외부 클라우드로 보내는 대신 집 안의 전력계에서 값을 읽어 에너지 화면에 쌓는 구조다.

## 장비가 지원되는지 먼저 확인한다

처음엔 모든 전력계를 같은 방식으로 연결할 수 있을 줄 알았는데, 실제로는 제조사별 레지스터 주소와 데이터 형식이 달랐다. 아래처럼 장비 유형에 따라 시작점이 다르다.

| 장비 | 2026.9에서 권장하는 시작점 | 필요한 정보 |
|---|---|---|
| Fronius 인버터 | 설정 → 기기 및 서비스 → 통합 추가 → Fronius | 인버터 IP, Modbus TCP 활성화 |
| Sofar 인버터 | Sofar 통합 추가 | 인버터 IP 또는 Modbus 브리지 IP |
| 일반 전력계 | Modbus YAML 설정 | IP, 포트 502, slave ID, 레지스터 맵 |
| RS-485 전력계 | USB-RS485 + serial 설정 | `/dev/ttyUSB0`, baudrate, parity |

Fronius의 경우 2026.9에서 Modbus TCP(SunSpec)를 선택적으로 켤 수 있고, 기존 HTTP API에서 보이지 않던 문자열별 전압·전류·전력·누적 에너지도 읽을 수 있다. 반면 일반 전력계는 여전히 제조사 매뉴얼의 레지스터 표가 기준이다.

## NAS에서 Modbus TCP 연결하기

전력계가 LAN에 있고 Modbus TCP를 지원한다면 Synology NAS의 컨테이너가 같은 네트워크의 502 포트로 접근하면 된다. Docker를 `host` 네트워크로 바꿀 필요는 없다. 오히려 기존 포트 설정을 건드리다가 Home Assistant 접속이 끊기는 경우가 더 많았다.

지원되는 인버터라면 Home Assistant에서 통합을 추가하고 장비 IP를 입력하는 것으로 끝난다. 일반 장비는 `configuration.yaml`에 아래처럼 통신부와 센서를 추가한다. 주소 100과 `uint32`는 예시이므로 전력계 매뉴얼 값으로 교체해야 한다.

```yaml
modbus:
  - name: home_power_meter
    type: tcp
    host: 192.168.10.50
    port: 502
    timeout: 5
    message_wait_milliseconds: 30
    sensors:
      - name: house_power
        address: 100
        input_type: input
        slave: 1
        data_type: uint32
        unit_of_measurement: W
        device_class: power
        state_class: measurement
        scan_interval: 15
```

YAML을 저장한 뒤 설정 → 시스템 → 서버 제어 → 구성 확인을 실행하고, 오류가 없을 때 Home Assistant를 재시작한다. 센서가 만들어졌는데 값이 `unknown`이면 IP보다 `input_type`, slave ID, 레지스터 주소, 16비트/32비트 순서를 먼저 의심하는 편이 빠르다.

## 에너지 대시보드에 넣을 때의 함정

전력(W)과 누적 에너지(kWh)는 다른 센서다. 순간 전력 센서를 에너지 대시보드에 바로 넣으면 값이 이상하게 누적된다. 전력계가 누적 kWh 레지스터를 제공하면 그 센서에 `device_class: energy`, `state_class: total_increasing`, `unit_of_measurement: kWh`를 적용한다. 누적값이 없고 W만 있다면 별도의 적분 센서가 필요하다.

설정 전후 확인 체크리스트는 다음과 같다.

- [ ] 전력계 고정 IP와 Home Assistant 컨테이너의 접근 가능 여부 확인
- [ ] 장비 매뉴얼의 Modbus TCP 활성화와 포트 502 확인
- [ ] slave ID와 레지스터 주소를 0-based/1-based 중 맞는 방식으로 입력
- [ ] 순간 전력에는 `power`, 누적 전력량에는 `energy` 클래스 사용
- [ ] 외부 포트포워딩 없이 LAN에서만 먼저 정상 동작 확인

여기서 외부 공개는 하지 않는 게 좋다. Modbus TCP 포트 502를 공유기에 그대로 열면 전력계 제어 레지스터까지 인터넷에 노출될 수 있다. 원격에서 확인해야 한다면 Tailscale 같은 VPN으로 NAS에 접속한 뒤 Home Assistant를 여는 쪽이 안전하다.

## 정리

Home Assistant 2026.9의 변화는 단순히 새 센서 하나가 추가된 것이 아니다. 지원되는 인버터는 UI 통합으로 진입 장벽이 낮아졌고, Synology NAS에 Docker로 올린 Home Assistant도 로컬 전력 데이터를 안정적으로 모을 수 있다. 다만 일반 Modbus 장비는 레지스터 맵을 대신 찾아주는 기능이 아니므로, 주소와 데이터 형식을 확인하는 시간이 가장 중요하다.

참고한 공식 문서: [Home Assistant 2026.9 릴리스 노트](https://www.home-assistant.io/blog/2026/09/02/release-20269/), [Modbus 통합 문서](https://www.home-assistant.io/integrations/modbus/)

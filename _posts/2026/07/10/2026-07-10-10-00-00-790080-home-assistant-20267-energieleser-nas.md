---
layout: post
title: "Home Assistant 2026.7 energieleser - NAS Docker에서 전력계 내부망으로 붙이기"
description: "Home Assistant 2026.7 새 energieleser 통합을 NAS Docker 환경에서 내부망 HTTP 장비로 등록하고 에너지 대시보드까지 확인하는 순서."
date: 2026-07-10
tags: [HomeAssistant, Docker, 스마트홈, 에너지모니터링]
comments: true
share: true
---
![NAS에서 Home Assistant 에너지 계측 데이터를 확인하는 화면](https://images.unsplash.com/photo-1558618666-fcd25c85cd64?w=800&q=80)
이 그림에서는 전력계가 클라우드가 아니라 집 안 네트워크와 홈서버로 이어지는 흐름을 봐야 한다.

Home Assistant 2026.7에서 energieleser를 쓴다면 전력계부터 외부에 열지 말고, NAS Docker 컨테이너가 내부망 HTTP 주소에 접근되는지만 확인하면 된다. 2026년 7월 1일 공식 릴리스 노트 기준 energieleser는 stromleser, gasleser, wasserleser, waermeleser 같은 장비의 실시간 계측값을 로컬 HTTP API로 읽는 새 통합이다. 내가 확인한 기준으론 이 장점은 단순하다. 전력 데이터를 받으려고 포트포워딩이나 클라우드 계정부터 만질 필요가 없다.

환경은 Synology DSM 7.3, Container Manager, Home Assistant Core 2026.7.1 Docker, 내부망 `192.168.0.0/24`, 전력계 예시 주소 `192.168.0.45`로 잡았다. 처음엔 HA 화면에서 통합 검색만 하면 끝날 줄 알았는데, NAS 방화벽과 컨테이너 네트워크가 갈라져 있으면 장비 검색이 안 된다. 특히 `bridge` 네트워크에 둔 컨테이너는 같은 공유기 아래 장비라도 mDNS나 자동 검색이 애매할 때가 있다.

| 확인 지점 | 권장값 | 이유 |
|---|---:|---|
| 전력계 IP | 공유기 DHCP 예약 | 센서 주소가 바뀌면 HA 통합이 끊긴다 |
| HA 포트 | 내부망 `8123`만 허용 | 에너지 데이터 때문에 외부 공개할 이유가 없다 |
| Docker 네트워크 | 가능하면 host 또는 macvlan 검토 | 로컬 장비 자동 검색 실패를 줄인다 |
| 백업 범위 | `/config` 포함 | 통합 등록 정보와 에너지 통계가 같이 남는다 |

NAS에서 전력계가 보이는지 터미널로 확인한다. DSM 터미널이나 SSH에서 장비 주소만 바꿔 실행한다.

```bash
curl -I http://192.168.0.45
```

응답이 없으면 Home Assistant 문제가 아니다. 공유기에서 전력계 IP를 고정하고, NAS 방화벽에서 같은 대역 접속을 허용한다. 여기서 삽질했던 건 스마트폰 앱에서는 전력계가 보이는데 NAS에서는 안 보이는 경우였다. 앱은 같은 Wi-Fi에 붙어 있었고, NAS는 별도 VLAN에 있었다. 이 경우엔 VLAN 간 HTTP 허용 규칙을 하나 열어야 한다.

Home Assistant에서는 `Settings -> Devices & services -> Add integration`에서 `energieleser`를 검색해 등록한다. 자동 검색이 뜨면 그대로 잡고, 안 뜨면 IP를 직접 넣는다. 등록 뒤에는 센서 이름을 바로 정리한다. `sensor.stromleser_power`처럼 장비명이 섞인 이름은 나중에 자동화에서 읽기 어렵다. 나는 `sensor.home_power_now`, `sensor.home_energy_total`처럼 순간전력(W)과 누적에너지(kWh)를 구분해 별칭을 붙인다.

에너지 대시보드에 넣을 때는 순간전력 센서가 아니라 누적 kWh 센서를 고른다. 이걸 잘못 고르면 그래프가 비거나 지원되지 않는 센서로 나온다.

```yaml
homeassistant:
  customize:
    sensor.home_energy_total:
      device_class: energy
      state_class: total_increasing
      unit_of_measurement: kWh
```

위 설정은 센서 메타데이터가 자동으로 맞지 않을 때만 쓴다. 통합이 이미 올바른 `device_class`와 `state_class`를 주면 건드리지 않는 편이 낫다. 수정한 뒤에는 개발자 도구에서 상태 단위가 `kWh`인지 보고, 하루 정도 기록이 쌓인 뒤 에너지 화면을 확인한다.

주의할 점은 세 가지다. 전력계 HTTP API를 인터넷에 공개하지 않는다. NAS Docker의 `/config`를 백업하지 않으면 에너지 통계와 통합 설정을 같이 잃을 수 있다. 또 2026.7의 새 통합이라고 해서 모든 energieleser 계열 장비가 같은 필드명을 쓰는 건 아니다. 값이 0으로 고정되면 통합 삭제보다 장비 펌웨어, IP, 단위부터 본다.

짧게 정리하면 이렇다. Home Assistant 2026.7의 energieleser는 NAS Docker와 잘 맞지만, 핵심은 통합 추가 버튼이 아니라 내부망 도달성이다. 전력계 IP 고정, NAS에서 `curl` 확인, 누적 kWh 센서 선택, `/config` 백업까지 끝내면 에너지 모니터링은 꽤 안정적으로 굴러간다.

확인 출처: [Home Assistant 2026.7 공식 릴리스 노트](https://www.home-assistant.io/blog/2026/07/01/release-20267/)

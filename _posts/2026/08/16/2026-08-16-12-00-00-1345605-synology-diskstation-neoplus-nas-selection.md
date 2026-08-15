---
layout: post
title: "Synology DiskStation neo+ 2026 신제품 비교 - DS725neo+·DS925neo+ 홈서버 설정 기준"
description: "2026년 8월 공개된 Synology DiskStation neo+ 시리즈의 모델별 차이와 DSM 초기 설정, 메모리·네트워크 선택 기준을 홈서버 관점에서 정리한다."
date: 2026-08-16
tags: [Synology, NAS설정, 홈서버, HomeLab, DSM]
comments: true
share: true
---

![Synology DiskStation neo+ 홈서버 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)

이 그림에서 볼 것은 NAS 본체보다 저장장치와 네트워크가 함께 홈서버 구성을 결정한다는 점이다.

Synology가 2026년 8월 DiskStation neo+ 시리즈를 추가했다. DS725neo+, DS925neo+, DS1525neo+, DS1825neo+ 네 모델이며, 기존 Plus 계열과 같은 플랫폼을 쓰면서 기본 메모리와 가격 구성을 낮춘 라인업이다. Synology 공식 제품 목록에도 네 모델이 새 제품으로 등록돼 있다. ([공식 제품 목록](https://www.synology.com/en-me/products?product_line=ds_plus))

처음엔 기본 메모리가 적으면 손해라고 생각했는데, Docker와 Synology Drive를 같이 쓸 계획이라면 오히려 메모리 확장 가능 여부를 먼저 보는 편이 맞다. 현재 공개 자료 기준 네 모델은 4GB로 시작하고 최대 32GB까지 확장할 수 있다. ([신제품 소개](https://www.techradar.com/pro/synology-reveals-a-host-of-new-nas-devices-it-says-can-balance-budget-and-power-requirements-for-all-users))

## 모델 선택은 베이 수보다 확장 계획이 기준이다

| 모델 | 드라이브 베이 | 기본 네트워크 | 이런 경우에 맞다 |
|---|---:|---|---|
| DS725neo+ | 2베이 | 2.5GbE 1개 + 1GbE 1개 | 사진 백업, 가벼운 Docker |
| DS925neo+ | 4베이 | 2.5GbE 2개 | Jellyfin·Nextcloud를 함께 운영 |
| DS1525neo+ | 5베이 | 2.5GbE 2개, 10GbE 확장 | VM과 대용량 파일을 함께 사용 |
| DS1825neo+ | 8베이 | 2.5GbE 2개, 10/25GbE 확장 | RAIDZ가 아닌 SHR 확장형 저장소를 오래 운영 |

DS725neo+는 2베이만 보면 저렴해 보이지만, Docker 컨테이너를 많이 올리면 저장공간보다 메모리가 먼저 부족해진다. 반대로 DS925neo+는 4베이와 M.2 NVMe 슬롯이 있어 사진 원본, 미디어, 백업을 나누기 쉽다. 네트워크가 1GbE 공유기라면 10GbE 확장 모델을 사도 당장 속도가 빨라지지 않으니 공유기와 스위치 교체 비용까지 같이 계산해야 한다.

## 구매 직후 DSM 설정 순서

환경은 DSM 7.4 계열, NAS 4베이, 공유기 2.5GbE, 하드디스크 4개 구성으로 잡았다. 디스크를 장착한 뒤 아래 순서로 진행한다.

1. `find.synology.com`에서 NAS를 찾고 DSM 설치를 완료한다.
2. 관리자 계정 이름을 `admin`이 아닌 별도 이름으로 만든다.
3. **제어판 → 보안 → 2단계 인증**에서 인증 앱을 등록한다.
4. **저장소 관리자 → 저장소 풀**에서 SHR-2가 필요한지 결정한다.
5. **제어판 → 네트워크 → 네트워크 인터페이스**에서 2.5GbE 링크 속도를 확인한다.
6. **제어판 → 업데이트 및 복원**에서 중요한 DSM 업데이트 자동 설치만 켠다.

초기 볼륨은 무조건 최대 용량으로 만들지 않았다. 사진과 문서가 들어갈 공유 폴더는 Btrfs로 만들고, 스냅샷을 켤 공간을 15~20% 남겼다. 스냅샷은 백업이 아니라 같은 NAS 안에서 빠르게 되돌리는 기능이라 USB나 다른 NAS 백업을 대신할 수 없다.

## 메모리와 Docker를 같이 쓸 때 확인할 것

Synology Drive, Container Manager, Jellyfin을 동시에 사용할 계획이면 4GB로 시작해도 되지만 컨테이너 5개 이상부터는 메모리 사용량을 확인하는 게 낫다. **작업 관리자 → 성능 → 메모리**에서 스왑 사용량이 반복해서 0이 아니면 메모리 확장을 검토한다.

컨테이너 데이터는 시스템 볼륨과 분리한 공유 폴더에 둔다. 예를 들어 `docker/appdata`, `docker/media`를 나누면 앱을 지울 때 설정까지 함께 삭제하는 실수를 줄일 수 있다. 외부 공개가 필요할 때는 DSM 관리자 포트를 바로 포워딩하지 말고 Tailscale VPN이나 역방향 프록시를 먼저 검토한다.

## 구매 전 체크리스트

- [ ] 2년 안에 필요한 베이 수를 계산했는가
- [ ] 4GB 이후 메모리 확장 비용을 포함했는가
- [ ] 공유기·스위치가 2.5GbE를 지원하는가
- [ ] M.2 SSD를 캐시로 쓸지 별도 저장소로 쓸지 정했는가
- [ ] NAS 밖에 보관할 백업 장치를 준비했는가

neo+는 기존 Plus를 대체하는 제품이 아니라 선택지를 늘린 모델이다. 지금 당장 파일 공유만 한다면 DS725neo+도 충분하지만, 자체 호스팅과 백업을 같이 운영할 계획이면 DS925neo+부터 보는 편이 덜 후회한다. 기본 메모리보다 베이 수, 네트워크 장비, 백업 비용을 합친 총액이 실제 구매 기준이다.

참고 자료: [Synology 공식 제품 목록](https://www.synology.com/en-me/products?product_line=ds_plus), [DiskStation neo+ 시리즈 소개](https://www.techradar.com/pro/synology-reveals-a-host-of-new-nas-devices-it-says-can-balance-budget-and-power-requirements-for-all-users)

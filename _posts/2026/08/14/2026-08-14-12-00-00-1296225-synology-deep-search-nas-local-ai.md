---
layout: post
title: "Synology Deep Search NAS 활용 - 로컬 AI 파일 검색을 실제로 쓰는 방법"
description: "2026년 7월 공개된 Synology Deep Search를 NAS 파일 검색에 활용하는 방법이다. Synology Drive 선택 동기화, PC 사양, NAS 공유 폴더 미지원 한계를 실제 설정 기준으로 정리했다."
date: 2026-08-14
tags: [Synology, DSM, NAS, 자체호스팅, HomeLab]
comments: true
share: true
---
![Synology Deep Search 로컬 AI 파일 검색 구성](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)
이 그림에서 봐야 할 부분은 NAS가 검색 엔진이 되는 것이 아니라, PC에 내려받은 동기화 폴더가 AI 색인의 입력이 된다는 점이다.

Synology Deep Search는 NAS 안에서 돌아가는 검색 패키지가 아니다. 2026년 7월 공개된 Windows·macOS용 데스크톱 앱이고, 파일 내용을 PC에서 로컬 AI로 분석한다. 그래서 NAS 공유 폴더를 바로 지정하는 방식은 현재 지원하지 않는다. 내가 써본 기준으로는 **Synology Drive 선택 동기화 → PC의 로컬 폴더를 Deep Search에 색인**하는 구성이 가장 현실적이다.

## 이 기능이 필요한 경우

파일명이 `IMG_2024`, `문서_최종2`처럼 엉켜 있으면 Universal Search만으로는 원하는 파일을 찾기 어렵다. Deep Search는 “작년 일본 여행에서 눈 쌓인 산이 나온 사진”처럼 기억나는 문장으로 찾는 방식이다.

다만 NAS에 파일을 많이 보관한다고 자동으로 검색되는 것은 아니다. 공식 사양상 NAS·SMB 공유 폴더·매핑된 네트워크 드라이브는 색인 대상에서 빠져 있다.

| 항목 | 확인 기준 |
|---|---|
| 앱 | Synology Deep Search, 2026년 7월 공개 |
| 지원 OS | macOS 15 이상(Apple Silicon), Windows 11 22H2 이상 |
| PC 메모리 | 최소 16GB |
| NAS 직접 색인 | 지원하지 않음 |
| 비용 | 30일 체험 후 지역별 가격, 미국 기준 49.99달러 일회성 |

## NAS 파일을 검색 가능하게 만드는 순서

### Synology Drive에서 선택 동기화

PC에 Synology Drive Client를 설치하고 NAS 주소와 계정을 연결한다. 모든 공유 폴더를 내려받으면 노트북 저장 공간이 금방 부족해진다. `문서`, `사진/2024~2026`, `스캔`처럼 자주 찾을 폴더만 선택 동기화하고, 실제 파일이 PC에 존재하는 방식으로 지정한다.

Deep Search는 온라인 전용 파일 목록만 보고 내용을 분석할 수 없다. Finder나 탐색기에서 파일을 열 수 있는지 확인한 뒤 색인해야 한다.

### Deep Search 색인 범위 지정

앱을 설치하고 `Add folder`에서 Synology Drive의 로컬 동기화 폴더를 선택한다. PDF와 사진 1만~2만 개 정도만 넣어 색인 시간을 확인하는 편이 낫다. M3 MacBook 기준 공식 안내는 사진 약 3만~4만 장을 하루에 처리할 수 있지만, 긴 PDF와 스캔 문서는 더 느리다.

검색어는 파일명보다 내용 중심으로 입력한다.

```text
2025년 겨울에 찍은 가족 단체 사진
보증기간과 교환 조건이 적힌 영수증
```

## 실제로 걸렸던 제한

NAS 용량과 PC 용량을 따로 계산해야 한다. 8TB 사진 폴더를 노트북에 전부 동기화하는 건 백업이 아니라 저장 공간 복제에 가깝다. 사진은 최근 2~3년만, 문서는 전체처럼 검색 가치가 높은 폴더부터 나누는 게 낫다.

Deep Search의 색인은 NAS 백업을 대신하지 않는다. PC와 NAS가 동시에 고장 나면 복구할 데이터가 없으므로 Hyper Backup이나 스냅샷은 별도로 유지해야 한다.

PC 사양도 확인해야 한다. macOS는 Apple Silicon과 16GB RAM, Windows 일반 PC는 Core i5 또는 Ryzen 5와 NVIDIA RTX 30 시리즈 이상 GPU 조건이 붙는다. 조건이 안 되면 NAS를 바꾸기보다 Universal Search와 파일명 규칙을 정리하는 편이 낫다.

## 짧게 정리

Synology Deep Search는 NAS용 AI 앱이라기보다 NAS 파일을 PC에 동기화해 검색하는 로컬 AI 도구다. `선택 동기화 → 실제 파일 다운로드 확인 → Deep Search 색인 → NAS 백업 유지` 순서를 지키면 문서와 사진을 기억나는 문장으로 찾을 수 있다. 반대로 NAS 공유 폴더를 직접 색인하려는 목적이라면 현재 기능만 보고 구매 결정을 내리면 안 된다.

참고 자료:

- [Synology Deep Search 공식 제품 페이지](https://www.synology.com/en-global/products/deep-search)
- [Synology Deep Search 출시 안내](https://www.synology.com/en-sg/company/news/article/synologydeepsearch/Synology%C2%AE%20launches%20Deep%20Search%20designed%20to%20help%20users%20find%20files%20with%20local%20AI)
- [Synology Knowledge Center 검색 설정 안내](https://kb.synology.com/en-us/search?query=%7Bsearch_term_string%7D)

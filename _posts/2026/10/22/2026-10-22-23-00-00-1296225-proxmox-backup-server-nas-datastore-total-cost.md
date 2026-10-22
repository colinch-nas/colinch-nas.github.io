---
layout: post
title: "Proxmox Backup Server 4.2 NAS 저장소 - NFS 공유와 전용 디스크의 호환성·총비용"
description: "Proxmox Backup Server 4.2를 NAS NFS 공유에 둘지 전용 디스크에 둘지, datastore 호환성·복원 위험·4베이 기준 총비용으로 판단하는 방법을 정리한다."
date: 2026-10-22
tags: [Proxmox, ProxmoxBackupServer, NAS, TrueNAS, QNAP, HomeLab, 백업]
comments: true
share: true
---

![Proxmox Backup Server NAS 저장소와 전용 디스크 비교](https://images.unsplash.com/photo-1558494949-ef010cbdcc31?w=1200&q=80)
그림에서 볼 부분은 NAS 공유 폴더와 PBS 전용 디스크가 같은 “백업 저장소”처럼 보여도 장애 범위와 복원 절차가 다르다는 점이다.

Proxmox Backup Server(PBS) 4.2를 처음 구성한다면 1~2대의 Proxmox 노드에는 로컬 전용 디스크가, 이미 4베이 이상 NAS를 운영 중이고 백업 대상을 한곳에 모아야 한다면 NAS의 NFS 공유가 유리하다. 다만 NAS 한 대에 PBS와 원본 VM을 같이 두면 NAS 장애 때 원본과 백업이 동시에 멈춘다. 이 글은 2026년 10월 22일 공식 문서를 확인한 구매 판단이며, 실제 장비를 연결해 측정한 결과는 아니다.

## 로컬 디스크와 NAS 공유 비교

PBS 공식 문서에서 datastore는 일반 Unix 파일시스템의 디렉터리이며 ext4, xfs, zfs를 사용한다. 따라서 PBS가 설치된 호스트에 디스크를 직접 붙이는 구성이 기본에 가깝다. 반면 QNAP 공식 안내는 Proxmox VE가 NAS의 SMB/NFS 공유를 백업 대상으로 등록하는 절차를 설명한다. NAS 경로를 PBS datastore로 쓸 때는 네트워크 단절, NFS 권한, 재부팅 후 마운트 순서를 별도로 검증해야 한다.

| 구성 | 초기 구매 | 장애 범위 | 어울리는 규모 | 구매하지 않아도 되는 조건 |
|---|---:|---|---|---|
| PBS 전용 HDD 2개 미러 | 디스크 2개 | PBS 호스트 장애에 취약 | 노드 1~2대, 백업 4TB 이하 | 이미 NAS에 별도 여유 풀과 외부 복제가 있을 때 |
| NAS NFS datastore | 추가 디스크가 적음 | NAS·네트워크 장애에 취약 | 노드 2~4대, 중앙 백업 | 단순 VM 1대이고 원본과 백업을 같은 NAS에 둘 때 |
| PBS 로컬 + NAS 동기화 | 디스크와 NAS 공간 모두 필요 | 한쪽 장애를 분리 | 복원 시간을 줄여야 하는 홈랩 | 재생성 가능한 테스트 VM만 있을 때 |

여기서 핵심은 NFS가 빠르냐가 아니다. PBS는 청크를 중복 제거해 저장하므로 같은 백업을 파일 복사처럼 단순 계산할 수 없다. 공식 문서 기준 datastore 하나는 동시에 여러 PBS 인스턴스가 운영할 수 없고, 장애 후 새 설치에서 기존 내용을 재사용할 때도 같은 datastore 이름과 재연결 절차가 필요하다. NAS 스냅샷만 찍어 두고 PBS 복원이라고 부르면 안 되는 이유다.

## 4베이 NAS 기준으로 총비용 계산하기

가상 사례로 4베이 NAS에 12TB HDD 4개 RAIDZ2 또는 RAID 6급 구성을 쓰고, Proxmox 노드 2대의 백업 원본이 5TB라고 하자. NAS에 PBS datastore를 두면 추가 장비는 거의 없지만, NAS 풀 사용량과 NFS 성능을 계속 공유한다. PBS 전용 디스크 2개를 노드에 추가하면 디스크 비용과 전력은 늘지만 NAS가 꺼져도 백업 서버를 별도로 복구할 선택지가 생긴다.

| 계산 항목 | NAS NFS | PBS 전용 디스크 |
|---|---|---|
| 추가 하드웨어 | 보통 0원 | HDD 2개 또는 SSD 2개 |
| 운영 비용 | NAS 전력·네트워크 사용 | 디스크 전력·호스트 베이 사용 |
| 복원 병목 | NFS·스위치·NAS 풀 | PBS 디스크와 호스트 I/O |
| 별도 재해 대응 | 다른 NAS·클라우드 필요 | NAS 또는 오프사이트 복제 필요 |

백업 파일이 5TB라고 해서 5TB 디스크 하나만 사면 충분하지 않다. 보존 기간, 변경량, PBS 중복 제거율, 검증 작업의 임시 여유 공간을 포함해야 한다. 실제 사용량을 모르면 `원본 데이터 × 보존 배수 + 20% 여유`를 보수적인 시작값으로 잡고, 구매 전 PBS의 datastore usage와 GC 결과로 다시 계산한다. 가격은 판매처와 시점에 따라 달라지므로 특정 금액을 고정하지 않았다.

## 설정 뒤 반드시 확인할 체크리스트

- [ ] PBS 4.2 datastore가 ext4·xfs·zfs의 정상 경로에 만들어졌는가
- [ ] NAS NFS export에 PBS 전용 계정과 최소 권한만 부여했는가
- [ ] NAS 재부팅 뒤 PBS가 마운트를 잃었을 때 백업 작업이 실패로 기록되는가
- [ ] `Datastore → Verify Jobs`를 주기적으로 실행하고 실패 알림을 받는가
- [ ] prune 뒤 즉시 공간이 줄지 않을 수 있으므로 GC 순서를 알고 있는가
- [ ] VM 하나를 다른 저장소에 실제 복원해 부팅·네트워크·애플리케이션을 확인했는가
- [ ] NAS와 PBS 중 한쪽을 잃어도 남는 복사본이 있는가

PBS 공식 문서는 verify가 청크의 체크섬을 다시 읽어 무결성을 확인한다고 설명한다. 백업 성공 표시만으로는 복원 가능성을 증명하지 못한다. 1~2대 홈랩이고 VM을 다시 만들 수 있다면 기존 NAS NFS를 써도 된다. 사진·문서·업무 VM처럼 복구 시간이 비용으로 이어지면 PBS 전용 디스크와 NAS 동기화를 함께 예산에 넣는 편이 낫다. 반대로 원본과 백업을 같은 NAS에 몰아넣고 디스크를 더 사지 않는 선택은 NAS 장애를 복구하지 못하는 조건부 절약이다.

참고한 문서는 [Proxmox Backup Server 4.2 datastore 문서](https://pbs.proxmox.com/docs/storage.html), [Proxmox Backup Server 공식 개요](https://pbs.proxmox.com/docs/), [QNAP의 Proxmox VE용 SMB/NFS 백업 안내](https://www.qnap.com/en-us/how-to/tutorial/article/how-to-set-up-proxmox-ve-backup-space-on-a-qnap-nas)다.

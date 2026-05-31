# W5. Database — RDS · Aurora · ElastiCache

> AWS SAA 스터디 정리 (이론 + 기출 유형 문제)
> 대상 서비스: RDS, Aurora, ElastiCache
> 작성 기준: 2026년 최신 AWS 사양 반영 (PDF 원본 + 업데이트 사항 보강)

---

## 목차

- [1. Amazon RDS](#1-amazon-rds)
  - [1.1 RDS란?](#11-rds란)
  - [1.2 DB 인스턴스 & 스토리지](#12-db-인스턴스--스토리지)
  - [1.3 Multi-AZ (고가용성)](#13-multi-az-고가용성)
  - [1.4 Read Replica (읽기 확장)](#14-read-replica-읽기-확장)
  - [1.5 백업: Automated Backup vs Snapshot](#15-백업-automated-backup-vs-snapshot)
  - [1.6 RDS Proxy](#16-rds-proxy)
  - [1.7 Enhanced Monitoring](#17-enhanced-monitoring)
  - [1.8 RDS vs EC2 자체 설치 DB](#18-rds-vs-ec2-자체-설치-db)
- [2. Amazon Aurora](#2-amazon-aurora)
  - [2.1 아키텍처](#21-아키텍처)
  - [2.2 클러스터 & 엔드포인트](#22-클러스터--엔드포인트)
  - [2.3 Global Database](#23-global-database)
  - [2.4 Serverless & I/O-Optimized](#24-serverless--io-optimized)
- [3. ElastiCache](#3-elasticache)
  - [3.1 캐시 & In-Memory DB](#31-캐시--in-memory-db)
  - [3.2 ElastiCache 개요 & 용도](#32-elasticache-개요--용도)
  - [3.3 엔진 비교: Memcached vs Redis OSS vs Valkey](#33-엔진-비교-memcached-vs-redis-oss-vs-valkey)
  - [3.4 캐싱 전략](#34-캐싱-전략)
- [4. 핵심 비교 요약표](#4-핵심-비교-요약표)
- [5. SAA 기출 유형 문제](#5-saa-기출-유형-문제)
- [6. 시험 직전 한 줄 정리](#6-시험-직전-한-줄-정리)

---

## 1. Amazon RDS

### 1.1 RDS란?

관계형 데이터베이스를 AWS에서 운영할 수 있게 해주는 **관리형(Managed) 서비스**다.

- 지원 엔진: **MySQL, MariaDB, PostgreSQL, Oracle, MS SQL Server, Aurora**
- AWS가 백업 / 소프트웨어 패치 / 장애 감지 및 복구를 대신 수행
- **DB 인스턴스에 SSH 접근 불가, OS 제어 불가** (관리형이기 때문)
- 스토리지 용량 **Auto Scaling** 지원

> 💡 **시험 포인트**: "OS 레벨 접근이 필요하다 / DB 엔진을 직접 커스터마이징해야 한다"는 요구사항이 나오면 RDS가 아니라 **EC2 자체 설치 DB**가 정답이다.

### 1.2 DB 인스턴스 & 스토리지

DB 인스턴스는 클라우드에서 실행되는 격리된 DB 환경이며, EC2처럼 인스턴스 클래스(`db.m5`, `db.r5` 등)를 가진다. 스토리지는 **EBS** 기반이다.

| 스토리지 유형 | 특징 | 적합한 워크로드 |
|---|---|---|
| 범용 SSD (gp2/gp3) | 무난한 가격/성능 | 대부분의 일반 워크로드 |
| 프로비저닝 IOPS (io1/io2) | 빠르고 **일관적인 낮은 지연** | I/O 집약적, 낮은 지연이 중요한 OLTP |
| 마그네틱 | 저렴, 느림 (레거시) | 접속 빈도가 낮은 워크로드 |

> 💡 **시험 포인트**: "일관적으로 낮은 지연 시간(consistent low latency) + 높은 IOPS"가 요구되면 → **Provisioned IOPS**.

### 1.3 Multi-AZ (고가용성)

Multi-AZ는 **가용성(HA)을 위한 기능**이지 읽기 확장(scaling) 기능이 아니다. 이 구분이 SAA의 단골 함정이다.

현재 RDS Multi-AZ에는 **두 가지 배포 모드**가 있다 (PDF 원본은 ① 기준).

#### ① Multi-AZ DB 인스턴스 (전통적 방식)

- Primary 1개 + **Standby 1개**를 다른 AZ에 배치
- Primary → Standby로 **동기식(synchronous) 복제**
- **Standby는 읽기/쓰기 불가** (단순 대기 복제본)
- 하나의 DNS 엔드포인트 공유 → 장애 시 자동 페일오버 (보통 **60~120초**)
- 모든 RDS 엔진 지원

#### ② Multi-AZ DB 클러스터 (신규 방식)

- Writer 1개 + **읽기 가능한 Reader 2개**를 **3개 AZ**에 배치
- **반동기식(semi-synchronous) 복제**
- **Reader 2개가 읽기 트래픽 처리 가능** + 페일오버 대상
- 페일오버 일반적으로 **35초 이내**, write latency 더 낮음
- **MySQL, PostgreSQL만 지원**

**페일오버가 발생하는 상황**: AZ 중단, Primary 장애, 인스턴스 타입 변경, OS 패치, 수동 Failover 재부팅.

> 💡 **시험 포인트**
> - "고가용성만 필요" → Multi-AZ DB 인스턴스
> - "고가용성 + 읽기 트래픽도 분산 + MySQL/PostgreSQL" → Multi-AZ DB **클러스터**
> - Standby 복제본을 읽기 용도로 쓰려는 보기는 **DB 인스턴스 모드에서는 오답**

### 1.4 Read Replica (읽기 확장)

읽기 부하를 분산하기 위한 **읽기 전용 복제본**. Multi-AZ와 목적이 정반대다.

- Primary로 들어오는 읽기 쿼리를 복제본으로 우회 → Primary 부하 감소
- **비동기식(asynchronous) 복제** → 복제 지연(replication lag) 발생 가능
- 인스턴스당 **최대 15개** (교차 리전 포함)
- **독립 인스턴스로 승격(promote)** 가능 → DR 용도로도 활용
- 교차 리전(Cross-Region) 복제본 → 글로벌 읽기 지연 감소 + DR
- MySQL, MariaDB, PostgreSQL, Oracle 지원

| 구분 | Multi-AZ | Read Replica |
|---|---|---|
| 목적 | **고가용성(HA)** | **읽기 성능 확장** |
| 복제 방식 | 동기식(반동기식) | **비동기식** |
| 복제본 읽기 | DB 인스턴스 모드는 불가 | **가능** |
| 페일오버 | 자동 | 수동 승격 필요 |
| 리전 | 동일 리전 | 동일/교차 리전 |

> 💡 **시험 포인트**: "읽기 쿼리가 너무 많아 Primary가 느려진다" → **Read Replica**. "Primary 장애에도 서비스 중단 없어야 한다" → **Multi-AZ**.

### 1.5 백업: Automated Backup vs Snapshot

| 구분 | Automated Backup | Snapshot |
|---|---|---|
| 대상 | DB 인스턴스 전체 | DB 인스턴스 전체 |
| 주기 | 매일 자동 | 자동/수동 |
| 보존 | 1~35일 (0=비활성화) | **만료 없음** (직접 삭제 전까지 유지) |
| 복원 시점 | **특정 시점(PITR)**, 최근 5분 전까지 | **스냅샷 생성 시점만** |
| 복원 결과 | 새 DB 인스턴스 생성 | 새 DB 인스턴스 생성 |
| 공유/복사 | 제한적 | **복사·공유·마이그레이션 가능** |

- 기본 보존 기간: **CLI 생성 시 1일 / 콘솔 생성 시 7일**
- 단일 AZ에서는 백업/스냅샷 중 스토리지 I/O가 잠시 중단될 수 있으나, **Multi-AZ(MariaDB/MySQL/Oracle/PostgreSQL)에서는 Primary AZ I/O 중단 없음**

> 💡 **시험 포인트**: "장기 보관 / 다른 계정·리전으로 공유" → **Snapshot**. "임의 시점으로 복구(point-in-time)" → **Automated Backup**.

### 1.6 RDS Proxy

애플리케이션과 RDS 사이에 두는 **완전관리형 커넥션 풀(connection pool)**.

- 애플리케이션 ↔ Proxy ↔ RDS 구조로 직접 연결을 중재
- **열린 연결 수를 줄여 DB 리소스 부하 감소**
- Serverless·Multi-AZ·Auto Scaling 지원
- 페일오버 시 **페일오버 시간을 최대 66%까지 단축**
- 지원: MySQL, PostgreSQL, MariaDB, Aurora
- **IAM 인증 강제 가능**, 퍼블릭 액세스 불가 (VPC 내부)
- **Lambda와의 조합이 베스트** — Lambda는 빠르게 생성·소멸되며 수많은 커넥션을 만들어 DB 커넥션을 고갈시키기 쉬운데, Proxy가 이를 풀링으로 흡수

> 💡 **시험 포인트**: "Lambda가 RDS 연결을 너무 많이 만들어 'too many connections' 오류 발생" → **RDS Proxy**.

### 1.7 Enhanced Monitoring

- RDS 지표를 **실시간(최대 1초 단위)**으로 수집하는 강화 모니터링
- 지표는 **CloudWatch Logs에 30일 저장**
- 일반 모니터링은 **하이퍼바이저**에서 수집, Enhanced Monitoring은 **인스턴스 내부 에이전트(OS 레벨)**에서 수집 → CPU/메모리/프로세스별 상세 지표 확인 가능

### 1.8 RDS vs EC2 자체 설치 DB

| 구분 | RDS | EC2 자체 설치 |
|---|---|---|
| 관리 | AWS가 백업·패치 담당 | **직접 관리** |
| OS/SSH 접근 | 불가 | **가능** |
| 커스터마이징 | 제한적 | **자유로움** |

---

## 2. Amazon Aurora

### 2.1 아키텍처

"클라우드에서 DB를 처음부터 설계하면?"이라는 발상에서 출발한 **AWS 최적화 관계형 DB**.

- **MySQL / PostgreSQL 호환**
- 스토리지 자동 확장 (10GiB 단위 증가)
  - 오랫동안 SAA 정답은 **최대 128 TiB**였으나, 최근 **256 TiB**로 상향됨 (시험에서는 보기 맥락으로 판단)
- **3개 AZ에 걸쳐 데이터 6개 사본(2개 × 3 AZ)** 자동 복제
  - 읽기는 6개 중 3개, 쓰기는 6개 중 4개 사본만 있어도 동작 → 가용성·내구성 ↑
    ```
    이게 핵심인데, Aurora는 모든 사본이 응답해야 동작하는 게 아니라 정족수(쿼럼)만 충족되면 동작하도록 설계되어 있습니다.
    쓰기: 6개 중 4개가 성공하면 쓰기 완료로 간주 (Write quorum = 4)
    읽기: 6개 중 3개에서 읽으면 읽기 성공으로 간주 (Read quorum = 3)
    ```
  - 6개 사본이 있어도 **과금은 리전당 논리적 1개 사본 기준**
  ```
  물리적으로는 6벌이 디스크에 깔려 있지만, AWS가 청구하는 스토리지 비용은 6배가 아니라 실제 데이터 크기(1벌) 기준이라는 뜻입니다. 100GB짜리 DB라면 600GB가 아니라 100GB만큼만 스토리지 요금을 냅니다. 6중 복제로 인한 중복 저장은 Aurora가 내부적으로 떠안고 사용자에게 전가하지 않는 구조!
  ```
- 자가 치유(self-healing) 스토리지 — 디스크 오류를 스스로 탐지·복구
- **컴퓨팅과 스토리지가 분리된** 구조 (인스턴스는 데이터를 직접 보관하지 않음)

### 2.2 클러스터 & 엔드포인트

기본 DB 인스턴스(Writer) + 읽기 복제본(Reader)을 묶어 **클러스터**로 구성.

- 읽기 복제본 **최대 15개**, 백업/스냅샷이 성능에 영향 없음
- **Aurora Auto Scaling**: 부하에 따라 읽기 복제본 수를 자동 조정 (MySQL/PostgreSQL 모두)
- Primary 장애 시 복제본으로 **자동 페일오버**
- 엔드포인트
  - **Writer(Cluster) Endpoint**: 항상 현재 Writer(Primary)를 가리킴 (쓰기)
  - **Reader Endpoint**: 모든 읽기 복제본에 자동 로드밸런싱 (읽기)
  - **Custom Endpoint**: 특정 복제본 그룹만 묶어 라우팅 (예: 분석용 인스턴스)

> 💡 **시험 포인트**: "읽기 부하를 자동으로 분산"하려면 애플리케이션을 **Reader Endpoint**에 연결한다. 개별 복제본 IP를 직접 코딩하는 보기는 오답.

### 2.3 Global Database

여러 리전에 Aurora 클러스터를 두는 글로벌 구성.

- **Primary 리전 1개**(읽기/쓰기) + **Secondary 리전 최대 5개**(읽기 전용)
- 리전 간 복제는 **스토리지 레벨**에서 이뤄져 일반적으로 1초 미만 지연
- **RPO 1초 / RTO 1분 미만**
- Primary 리전 장애 시 Secondary 리전을 승격(promote)하여 재해 복구

> 💡 **시험 포인트**: "여러 대륙의 사용자에게 낮은 읽기 지연 + 리전 단위 DR(낮은 RTO/RPO)" → **Aurora Global Database**. 단순 읽기 확장만이면 Read Replica로 충분.

### 2.4 Serverless & I/O-Optimized

- **Aurora Serverless**: 용량(ACU)을 워크로드에 맞춰 자동 조절. 가변적·예측 불가 워크로드, dev/test에 적합. 미사용 시 0에 가깝게 축소 가능
  - ACU 1개 ≈ 약 2GB 메모리 + 비례 CPU/네트워크
- **Aurora Standard vs I/O-Optimized**
  - Standard: 스토리지 + I/O 요청 **개별 과금**
  - I/O-Optimized: I/O 무과금, 스토리지 단가만 ↑ → **I/O 비용이 전체의 25% 초과 시 최대 40% 절감**

> 💡 **시험 포인트(비용 최적화)**: I/O 집약적이고 I/O 요금 비중이 큰 경우 → **Aurora I/O-Optimized**로 전환해 예측 가능한 비용 확보.

---

## 3. ElastiCache

### 3.1 캐시 & In-Memory DB

- **캐시**: 자주 쓰는 데이터를 빠른 임시 공간에 복사해두어 처리 속도를 높이는 것
- **In-Memory DB**: 모든 데이터를 메모리에 올려 운영하는 DBMS → 낮은 지연
  - 디스크 기반 DB(RDS)에 매번 접근하는 비효율을 줄임
  - 단점: **휘발성** — 전원 차단 시 데이터 유실, 할당된 메모리 한도까지만 저장

### 3.2 ElastiCache 개요 & 용도

AWS의 완전관리형 **In-Memory 캐시 서비스**. 두 가지 대표 용도:

1. **DB 부하 경감 (read-heavy 완화)**: 자주 조회되는 데이터를 캐시에 저장. 캐시 미스 시 RDS 조회 후 결과를 캐시에 적재
2. **세션 스토어**: 세션을 애플리케이션 외부(ElastiCache)에 저장 → 사용자가 다른 인스턴스로 라우팅돼도 **재로그인 불필요**

특징:
- **노드(Node)** 단위 구성, EC2처럼 타입별로 메모리 크기 선택 (작은 워크로드엔 작은 노드 → 비용 절감)
- AWS가 OS 유지보수·패치·모니터링·장애 복구·백업 수행
- 노드 기반 외에 **ElastiCache Serverless** 옵션도 제공(용량 자동 확장, 분 단위 시작)

> 💡 **시험 포인트(세션)**: "ELB 뒤 여러 EC2가 stateless해야 하고, 어느 인스턴스로 가든 세션 유지" → **ElastiCache에 세션 저장** (DynamoDB도 종종 정답 보기로 등장).

### 3.3 엔진 비교: Memcached vs Redis OSS vs Valkey

원본 PDF는 Memcached/Redis 2종 기준이지만, **2024년 10월부터 Valkey 엔진이 추가**되어 현재 ElastiCache는 **3개 엔진**을 지원한다. Valkey는 Redis가 라이선스를 변경한 뒤 Linux Foundation이 관리하는 오픈소스로, **Redis OSS의 드롭인 대체재**이며 ElastiCache에서 **Serverless 33% / 노드 기반 20% 더 저렴**하다.

| 항목 | Memcached | Redis OSS / Valkey |
|---|---|---|
| 데이터 구조 | 단순 Key-Value | String/List/Set/Hash/Sorted Set 등 풍부 |
| 멀티스레드 | **O** (스케일업에 유리) | 제한적 |
| 복제본 / 페일오버 | **불가** | **가능** (복제본을 Primary로 승격) |
| Multi-AZ | 미지원 | **지원** |
| 영속성(백업/스냅샷) | 미지원 | **지원** |
| 클러스터 구조 | 노드들로 구성, 샤딩(분배)만 | Shard(여러 Node) 구조, Cluster Mode 시 다중 Shard |
| 용도 | 단순·휘발성 캐시, 수평 확장 | HA·영속성·복잡한 자료구조·세션·리더보드 |

- **Memcached**: 클러스터 내 노드들이 캐시 역할. 노드를 늘려 용량 확장 가능하지만 **페일오버·복제 불가**.
- **Redis OSS / Valkey**: 기본은 단일 Shard지만 Cluster Mode 활성화 시 다중 Shard. Shard는 여러 Node로 구성되어 1개가 읽기/쓰기, 나머지는 복제본. **복제본 승격으로 페일오버 + Multi-AZ 지원**.

> 💡 **시험 포인트**
> - "고가용성 / 페일오버 / 영속성 / 정렬·집합 같은 자료구조 / Pub-Sub" → **Redis OSS 또는 Valkey**
> - "가장 단순한 객체 캐시 + 멀티스레드로 손쉬운 수평 확장, HA 불필요" → **Memcached**
> - 비용 최적화로 Redis를 대체할 신규 오픈소스 → **Valkey**

### 3.4 캐싱 전략

업무에서도 자주 쓰는 두 가지 핵심 전략 (SAA + 실무 공통).

**① Lazy Loading (Cache-Aside)**
- 읽기 요청 → 캐시 확인 → **미스면 DB 조회 후 캐시에 적재**
- 장점: 실제 요청된 데이터만 캐싱(메모리 효율)
- 단점: 최초 요청은 캐시 미스 페널티(3-trip), 데이터가 **오래되어(stale)** 있을 수 있음

```
# Lazy Loading 의사코드
val = cache.get(key)
if val is None:           # cache miss
    val = db.query(key)   # DB 조회
    cache.set(key, val)   # 캐시에 적재
return val
```

**② Write-Through**
- 쓰기 시 **DB와 캐시를 함께 갱신**
- 장점: 캐시가 항상 최신
- 단점: 쓰기마다 캐시 갱신(쓰기 페널티), 조회되지 않을 데이터까지 캐싱(메모리 낭비)

**③ TTL (만료 시간)**
- 두 전략 모두 stale 데이터 문제를 완화하기 위해 **TTL을 함께 설정**하는 것이 일반적

> 💡 **시험/실무 포인트**: "오래된 데이터가 잠깐 보여도 무방하고 메모리 효율이 중요" → **Lazy Loading + TTL**. "데이터 일관성이 매우 중요" → **Write-Through (+ TTL)**.

---

## 4. 핵심 비교 요약표

**HA vs 읽기 확장 (가장 헷갈리는 포인트)**

| 요구사항 | 정답 |
|---|---|
| Primary 장애에도 자동 복구 (HA) | RDS Multi-AZ |
| 읽기 쿼리 부하 분산 | Read Replica / Aurora Reader Endpoint |
| HA + 읽기 분산 (MySQL/PostgreSQL) | RDS Multi-AZ **DB 클러스터** |
| 교차 리전 읽기 지연 감소 + DR | Cross-Region Read Replica / Aurora Global DB |
| 리전 단위 DR, RTO 1분·RPO 1초 | **Aurora Global Database** |
| DB 읽기 부하 자체를 줄이기 | **ElastiCache** |
| Lambda의 DB 커넥션 폭증 | **RDS Proxy** |
| 가변 워크로드 비용 최적화 | **Aurora Serverless** |
| I/O 비용이 큰 워크로드 | **Aurora I/O-Optimized** |

---

## 5. SAA 기출 유형 문제

> 정답과 해설은 `▶ 정답 보기`를 펼쳐서 확인하세요.

**Q1.** 한 회사의 RDS for MySQL DB가 읽기 쿼리 폭증으로 응답이 느려지고 있다. 쓰기 부하는 낮다. 가장 적절한 해결책은?

- A) Multi-AZ 배포 활성화
- B) Read Replica 추가
- C) 인스턴스 클래스를 더 큰 것으로 변경
- D) Automated Backup 보존 기간 연장

<details><summary>▶ 정답 보기</summary>

**정답: B** — 읽기 부하 분산은 Read Replica의 역할. A(Multi-AZ)는 HA용이며 Standby는 읽기 트래픽을 받지 못한다(인스턴스 모드 기준).
</details>

---

**Q2.** AZ 장애가 발생해도 데이터베이스가 **자동으로** 다른 AZ에서 서비스를 이어가야 한다. 단, 읽기 확장은 필요 없다. 어떤 구성이 가장 적절한가?

- A) Read Replica를 다른 AZ에 배치
- B) Multi-AZ DB 인스턴스 배포
- C) 매일 스냅샷 생성 후 수동 복원
- D) ElastiCache 도입

<details><summary>▶ 정답 보기</summary>

**정답: B** — 자동 페일오버가 핵심. Read Replica는 수동 승격이 필요하고, 스냅샷 복원은 자동 HA가 아니다.
</details>

---

**Q3.** 글로벌 사용자를 대상으로 하는 서비스가 **여러 리전에서 낮은 읽기 지연**을 제공하고, 리전 장애 시 **RTO 1분 미만**의 재해 복구가 필요하다. 가장 적합한 것은?

- A) RDS Multi-AZ
- B) 동일 리전 Read Replica 15개
- C) Aurora Global Database
- D) ElastiCache Global Datastore

<details><summary>▶ 정답 보기</summary>

**정답: C** — 다중 리전 읽기 + 낮은 RPO/RTO DR은 Aurora Global Database의 전형적 시나리오. D는 캐시 계층이라 주(主) DB 요구를 충족하지 못한다.
</details>

---

**Q4.** ELB 뒤에 여러 EC2 인스턴스가 있고, 사용자가 어느 인스턴스로 라우팅되든 **로그인 세션이 유지**되어야 한다. 가장 적절한 방법은?

- A) 각 EC2 로컬 디스크에 세션 저장
- B) ElastiCache(또는 DynamoDB)에 세션 저장
- C) RDS에 매 요청마다 세션 기록
- D) ELB 스티키 세션만 사용

<details><summary>▶ 정답 보기</summary>

**정답: B** — 세션을 외부 In-Memory 스토어로 분리해 애플리케이션을 stateless하게 만든다. D(스티키 세션)는 인스턴스 장애 시 세션이 사라져 권장되지 않는다.
</details>

---

**Q5.** Lambda 함수가 트래픽 급증 시 RDS에 동시에 너무 많은 연결을 만들어 `too many connections` 오류가 발생한다. 코드 변경을 최소화하며 해결하려면?

- A) RDS 인스턴스 크기 증설
- B) RDS Proxy 도입
- C) Read Replica 추가
- D) Multi-AZ 활성화

<details><summary>▶ 정답 보기</summary>

**정답: B** — RDS Proxy가 커넥션 풀링으로 연결 수를 흡수한다. Lambda + RDS 조합의 정석 답.
</details>

---

**Q6.** 캐시 계층에서 **자동 페일오버, 데이터 영속성, Sorted Set 기반 리더보드**가 필요하다. 적절한 ElastiCache 엔진은?

- A) Memcached
- B) Redis OSS / Valkey
- C) DynamoDB DAX
- D) S3

<details><summary>▶ 정답 보기</summary>

**정답: B** — 복제·페일오버·영속성·풍부한 자료구조는 Redis OSS/Valkey의 특징. Memcached는 단순 K-V이며 복제·페일오버 불가.
</details>

---

**Q7.** 단순한 객체 캐시이며 HA가 필요 없고, **멀티스레드로 손쉽게 수평 확장**하고 싶다. 적절한 엔진은?

- A) Redis OSS
- B) Valkey
- C) Memcached
- D) Aurora Serverless

<details><summary>▶ 정답 보기</summary>

**정답: C** — 멀티스레드 + 단순 K-V + 노드 추가로 수평 확장은 Memcached의 강점.
</details>

---

**Q8.** DB 백업을 **다른 AWS 계정으로 공유**하고 **만료 없이 장기 보관**해야 한다. 가장 적절한 것은?

- A) Automated Backup 보존 35일
- B) 수동 DB Snapshot
- C) Enhanced Monitoring
- D) Read Replica 승격

<details><summary>▶ 정답 보기</summary>

**정답: B** — Snapshot은 만료가 없고 복사·공유·마이그레이션이 가능. Automated Backup은 최대 35일 제한이며 공유에 제약이 있다.
</details>

---

**Q9.** RDS for PostgreSQL에서 **고가용성**과 함께 **읽기 트래픽 분산**까지 단일 배포로 해결하고, 페일오버를 35초 내로 단축하고 싶다.

- A) Multi-AZ DB 인스턴스
- B) Multi-AZ DB 클러스터
- C) Single-AZ + Read Replica 2개
- D) ElastiCache

<details><summary>▶ 정답 보기</summary>

**정답: B** — Multi-AZ DB 클러스터는 Writer 1 + 읽기 가능한 Reader 2(3 AZ) 구성으로 HA와 읽기 분산을 동시에 제공하며 페일오버가 ~35초로 빠르다. MySQL/PostgreSQL만 지원.
</details>

---

**Q10.** 데이터베이스가 **일관적으로 낮은 지연 시간과 높은 IOPS**를 요구하는 I/O 집약 OLTP 워크로드다. 적절한 RDS 스토리지는?

- A) 마그네틱
- B) 범용 SSD(gp2)
- C) 프로비저닝 IOPS(io1/io2)
- D) S3

<details><summary>▶ 정답 보기</summary>

**정답: C** — 일관적이고 예측 가능한 낮은 지연 + 높은 IOPS는 Provisioned IOPS.
</details>

---

**Q11.** Aurora 클러스터에서 애플리케이션의 **읽기 쿼리를 모든 복제본에 자동 부하 분산**하려고 한다. 무엇에 연결해야 하는가?

- A) Writer Endpoint
- B) Reader Endpoint
- C) 각 복제본의 인스턴스 엔드포인트를 코드에 하드코딩
- D) Custom Endpoint 하나만

<details><summary>▶ 정답 보기</summary>

**정답: B** — Reader Endpoint가 모든 읽기 복제본에 자동 로드밸런싱한다.
</details>

---

**Q12.** Aurora를 쓰는데 **I/O 요청 비용이 전체 청구액의 30%**를 차지한다. 비용을 예측 가능하게 만들고 절감하려면?

- A) Aurora Standard 유지
- B) Aurora I/O-Optimized로 전환
- C) Read Replica 제거
- D) 스토리지를 마그네틱으로 변경

<details><summary>▶ 정답 보기</summary>

**정답: B** — I/O 비용 비중이 25%를 넘으면 I/O-Optimized가 보통 더 저렴하고 비용이 예측 가능해진다.
</details>

---

**Q13.** 예측이 어려운 가변적 워크로드(피크/유휴 차이 큼)에 비용 효율적인 Aurora 구성은?

- A) 가장 큰 프로비저닝 인스턴스 고정
- B) Aurora Serverless
- C) Multi-AZ DB 인스턴스 고정
- D) Read Replica 15개 상시 운영

<details><summary>▶ 정답 보기</summary>

**정답: B** — Aurora Serverless는 부하에 따라 용량(ACU)을 자동 조절하므로 가변 워크로드에 비용 효율적.
</details>

---

## 6. 시험 직전 한 줄 정리

- **Multi-AZ = 가용성(HA)**, **Read Replica = 읽기 확장** — 절대 헷갈리지 말 것
- Multi-AZ에는 **DB 인스턴스(1 Standby, 읽기 불가)** vs **DB 클러스터(2 Reader, 읽기 가능, MySQL/PostgreSQL)** 두 모드
- Read Replica = **비동기**, 최대 **15개**, 승격 가능, 교차 리전 가능
- **Snapshot** = 만료 없음·공유 가능 / **Automated Backup** = PITR(최근 5분 전까지)
- **RDS Proxy** = 커넥션 풀링, **Lambda**와 베스트
- Aurora = **6 사본 / 3 AZ**, 자동 복구, 복제본 15개, 최대 128 TiB(최신 256 TiB)
- Aurora 엔드포인트: **Writer(쓰기) / Reader(읽기 분산) / Custom(그룹)**
- **Aurora Global DB** = 다중 리전, RTO 1분·RPO 1초
- ElastiCache 엔진 = **Memcached(단순·멀티스레드·HA 불가)** vs **Redis OSS/Valkey(HA·영속성·자료구조)**, Valkey는 비용 절감형 신규 엔진
- 캐싱 전략: **Lazy Loading(메모리 효율, stale 위험)** vs **Write-Through(일관성, 메모리 낭비)** + **TTL**

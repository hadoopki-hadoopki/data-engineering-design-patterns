# 데이터 엔지니어링 디자인 패턴 - 데이터 저장 디자인 패턴

> 출처: Data Engineering Design Patterns (Bartosz Konieczny, O'Reilly 2025) Chapter 8 | 실무 데이터 엔지니어링 관점 정리

> **도식 기호 범례** — `⚠` 위험·함정(조심해야 할 동작) · `✗` 잘못된 결과(깨진 상태) · `✓` 올바른 결과 · `⇒` 그 결과 도출

---

## 목차

1. [파티셔닝 (Partitioning)](#1-파티셔닝-partitioning)
   - 패턴 #50: 수평 파티셔너 (Horizontal Partitioner)
   - 패턴 #51: 수직 파티셔너 (Vertical Partitioner)
2. [레코드 조직화 (Records Organization)](#2-레코드-조직화-records-organization)
   - 패턴 #52: 버킷 (Bucket)
   - 패턴 #53: 정렬기 (Sorter)
3. [요약](#3-요약)

> 본 문서는 챕터 8 의 앞 두 절 — **#50·#51(8.1 Partitioning), #52·#53(8.2 Records Organization)** — 을 다룸.
> ※ **#51 Vertical Partitioner** 는 챕터 7 의 **#41 Vertical Partitioner** 와 이름이 같음.
> **#41 은 보안 특화**(개인정보 삭제·잊힐 권리), **#51 은 스토리지 특화**(불변 속성 중복 저장 제거)로 목적이 다름.

---

## 책의 use case (챕터 도입)

> 빅데이터 환경에서 쿼리나 잡 결과를 **2분 넘게** 기다려 본 적이 있는가? 많은 사람이 "그렇다" 고 답할 것이고,
> 어떤 이는 **10분 넘게** 기다려 봤을 것. 이 **시간 요소** 가 데이터 엔지니어링 작업의 중요한 축.
> 쿼리·잡이 빨리 끝날수록 응답을 일찍 받고, 대개 **비용도 싸짐**.

시간 요소를 최적화하는 길은 두 가지.

- **① 컴퓨팅 자원 추가** — 조직적 절차 없이 상대적으로 빠르고 쉬움. 단 **사후 대응(retroactive)** 이고,
  사용자가 읽기 지연을 불평하기 시작한 **압박 상황에서** 하게 되는 조치.
- **② 사전 대응(preemptive)** — 이 챕터의 **데이터 저장 디자인 패턴** 으로 **현명한 데이터 조직** 을 짜는 것.
  실행 시간을 개선하고 피드백을 더 일찍 준다는 점이 핵심.

```
[챕터 8 데이터 저장 패턴 — 네 카테고리]
──────────────────────────────────────────────────────────────────────
 8.1 Partitioning            처리할 데이터 볼륨 자체를 줄임      #50 Horizontal / #51 Vertical Partitioner
 8.2 Records Organization    고카디널리티용 국소 최적화          #52 Bucket / #53 Sorter
 8.3 Read Performance        메타데이터·구체화·리스팅 회피       #54 Metadata Enhancer / #55 Dataset Materializer / #56 Manifest
 8.4 Data Representation     정합성 ↔ 실행 시간 트레이드오프     #57 Normalizer / #58 Denormalizer
──────────────────────────────────────────────────────────────────────
 흐름 — 먼저 나누고(8.1) → 저카디널리티로 안 되는 건 묶고·정렬하고(8.2)
        → 읽을 때 건드릴 것을 줄이고(8.3) → 표현 방식을 고름(8.4).
 ※ 본 문서는 8.1 · 8.2 (#50~#53) 를 다룸.
```

**파티셔닝의 한계가 이 챕터의 이야기 축** — 파티셔닝은 **저카디널리티 값**(어떤 속성의 서로 다른 값이 많지 않을 때)에서만 잘 동작.
**고카디널리티 값** 에는 **버킷팅·정렬** 같은 더 국소적인 최적화 전략이 필요하고, 그게 두 번째 패턴 계열(8.2).

또한 수평 파티셔닝은 챕터 4 의 **Fast Metadata Cleaner** 같은 **멱등성 패턴 구현을 가능하게 하는 토대** 이기도 함.

### 패턴 흐름 — 챕터 7에서 챕터 8 으로

```
[패턴 흐름 — 챕터 7에서 챕터 8 으로]
──────────────────────────────────────────────────────────────────────
 챕터 7: 데이터 자산을 규정·접근·암호화·연결로 "지킴"
      │ 남은 과제: 지키는 것과 별개로 "빠르고 싸게 읽히게" 만들어야 함
      ▼ "쿼리가 2분 걸린다. 컴퓨팅을 더 붙이는 것 말고 방법이 없나?"
 8.1 Partitioning (데이터를 나누기)
   #50 Horizontal Partitioner — row 전체를 파티션 값별로 다른 위치에 (날짜·국가 등 저카디널리티)
      │ (한계) 카디널리티가 높으면 파티션이 폭증 → 메타데이터·small files 문제
      │ (다른 축) row 안의 컬럼 그룹을 나누고 싶다면?
      ▼
   #51 Vertical Partitioner — row 를 컬럼 그룹으로 쪼개 다른 위치에 (불변 속성 1회 저장)
      │ (다음) 파티셔닝으로 안 되는 고카디널리티 컬럼은?
      ▼
 8.2 Records Organization (레코드를 묶고 정렬하기)
   #52 Bucket — 서로 다른 값을 같은 저장 공간에 묶음 (hash(key) % N) → 파티션 폭증 회피
   #53 Sorter — 디스크에 정렬해 저장 → 무관한 데이터 블록 skip (data skipping)
──────────────────────────────────────────────────────────────────────
 핵심 — 파티셔닝은 "위치를 나누는" 조잡한 도구, 버킷·정렬은 "그 안에서 잘 배치하는" 정교한 도구.
```

---

## 1. 파티셔닝 (Partitioning)

저장 계층 레이아웃을 정의할 때 가장 먼저 답해야 할 질문 — **"데이터셋을 어떻게 나눠야 접근하기 쉬운가?"**
답은 **수평(horizontal)·수직(vertical)** 두 방향의 조직화 패턴 두 개.

- **#50 Horizontal Partitioner** — **row 전체** 를 파티션 값에 따라 물리적으로 분리된 공간에 저장 (아래 1-1 절).
- **#51 Vertical Partitioner** — **한 row 를 컬럼 그룹으로 쪼개** 서로 다른 위치에 저장 (아래 1-2 절).

---

### 1-1. 패턴 #50: 수평 파티셔너 (Horizontal Partitioner)

> 데이터 조직화 접근 중 **수평 조직화** 가 아마 가장 흔히 쓰임 — 구현이 단순하고,
> 데이터 엔지니어링 초창기부터 오래 쓰여 온 인기 덕분.

#### 상황 (Problem)

**책의 use case** — 롤링 집계 배치 잡의 성능 저하:

- **직전 4일치 롤링 집계(rolling aggregates)** 를 계산하는 배치 잡을 만들었고, 몇 달간 잘 돌았음.
- 저장 계층에 데이터가 더 쌓이기 시작하자 **잡 성능이 떨어짐**.
- 가장 큰 문제 — **4일보다 오래된 레코드를 걸러내는 필터링 연산의 실행 시간 증가**.
- 임시 완화책으로 **잡 클러스터에 컴퓨팅 파워를 추가** 했지만, 그만큼 **비용이 올라감**.
- **결정적 제약**: 새 데이터가 계속 들어와도 **비용은 초기 수준으로 유지** 하면서 **실행 시간은 줄여야** 함.

#### 해결 (Solution)

문제 진술의 롤링 집계는 **전체 데이터셋의 일부만 쓰는 증분 처리(incremental processing)** 의 사례.
즉 **Horizontal Partitioner** 로 실행 시간과 비용의 균형을 잡기에 완벽한 조건.

- **파티셔닝 속성(partitioning attribute) 식별** — **분산 키(distribution key)** 라고도 부름.
  데이터 수집 프로세스나 데이터스토어가 이 속성을 써서, **파티셔닝 값마다 물리적으로 격리된 저장 공간** 에 데이터셋을 저장.
- **시간 기반 파티션** — 가장 인기 있는 형태. 데이터 처리 단계의 **시간 경계** 를 정의해 관련 정보만 빠르고 싸게 조회.
  시간 속성의 출처는 두 가지.
  - **잡 실행 컨텍스트(job execution context)** — 잡의 **실행 시각** 에 기반. **모든 레코드가 같은 파티션 값** 을 가짐.
    예: `2024-12-31` 에 실행된 잡의 모든 레코드는 그 실행 날짜 파티션에 안착.
  - **데이터셋(dataset)** — **이벤트 시각(event time)** 기준으로 파티셔닝. **지연 데이터(late data)** 현상 때문에
    파티션된 데이터셋에 **여러 파티션의 값이 섞여 들어올 수 있음**.
- **시간 말고도 가능** — **비즈니스 키**(고객 ID, 파트너 ID, 고객의 지리적 지역 등)도 파티셔닝 키가 될 수 있고,
  **시간 + 비즈니스 속성을 결합한 중첩 파티셔닝(nested partitioning)** 스키마까지 갈 수 있음.
- **파티션 정의 주체 — 두 모드**
  - **선언적(declarative)** — 테이블 생성 시 지정. Databricks·GCP BigQuery 의 `CREATE TABLE ... PARTITIONED BY`.
    **데이터 프로듀서는 하부 파티셔닝을 몰라도 되고**, 수집 시 파티션 값을 정의하지 않아도 됨.
  - **프로듀서 주도** — 파티셔닝 로직이 프로듀서에서 옴. Spark 의 `partitionBy`(기존 컬럼에서 파티션 생성,
    그 컬럼 자체가 복잡한 계산 결과일 수 있음), Kafka 의 **커스텀 partitioner 클래스**.
    유연성은 선언적 모드 쪽이 큼.
- **파티션 메타데이터** — 일부 데이터스토어는 **마지막 갱신 시각·row 수·생성 시각** 까지 관리.
  - GCP BigQuery — `INFORMATION_SCHEMA.PARTITIONS` 뷰
  - Databricks — `DESCRIBE TABLE EXTENDED` 출력의 일부
  - Apache Iceberg — `SELECT * FROM a_catalog.a_namespace.a_table.partitions`
- **멱등성과의 연결** — 데이터 조회 최적화 외에도, 수평 파티셔닝은 **멱등성의 중요한 구성 요소**.
  챕터 4 의 **Fast Metadata Cleaner 패턴** 이 파티셔닝을 활용해 멱등 파이프라인을 만드는 대표 사례.

```
[Example 8-1 재현] event time + country 중첩 파티셔닝
──────────────────────────────────────────────────────────────────────
 visits/
 └── 2024
     └── 05
          └── 05
              ├── france
              ├── india
              ├── poland
              └── usa
──────────────────────────────────────────────────────────────────────
 이벤트 시각(연/월/일) 파티션 위에 user country 속성을 한 겹 더 얹은 형태.
 → "2024-05-05 의 poland 방문" 만 읽는 쿼리는 디렉터리 하나만 스캔.
```

#### 고려사항 (Consequences)

역설적이게도, Horizontal Partitioner 의 가장 큰 단점은 **…수평 파티션 그 자체**, 더 정확히는 그 **정적(static) 성격**.

- **Granularity and metadata overhead (세분성과 메타데이터 오버헤드)**
  - 파티션은 **한 속성의 같은 값을 공유하는 엔티티들을 담는 물리적 위치**.
    따라서 **파티션이 너무 많으면** 데이터베이스에 부정적 영향.
  - 책의 예 — 웹사이트를 **하루에 100만 명의 고유 사용자** 가 방문하는데 **username 으로 파티셔닝** 하면
    **파티션 100만 개** 가 생김 ⇒ **파티션 리스팅 연산이 느려지고**, 읽을 **작은 파일이 폭증**(챕터 2 **Compactor** 의 small files 문제).
  - **경험칙** — **저카디널리티 속성**(서로 다른 값이 적은 속성)을 쓸 것.
    **시·일 단위로 반올림한 이벤트 시각** 이 좋은 예 — 보통 하루/한 시간마다 파티션 하나에 레코드 뭉치가 들어감.
    반대로 **IoT 디바이스 ID** 를 쓰면 수천 개의 작은 파티션이 생김 ⇒ 이런 경우는 **#52 Bucket 패턴** 이 더 나은 선택.
- **Skew (스큐)**
  - 수평 파티셔닝이 **균등 분포를 보장하지 않음**. 스큐 파티션은 흔한 지연 원인.
  - 대표 사례 — **마이크로배치 스트림 처리** 모델. 작은 배치를 증분 처리하되 **블로킹 방식**(이전 배치가 끝나야 다음 배치 실행).
    마이크로배치 안에 **불균형 파티션이 하나라도 있으면 그 파티션이 마이크로배치 전체 소요 시간을 결정** —
    짧은 파티션들이 먼저 처리되지 못하고 대기.
  - **완화책 — backpressure 메커니즘**: 스큐 파티션의 **잉여 레코드를 별도 버퍼에 저장** 하고 **다음 마이크로배치에서 처리**.
    스큐 파티션의 **전달 지연은 커지지만**, 나머지 태스크는 **거의 실시간** 으로 돌 수 있음.
- **Mutability (가변성)**
  - **파티션 키 변경이 어려움** — 이미 쓴 데이터를 전부 다른 위치로 옮겨야 하므로 **비싸고 오래 걸림**.
  - 일부 데이터스토어는 이를 조금 낫게 다룸. **Apache Iceberg** 는 **언제든 파티셔닝 스키마 변경 가능**.
    단 이는 **메타데이터 계층에서만 동작**(파일을 새 파티션으로 옮기지 않음) ⇒ **옛 레코드의 저장 구조는 그대로** 남고,
    새 조직은 **파티션 진화 이후 생성된 레코드에만** 적용됨.

```
[Figure 8-1 재현] 수평 파티션 스트리밍 브로커의 데이터 스큐 처리
──────────────────────────────────────────────────────────────────────
   ┌───────────┐        ┌──────────────┐        ┌────────────┐
   │   Input   │ ─────► │  Microbatch  │ ─────► │   Output   │
   └───────────┘        │     job      │        └────────────┘
         ▲              └──────┬───────┘
         │                     │
         │ <Load extra rows    │ <Save extra rows
         │  saved in the       │  saved for the
         │  previous           │  next microbatch>
         │  microbatch>        ▼
         │             ┌────────────────┐
         └─────────────┤  Backpressure  │
                       │     buffer     │
                       └────────────────┘
──────────────────────────────────────────────────────────────────────
 backpressure 버퍼는 "선택적 데이터 소스" 로 취급됨 — 이전 배치가 남긴 잉여 row 를 다음 배치가 함께 로드.
 ⚠ 스큐 파티션의 데이터 전달 지연은 커짐. 대신 나머지 태스크가 실시간에 가깝게 도는 것을 보장.
```

```
[파티션 카디널리티 — 무엇을 키로 잡을지]
──────────────────────────────────────────────────────────────────────
 ✓ 저카디널리티 (파티셔닝에 적합)
   event_time(일 단위)  → 365 파티션/년   각 파티션에 레코드 다수
   country              → 수십 개          균등하진 않아도 관리 가능

 ✗ 고카디널리티 (파티셔닝 부적합)
   username             → 100만 파티션    리스팅 지연 + small files 폭증
   iot_device_id        → 수천 파티션      각 파티션에 레코드 몇 건뿐
   ⇒ 이런 컬럼은 #52 Bucket 으로
──────────────────────────────────────────────────────────────────────
```

> **참고 사항 — 수평 파티셔닝 vs 샤딩 (Horizontal Partitioning Versus Sharding)**
> **샤딩(sharding)** 은 데이터셋을 **여러 머신** 으로 쪼개는 것으로, **물리적 데이터 분할** 을 수반.
> 수평 파티셔닝도 데이터셋을 여러 위치로 나누지만 **머신 간 데이터 이동을 요구하지 않음**.
> 따라서 **샤딩은 물리(하드웨어) 계층에 기반한 수평 파티셔닝의 특수한 한 종류**.

#### 구현 예시 (Examples)

**예시 1 — Apache Spark `partitionBy` (Example 8-2)**

Spark 의 내장 `partitionBy` 는 쓰는 데이터셋을 네이티브로 파티션 분할. `change_date` 컬럼에서 세분화된 파티션 컬럼을 만듦:
```python
partitioned_users = (input_users
 .withColumn('year', functions.year('change_date'))     # 파티션 컬럼을 미리 파생
 .withColumn('month', functions.month('change_date'))
 .withColumn('day', functions.day('change_date'))
 .withColumn('hour', functions.hour('change_date')))

(partitioned_users.write.mode('overwrite').format('delta')
 .partitionBy('year', 'month', 'day', 'hour')           # 프로듀서가 파티셔닝 로직을 소유
 .save(output_dir))
```

> 실행 후 잡은 `year/month/day/hour` 로 파티션된 데이터셋을 생성.
> 파티셔닝 경로에 들어 있는 값들을 **조합한 다양한 접근 패턴** 이 가능해짐 (연 단위·월 단위·시 단위 조회 모두).

**예시 2 — Apache Kafka 커스텀 partitioner (Example 8-3)**

Kafka 는 **커스텀 partitioner** 로 파티셔닝 로직을 직접 구현. 레코드 key 에 따라 파티션 0 또는 1 로 보냄
(partitioner 구현 제약 때문에 Java 로 작성):
```java
public class RangePartitioner implements Partitioner {

  private static final int DEFAULT_PARTITION = 1;
  private final static Map<String, Integer> RANGES_PER_PARTITIONS = new HashMap<>();
  static {
    RANGES_PER_PARTITIONS.put("A", 0);   // key "A", "B" 는 파티션 0
    RANGES_PER_PARTITIONS.put("B", 0);
  }

  @Override
  public int partition(String topic, Object key, byte[] keyBytes,
   Object value, byte[] valueBytes, Cluster cluster) {
    String keyAsString = key.toString();
    return RANGES_PER_PARTITIONS.getOrDefault(keyAsString, DEFAULT_PARTITION);
  }
// ...
```

**예시 3 — 프로듀서에 커스텀 partitioner 등록 (Example 8-4)**

만든 클래스를 `partitioner.class` 프로퍼티에 참조로 걸어야 동작:
```java
Properties props = new Properties();
// ...
props.put("partitioner.class", "com.waitingforcode.RangePartitioner");
```

> **참고 사항 — 단순하게 유지할 것! (Keep It Simple!)**
> **모든 코드는 복잡도를 늘림**. 그래서 항상 **단순함을 선호** 하고, **꼭 필요할 때만** 코드(=복잡도)를 추가하는 게 좋음.
> 결과적으로 **대부분의 경우 Apache Kafka 의 기본 partitioner 를 그대로 쓰게 될 것**.

**예시 4 — PostgreSQL 범위 파티셔닝 (Example 8-5)**

수평 파티셔너는 관계형 DB 에도 있음. 웹사이트 방문을 저장하는 테이블에서 **각 파티션이 서로 다른 하루** 를 담당:
```sql
CREATE TABLE visits_all (
  visit_id CHAR(36) NOT NULL,
  event_time TIMESTAMP NOT NULL,
  user_id TEXT NOT NULL,
  page VARCHAR(20) NULL,
  PRIMARY KEY(visit_id, event_time)
) PARTITION BY RANGE(event_time);          -- 파티셔닝 축 선언

CREATE TABLE visits_all_20231124 PARTITION OF visits_all
FOR VALUES FROM('2023-11-24 00:00:00') TO ('2023-11-24 23:59:59')

CREATE TABLE visits_all_20231125 PARTITION OF visits_all
FOR VALUES FROM('2023-11-25 00:00:00') TO ('2023-11-25 23:59:59')
```

| 구현 방식 | 파티셔닝 주체 | 특징 | 주의사항 |
|---|---|---|---|
| **선언적** (`CREATE TABLE ... PARTITIONED BY`, `CLUSTER BY`) | 데이터스토어 | 프로듀서가 파티셔닝을 몰라도 됨 | 스토어가 지원하는 축만 가능 |
| **Spark `partitionBy`** | 프로듀서(잡) | 계산된 컬럼도 파티션 키로 사용 가능 | 프로듀서가 파티셔닝 규칙을 알아야 함 |
| **Kafka 커스텀 partitioner** | 프로듀서 | key 기반 임의 로직 | 복잡도 증가 — 기본 partitioner 우선 |
| **PostgreSQL `PARTITION BY RANGE`** | DB | 파티션마다 실제 테이블 생성 | 파티션 테이블을 **미리** 만들어 둬야 함 |

<details>
<summary><b>⚠ 트러블 로그</b> — 조회에 편하다는 이유로 고카디널리티 컬럼을 파티션 키로 잡으면 파티션이 폭증해 오히려 리스팅에서 잡이 죽음.</summary>
<div markdown="1">

**예 —** "사용자별 조회가 80% 라서" `partitionBy('user_id')` 로 Delta 테이블을 만든 뒤,
DAU 100만 기준 **파티션 100만 개 · 파일 수백만 개** 가 생겨 다음 배치에서 파일 리스팅만 40분이 걸림.
Hive Metastore 를 쓰는 환경이라면 파티션 등록 단계에서 메타스토어 한계에 먼저 부딪혀
`MetaException: Timeout when executing add_partitions` 로 잡이 실패하기도 함.

**반대 함정 —** 카디널리티를 낮추겠다고 `partitionBy('year')` 처럼 **너무 굵게** 잡으면
파티션 하나가 수 TB 가 되어 프루닝 효과가 사라지고, `MERGE`·`DELETE` 한 번에 파티션 전체를 다시 씀.

**권장 —** 파티션 키는 **파티션당 파일 크기가 수백 MB~1GB** 에 들어오는 세분성으로 잡을 것.
`user_id` 처럼 카디널리티가 높은데 조회에 자주 쓰이는 컬럼은 **파티션이 아니라 #52 Bucket**(`bucketBy`) 으로 처리할 것.

</div>
</details>

---

### 1-2. 패턴 #51: 수직 파티셔너 (Vertical Partitioner)

> **#50 Horizontal Partitioner 는 매번 row 전체를 다룸**. 다음 파티셔닝 패턴은 그 대안 —
> **각 row 를 쪼개** 분리된 조각들을 테이블·파일 같은 **서로 다른 곳** 에 씀.

> **참고 사항 — 스토리지용과 보안용 (For Storage and Security)**
> 챕터 7 에서 소개한 **Vertical Partitioner(#41)** 는 수직 파티셔닝을 **보안에 적용한 특화 버전**.
> 이 챕터의 Vertical Partitioner(#51)는 **데이터 스토리지에 헌정된 특화 버전**.

#### 상황 (Problem)

**책의 use case** — 방문 데이터셋의 불변 속성 중복 저장:

- 파이프라인 중 하나에서 **웹사이트 사용자 방문(visits)** 을 추적 중.
- visits 데이터셋의 속성은 두 부류.
  - **가변(mutable)** — 방문마다 바뀜 (예: **방문 시각(visit time)**, **방문 페이지(visited page)**)
  - **불변(immutable)** — 방문 내내 동일 (예: **IP 주소**)
- **결정적 제약**: **불변 정보를 중복 저장하지 않고, 방문당 딱 한 번만** 저장할 방법이 필요.

#### 해결 (Solution)

문제 진술처럼 **두 종류의 속성** 을 가진 것이 **Vertical Partitioner** 를 쓰기에 완벽한 조건.

- **① 데이터 분류(data classification)** — **관련 속성끼리 묶음**. 여기서는 **가변 그룹 / 불변 그룹** 으로 나눔.
- **② 재결합용 속성 식별** — 나중에 필요하면 두 조각을 다시 합칠 키. 이 예에서는 **`visit_id`**.
- **③ 쓰기** — 이 명세 단계가 끝나면, 데이터 처리 잡이 **그룹별 속성을 전용 위치**
  (데이터스토어의 테이블, 파일 시스템의 디렉터리 등)에 씀.
- **스토리지 비용 절감 + 유연성** — row 가 나뉘어 있으므로 조각마다
  **서로 다른 데이터 보존(retention)·데이터 접근(access) 정책** 을 쉽게 적용 가능.
  row 를 통째로 두면 훨씬 까다로울 일.
- **#50 과의 차이는 파티셔닝 휴리스틱** — 수평은 **row 전체에 규칙을 적용해 통째로 다른 위치로** 옮기고,
  수직은 **row 를 쪼개 각기 다른 위치에** 씀.

```
[Figure 8-2 재현] 한 visit row 의 수평 분할 vs 수직 분할
──────────────────────────────────────────────────────────────────────
 ■ Horizontal partitioning
   Raw                                              Partitioned by event_time
   [ visit_id | visit_time | page | visit_ip ] ──►  [ visit_id | visit_time | page | visit_ip ]
   [ visit_id | visit_time | page | visit_ip ] ──►  [ visit_id | visit_time | page | visit_ip ]
   [ visit_id | visit_time | page | visit_ip ] ──►  [ visit_id | visit_time | page | visit_ip ]
                                                    ⇒ row 가 통째로 다른 위치로 이동

 ■ Vertical partitioning
   Raw                                        Partitioned · mutable   ┊  Partitioned · immutable
   [ visit_id | visit_time | page | visit_ip ] ──► [*visit_id*| visit_time | page ] ┊ [*visit_id*| visit_ip ]
   [ visit_id | visit_time | page | visit_ip ] ──► [*visit_id*| visit_time | page ] ┊ [*visit_id*| visit_ip ]
   [ visit_id | visit_time | page | visit_ip ] ──► [*visit_id*| visit_time | page ] ┊ [*visit_id*| visit_ip ]
                                                    ⇒ row 가 쪼개져 두 위치로 분산
──────────────────────────────────────────────────────────────────────
 ┊ 점선 = 서로 다른 두 파티셔닝 위치 (책 그림의 dashed line).
 *visit_id* = 책 그림의 "채워진 박스" — 쪼개진 row 를 다시 합칠 때 쓰는 고유 ID.
```

```
[왜 저장량이 줄어드나 — visit_ip 가 방문 내내 같은 경우]
──────────────────────────────────────────────────────────────────────
 ✗ 비분할 (한 테이블)
   visit_id | visit_time       | page       | visit_ip
   v1       | 2024-05-01T10:00 | home.html  | 10.0.0.7
   v1       | 2024-05-01T10:03 | about.html | 10.0.0.7   ← 같은 IP 를 이벤트 수만큼 반복 저장
   v1       | 2024-05-01T10:09 | cart.html  | 10.0.0.7

 ✓ 수직 분할 (두 위치)
   mutable (이벤트 수만큼)                    immutable (visit 당 1행)
   visit_id | visit_time       | page        visit_id | visit_ip
   v1       | 2024-05-01T10:00 | home        v1       | 10.0.0.7   ← 1벌만
   v1       | 2024-05-01T10:03 | about
   v1       | 2024-05-01T10:09 | cart
──────────────────────────────────────────────────────────────────────
 ⇒ 방문 안 이벤트가 많을수록 절감 폭이 큼. 덤으로 immutable 쪽에만 별도 retention·접근 정책을 걸 수 있음.
```

#### 고려사항 (Consequences)

Vertical Partitioner 는 다음 영역에서 **논리적 파급 효과** 를 가짐.

- **Domain split (도메인 분할)**
  - 각 row 가 쪼개지므로, **논리적으로 관련된 속성들이 서로 다른 두 곳** 에 저장될 수 있음.
    **찾기가 쉽지 않을 수 있고**, 그래서 **최종 사용자를 위한 좋은 문서화 지원이 핵심**.
- **Querying (조회)**
  - 도메인 분할에서 따라오는 단점 — row 가 분리돼 있으니 **전체 그림을 얻기가** 수평 파티션 데이터셋보다 **어려워짐**.
  - **완화** — 수직 파티션된 엔티티의 **모든 테이블을 합친 뷰(view)** 로 데이터를 노출
    (예: **Dataset Materializer 패턴** 활용).
- **Data producer (데이터 프로듀서)**
  - 컨슈머 (조회하는 쪽)뿐 아니라 **프로듀서에도 영향** — 이제 **row 를 그대로 집어 다른 곳에 쓸 수 없음**.
  - 프로듀서가 **row 분할 로직을 구현** 해야 하고, 결과적으로 **여러 번 write** 를 수행 ⇒ **네트워크 통신 비용이 커질 수 있음**.

| 파티셔닝 | 무엇을 나누나 | 재결합 방법 | 대표 쓰임 |
|---|---|---|---|
| **#50 수평(horizontal)** | **row** 를 파티션 값별 위치로 | 재결합 불필요 (row 온전) | 날짜·국가 기반 증분 처리 |
| **#51 수직(vertical)** | **컬럼 그룹** 을 서로 다른 위치로 | 공통 ID(`visit_id`) 로 join | 불변 속성 1회 저장, 조각별 정책 분리 |

#### 구현 예시 (Examples)

**예시 1 — Apache Spark 로 user·technical 컨텍스트 분리 (Example 8-6)**

방문의 **user 컨텍스트** 와 **technical 컨텍스트** 를 두 테이블로 추출. 입력을 두 번 읽지 않도록 **`persist()` 호출을 잊지 말 것**:
```python
visits = spark_session.read.schema(visit_schema).json(input_location)
visits.persist()                                   # ⚠ 입력이 3번 쓰이므로 캐시 필수

# 파티션 1 — 가변 방문 데이터 (user·technical 컨텍스트 제거)
visits_without_user_technical_context = (visits.drop('user_id')
 .withColumn('context', F.col('context').dropFields('user'))
 .withColumn('context', F.col('context').dropFields('technical')))
visits_without_user_technical_context.write.format('delta').save(output_dir)

# 파티션 2 — user 컨텍스트만 (visit_id 를 재결합 키로 남김)
(visits.selectExpr('visit_id', 'context.user.*', 'user_id').dropDuplicates()
.write.format('delta').save(get_delta_users_table_dir()))

# 파티션 3 — technical 컨텍스트만
(visits.selectExpr('visit_id', 'context.technical.*').dropDuplicates()
.write.format('delta').save(get_delta_technical_table_dir()))

visits.unpersist()
```

> **`drop()`** 으로 제거할 컬럼을, **`select()`** 로 남길 컬럼을 각각 지정해 두 테이블을 만듦.
> `dropDuplicates()` 가 **불변 속성을 한 번만** 저장하는 목표를 달성해 줌.

**예시 2 — PostgreSQL `INSERT INTO ... SELECT FROM` (Example 8-7)**

각 row 를 명시적으로 선언하는 대신, **동적 `SELECT` 쿼리에 선언을 위임**:
```sql
INSERT INTO dedp.technical (visit_id, browser, browser_version, ...)
 (SELECT DISTINCT visit_id, context->'technical'->>'browser',
    context->'technical'->>'browser_version', ...
  FROM dedp.visits_all);
```

**예시 3 — CTAS 구성 (Example 8-8)**

`SELECT` 문에서 수직 파티션 테이블을 아예 **새로 만드는** 방식 — 흔히 **CTAS(CREATE TABLE AS SELECT)** 라 부름:
```sql
CREATE TABLE dedp.technical_select AS (SELECT DISTINCT
  visit_id, context->'technical'->>'browser' AS browser,
  context->'technical'->>'browser_version' AS browser_version, ...
  FROM dedp.visits_all;
```

```
[Examples 8-6 ~ 8-8 한 흐름 — 한 row 를 세 위치로]
──────────────────────────────────────────────────────────────────────
 raw visits (JSON)
      │  persist()
      ├─► drop(user_id) + dropFields(user, technical)  ─► [ visits (가변) ]        Delta
      ├─► select(visit_id, context.user.*)   + distinct ─► [ users (불변) ]        Delta
      └─► select(visit_id, context.technical.*) + distinct ─► [ technical (불변) ] Delta
                                                                    │
      컨슈머 (조회하는 쪽) ◄── view / Dataset Materializer 로 재결합 ◄┘  (join key = visit_id)
──────────────────────────────────────────────────────────────────────
 ⚠ persist() 를 빼면 입력 JSON 을 3번 읽음 — 수직 분할의 절감분을 읽기 I/O 로 되돌려줌.
```

<details>
<summary><b>⚠ 트러블 로그</b> — 수직 분할 후 재결합 키를 각 조각에 남기지 않으면 두 조각을 영영 못 붙임.</summary>
<div markdown="1">

**예 —** `context.technical.*` 만 `SELECT` 하고 `visit_id` 를 빼먹은 채 5억 건을 Delta 로 적재.
브라우저별 분석을 하려고 방문 테이블과 붙이는 순간 조인 키가 없어 **전량 재적재** 외에 방법이 없었음.
반대로 `dropDuplicates()` 를 빼면 technical 테이블이 방문 이벤트 수만큼 부풀어
"불변 속성을 한 번만 저장" 이라는 패턴의 목적 자체가 사라짐.

**반대 함정 —** 재결합 키를 남겼더라도 `persist()` 없이 브랜치를 3개 만들면
입력 JSON 을 3번 파싱해, 쓰기에서 아낀 시간을 읽기에서 그대로 토해냄.

**권장 —** 수직 분할 설계는 **① 컬럼 그룹 정의 → ② 재결합 키 확정 → ③ 각 조각에 키 강제 포함** 순서를 문서로 못 박고,
공통 입력에는 반드시 `persist()`(또는 임시 테이블)를, 불변 조각에는 반드시 `dropDuplicates()`/`DISTINCT` 를 붙일 것.
컨슈머 (다운스트림)에는 raw 테이블이 아니라 **재결합 view** 를 표준 진입점으로 노출할 것.

</div>
</details>

---

## 2. 레코드 조직화 (Records Organization)

파티셔닝은 데이터 조직화의 **첫 단계** 지만, 본 것처럼 **다소 조잡함(rudimentary)** —
전체 또는 부분 레코드를 **다른 위치로 옮기는 것** 이 전부. 게다가 **모든 속성에 쓸 수도 없음**
(고카디널리티 값은 수평 파티셔닝에 부적합).

다음 패턴 계열은 한 걸음 더 나아가 **레코드 배치(colocation)에 똑똑한 최적화** 를 적용 —
그중 하나가 **Horizontal Partitioner 의 카디널리티 문제** 를 다룸.

- **#52 Bucket** — 서로 다른 값을 **같은 저장 공간에 묶음**(`hash(key) % N`) (아래 2-1 절).
- **#53 Sorter** — 데이터를 **디스크에 정렬해 저장** 해 무관한 블록을 skip (아래 2-2 절).

---

### 2-1. 패턴 #52: 버킷 (Bucket)

> 고유 user ID 처럼 **카디널리티가 높은 컬럼** 의 접근을 개선해야 한다면 방법이 있음.
> 파티셔닝으로 **row 들을 같은 저장 공간에 배치** 하는 대신, **row 들의 그룹(groups of rows)** 을 배치하는 것.
> — 이것이 이 패턴의 (지나치게 단순화한) 정의.

#### 상황 (Problem)

**책의 use case** — 고카디널리티 비즈니스 속성:

- 모델링 중인 데이터셋에 **쿼리 술어(predicate)에 자주 쓰이는 비즈니스 속성** 이 있음.
- 처음엔 이 속성을 **파티셔닝 컬럼** 으로 쓰려 했으나 **카디널리티가 너무 높음**.
  파티션이 너무 많아져 **어느 시점엔 데이터스토어의 메타데이터 한계에 도달** 할 수 있음.
- **결정적 제약**: **연산의 80% 가 이 고카디널리티 속성에 의존** — 스토리지를 최적화하고는 싶은데
  파티셔닝은 못 쓰는 상황.

#### 해결 (Solution)

**쿼리에 자주 등장하는 고카디널리티 컬럼** 을 가진 것이 **Bucket 패턴** 을 쓸 좋은 이유.
겉보기엔 이것도 레코드를 전용 위치에 저장하지만, **#50 Horizontal Partitioner 와 달리
서로 다른 값을 같은 저장 영역에 함께 둠**.

- **① 버킷 컬럼(bucket columns) 결정** — 두 파티셔닝 패턴과 마찬가지로 **데이터 분석 단계** 로 시작.
  데이터셋이 이미 수평·수직 파티션돼 있다면, 이 속성들은 **2차 그룹핑 키** 로 볼 수 있음
  (파티션 키가 1차 키).
- **② 버킷 개수 설정** — 버킷 키의 **카디널리티에 의존**.
  - 카디널리티가 아주 높다 = 고유 값이 많다는 뜻.
  - **개수를 크게** ⇒ **더 작은 버킷이 많이**, **개수를 작게** ⇒ **더 큰 버킷이 적게**.
  - 이 의존 관계는 **모듈러 해싱(modular hashing)** 그룹핑 공식에서 옴 —
    각 key 의 버킷 번호는 **`hash(key) % buckets number`** 로 계산.
- **컨슈머 (조회하는 쪽)를 위한 두 가지 최적화**
  - **버킷 프루닝(bucket pruning)** — **버킷 컬럼이 쿼리의 술어로 쓰이면**, 쿼리 실행 엔진이 **버킷팅 알고리즘을 직접 적용해**
    필요한 key 가 없는 **버킷을 모두 제거**. 모든 필터링 연산에서 **상당한 성능 향상**.
  - **네트워크 교환(shuffle) 제거** — **양쪽이 동일한 버킷팅 구성** 을 가진 `JOIN` 연산에 적용.
    쿼리 러너가 버킷을 활용해 **각 데이터셋의 상관 레코드를 같은 join 프로세스로 직접 로드** ⇒
    **Distributed Aggregator 패턴** 에서 본 네트워크 교환 없이 결합.
- **역사** — 버킷팅 기능은 **Apache Hive** 가 대중화했고, 이후 **Apache Spark·AWS Athena** 등
  현대 데이터 솔루션에 통합됨.

```
[버킷팅 공식 — hash(key) % N]
──────────────────────────────────────────────────────────────────────
 bucket columns = user_id ,  buckets number = 50

   user_id = "u-91823"  ─► hash("u-91823") % 50 = 17  ─► bucket_00017 파일에 기록
   user_id = "u-44012"  ─► hash("u-44012") % 50 =  3  ─► bucket_00003 파일에 기록
   user_id = "u-77105"  ─► hash("u-77105") % 50 = 17  ─► bucket_00017 (다른 값이 같은 버킷에 공존)

 ⇒ 파티션: 값 하나 = 위치 하나 (100만 값 → 100만 위치)
   버킷  : 여러 값 = 위치 하나 (100만 값 → 50 위치)
──────────────────────────────────────────────────────────────────────
 이것이 "서로 다른 값을 같은 저장 영역에 배치" 의 의미 — 고카디널리티에서도 위치 수가 폭발하지 않음.
```

```
[Figure 8-3 재현] 동일 버킷 구성 테이블 간 shuffle 없는 분산 join
──────────────────────────────────────────────────────────────────────
 Table 1 / bucket column = letter
 ┌──────────┬──────────┬──────────┐
 │  A   B   │  E   F   │  L   M   │
 │  X   Y   │  J   K   │  N       │        ┌────────────────────────────────┐
 │ Bucket 1 │ Bucket 2 │ Bucket 3 │        │   Data loaded per join task    │
 └──────────┴──────────┴──────────┘        ├────────────────────────────────┤
                            │              │ [A B / X Y] + [A B / X]        │
                            ├─► JOIN   ──► │        Join task for Bucket 1  │
                            │   ON letter  ├────────────────────────────────┤
 Table 2 / bucket column = letter          │ [E F / J K] + [E F]            │
 ┌──────────┬──────────┬──────────┐        │        Join task for Bucket 2  │
 │  A   B   │  E   F   │  L   M   │        ├────────────────────────────────┤
 │  X       │          │  N       │        │ [L M / N ] + [L M / N]         │
 │ Bucket 1 │ Bucket 2 │ Bucket 3 │        │        Join task for Bucket 3  │
 └──────────┴──────────┴──────────┘        └────────────────────────────────┘
──────────────────────────────────────────────────────────────────────
 양쪽 테이블이 letter 로 "똑같이" 버킷팅돼 있으므로, 같은 번호의 버킷끼리만 만나면 됨.
 ⇒ 네트워크 shuffle 없이 join task 하나가 자기 버킷 짝만 로드. 버킷 구성이 다르면 이 최적화는 사라짐.
```

#### 고려사항 (Consequences)

또다시 **데이터가 정적(static)** 이라는 점이 Bucket 패턴의 가장 큰 문제.

- **Mutability (가변성)**
  - **버킷팅 스키마는 불변**. 기술적으로는 **컬럼이나 버킷 크기를 바꿀 수 있지만**,
    이는 **데이터셋 backfill 을 요구하는 비싼 연산**.
- **Bucket size (버킷 크기)**
  - Bucket 패턴은 **버킷 크기 설정을 요구** 하는데, 미래에 데이터가 더 들어올 것을 예상한다면 **적정 크기를 찾기가 어려움**.
  - **현재 볼륨 기준** 으로 잡으면 → 미래에 **큰 버킷** 이 생김.
  - **개수를 미리 예측** 하면 → 예측이 정확하리란 보장이 없고, 그 사이 writer 가 **필요 이상으로 많은 버킷** 을 만들 수 있음.
  - 둘 다 **수용 가능한 완화책** 이지만, 보다시피 **각각 함정** 이 있음.
- **직접 key 접근 (책의 패턴 요약표 기준)**
  - 버킷 하나에는 **여러 key 가 섞여** 있으므로, **key 하나에 직접 접근** 하려면 **여러 key 를 거쳐 지나가야** 함.
    파티션처럼 "값 하나 = 위치 하나" 가 아니라는 대가.

| 항목 | #50 Horizontal Partitioner | #52 Bucket |
|---|---|---|
| **적합 카디널리티** | 낮음 (날짜·국가) | 높음 (user_id·device_id) |
| **값 ↔ 위치 관계** | 값 1개 = 위치 1개 | 값 여러 개 = 위치 1개 (`hash % N`) |
| **위치 수 증가** | 고유 값 수에 비례 (폭증 위험) | 버킷 개수로 고정 |
| **주요 이득** | 파티션 프루닝, 멱등성(Fast Metadata Cleaner) | 버킷 프루닝, shuffle 없는 join |
| **변경 비용** | 파티션 키 변경 = 전체 이동 | 스키마 불변 — 변경 시 backfill |

#### 구현 예시 (Examples)

**예시 1 — AWS Athena 버킷팅 설정 (Example 8-9)**

**Amazon Athena** 는 **논리 수준에서** Bucket 패턴을 구현하는 서버리스 쿼리 서비스 —
**데이터를 직접 쓰지 않고**, S3 에 이미 저장된 테이블에 **기존 버킷팅 로직을 적용** 하기만 함.
그래서 **버킷 테이블에 `INSERT INTO` 를 날리면 에러**가 남.
```sql
CREATE EXTERNAL TABLE visits (...) ...
CLUSTERED BY (`user_id`) INTO 50 BUCKETS       -- 버킷 컬럼 + 버킷 개수
TBLPROPERTIES ('bucketing_format' = 'spark')   -- 버킷팅 포맷도 함께 선언
```

**예시 2 — Apache Spark `bucketBy` (Example 8-10)**

Spark 는 `bucketBy` 호출로 버킷 테이블을 생성 — 앞서 본 **모듈로 기반 알고리즘** 을 버킷 컬럼에 적용:
```python
input_dataset.write.bucketBy(50, 'user_id').saveAsTable(table_name)
```

> `saveAsTable` 인 점에 주의 — 버킷 정보는 **메타스토어에 등록** 돼야 컨슈머 (쿼리 엔진)가 프루닝·shuffle 제거에 활용할 수 있음.

<details>
<summary><b>⚠ 트러블 로그</b> — 조인 양쪽의 버킷 구성이 조금이라도 다르면 shuffle 제거가 조용히 사라져 "버킷팅했는데 왜 안 빨라지지" 가 됨.</summary>
<div markdown="1">

**예 —** `visits` 를 `bucketBy(50, 'user_id')`, `users` 를 `bucketBy(200, 'user_id')` 로 만들어 두고
조인이 여전히 느려 원인을 찾다가, 실행 계획에 `Exchange hashpartitioning(user_id, 200)` 이 그대로 남아 있는 것을 발견.
버킷 **개수가 다르면** 엔진은 한쪽을 다시 셔플하므로 최적화가 무효.
버킷 **컬럼 순서**(`bucketBy(50,'user_id','page')` vs `bucketBy(50,'page','user_id')`)나
`saveAsTable` 대신 `save(path)` 로 저장해 메타스토어 등록이 빠진 경우도 같은 결과.

**반대 함정 —** 버킷 수를 미래 대비로 크게(예: 2,000) 잡으면 파티션마다 2,000개의 작은 파일이 생겨
Compactor 로 해결하려던 small files 문제를 버킷팅으로 다시 만들어 냄.

**권장 —** 조인에 쓸 테이블들은 **버킷 컬럼·개수·순서를 하나의 표준으로 고정** 하고,
`explain()` 실행 계획에서 **`Exchange` 가 사라졌는지** 로 검증할 것.
버킷 수는 **버킷당 파일이 128MB~1GB** 가 되는 선에서 잡고, 바꿔야 하면 backfill 비용을 미리 계산할 것.

</div>
</details>

---

### 2-2. 패턴 #53: 정렬기 (Sorter)

> 버킷으로 레코드 그룹을 배치하는 것만이 저장 최적화는 아님.
> 쿼리와 무관한 데이터 블록을 제거하는 **또 다른 기법** 이 **데이터 저장 순서** 에 기댐.

#### 상황 (Problem)

**책의 use case** — 주간 테이블을 쓰지만 쿼리는 여전히 느림:

- **Fast Metadata Cleaner 패턴** 을 활용하려고 **주간 테이블(weekly tables)** 에 데이터를 저장하기로 결정.
- 덕분에 **일상 유지보수 작업은 덜 고통스러워졌지만**, **쿼리 실행 시간은 개선되지 않음**.
- **결정적 제약**: 이 **멱등성 전략은 바꾸고 싶지 않으면서**, 동시에 **데이터 접근 지연은 줄이고** 싶음.
- **좋은 소식** — **사용자 쿼리 유형을 알고 있음**. 대부분이 **`event time` 컬럼으로 필터하거나 정렬**.

#### 해결 (Solution)

**정렬·필터에 흔히 쓰이는 컬럼을 아는 것** 이 **Sorter 패턴** 으로 데이터 접근을 최적화하는 좋은 출발점.

- **① 정렬 컬럼 식별** → **② 테이블 생성 쿼리에 정렬 컬럼 선언** → 이후 **데이터베이스가 쓰이는 row 들을 정의된 순서로 조직**.
- **효과 — 데이터 스키핑(data skipping)**: 정렬된 저장 덕분에, **정렬 컬럼을 타깃으로 하는 어떤 쿼리든
  무관한 데이터 블록을 건너뛸 수 있음**. 이는 아주 흔히 **각 블록에 딸린 메타데이터 정보** 덕분
  (자세한 내용은 **#54 Metadata Enhancer** 패턴).
- **곡선 정렬(curved sorts)** — 고전적인 위→아래 정렬 알고리즘의 변형으로, **결과가 수직으로 정렬** 됨.
  대표 예가 **Z-order** — **사전식(lexicographical) 순서 대신 x-차원 공간의 row 들을 배치**.
  - Z-order 알고리즘 자체의 상세 설명은 책의 범위 밖이지만,
    **Apache Iceberg·Delta Lake** 같은 테이블 파일 포맷이 **네이티브로 구현**.
  - **여러 컬럼** 에 대해 Z-order 가 사전식보다 나은 이유가 중요 — **읽어야 할 데이터 블록 수가 줄어듦**.
- **다른 구현들** — **Amazon Redshift** 는 **interleaved sort keys** 기능으로 Z-curve 기반의 Z-order 유사 정렬을 제공.
  **GCP BigQuery·Snowflake** 는 **clustered tables** 로 고전적 정렬을 제공.

```
[Figure 8-4 재현] 데이터 스키핑을 위한 메타데이터 정보
──────────────────────────────────────────────────────────────────────
 ┌─ File A ────────────────────────────────────┐
 │ Visit_time        | page       | id         │
 │ 2024-05-01T00:03  | home.html  |  1         │
 │ 2024-05-01T00:00  | about.html |  2         │
 │ 2023-05-01T01:00  | home.html  |  3         │
 │ 2022-05-01T04:02  | about.html |  4         │
 │ 2022-05-01T04:00  | home.html  |  5         │
 ├─────────────────────────────────────────────┤
 │ visit_time range: 2022-05-01T04:00 –        │
 │                   2024-05-01T00:03          │
 └─────────────────────────────────────────────┘

 ┌─ File B ────────────────────────────────────┐
 │ Visit_time        | page       | id         │
 │ 2024-05-01T11:00  | home.html  |  7         │
 │ 2024-05-01T10:03  | about.html |  8         │
 │ 2023-05-01T11:00  | home.html  |  9         │
 │ 2023-05-01T10:40  | about.html | 10         │
 ├─────────────────────────────────────────────┤
 │ visit_time range: 2023-05-01T10:40 –        │
 │                   2024-05-01T11:00          │
 └─────────────────────────────────────────────┘
──────────────────────────────────────────────────────────────────────
 쿼리가 visit_time 을 두 range 중 한쪽 안에서 타깃하면, 엔진은 다른 파일 하나를 아예 열지 않아도 됨.
 예: WHERE visit_time = '2022-05-01T04:02' ⇒ File B 는 range 밖 ⇒ File B skip.
```

```
[Figure 8-5 재현] 두 컬럼 정렬에서 술어가 읽는 데이터 블록 수
──────────────────────────────────────────────────────────────────────
                    SELECT ... FROM ... WHERE X = 0 OR Y = 0
                                     │
              ┌──────────────────────┴──────────────────────┐
              ▼                                             ▼
   Lexicographical order                              Z-order
   x 축으로 먼저 다 훑고 y 로 넘어가는                  x·y 를 번갈아 접어 2×2 근방을
   "가로로 긴" 블록 배치                               한 블록에 담는 "곡선" 배치

   (0,0)(1,0)(2,0)(3,0) │ (4,0)(5,0)...              (0,0)(1,0) │ (2,0)(3,0) │ (4,0)(5,0)
   (0,1)(1,1)(2,1)(3,1) │ (4,1)(5,1)...              (0,1)(1,1) │ (2,1)(3,1) │ (4,1)(5,1)
   ────────────────────────────────────              ─────────────────────────────────────
   (0,2)(1,2)(2,2)(3,2) │ (4,2)(5,2)...              (0,2)(1,2) │ (2,2)(3,2) │ (4,2)(5,2)
   (0,3)(1,3)(2,3)(3,3) │ (4,3)(5,3)...              (0,3)(1,3) │ (2,3)(3,3) │ (4,3)(5,3)
              ⋮                                                  ⋮

   ⇒ 읽는 블록 9개                                    ⇒ 읽는 블록 7개
──────────────────────────────────────────────────────────────────────
 Y=0 조건은 두 방식 모두 첫 행 블록들을 읽게 하지만, X=0 조건에서 차이가 벌어짐.
 사전식은 x=0 이 "모든 가로 블록의 맨 앞"에 흩어져 있어 세로로 훑는 블록이 많아짐.
 Z-order 는 x·y 근방을 함께 묶어 두므로 같은 술어에 더 적은 블록만 열면 됨.
```

> **참고 사항 — 정렬 vs 클러스터링 (Sorting Versus Clustering)**
> Z-order 는 관련 레코드를 같은 파일에 배치한다는 점에서 **클러스터링 맥락** 으로도 언급됨.
> 하지만 그 방식은 **사전식 정렬이 하듯 데이터를 디스크에 실제로 정렬** 하는 것.
> 그래서 이 책에서는 Z-order 를 **Sorter 패턴의 사례로 분류**.

#### 고려사항 (Consequences)

**미리 정렬된 데이터셋은 reader 성능에 긍정적** 이지만, **writer 에는 부정적**.

- **Unsorted segments (정렬 안 된 세그먼트)**
  - 정렬이 **항상 즉시 일어나는 활동은 아님**. 즉 **새 레코드를 쓸 때마다 정렬되지 않은 블록** 이 생기고,
    이들은 **Sorter 의 최적화 혜택을 못 받음**.
  - **완화** — 정렬 작업을 **데이터 쓰기 잡 안에서** 또는 **밖에서** 스케줄링.
    단 **쓰기 프로세스에 정렬을 통합하면 실행 시간에 영향** 을 준다는 점을 유념.
- **Composite sort keys (복합 정렬 키)**
  - 사전식 순서에서 복합 정렬 키를 쓸 때는, **쿼리가 항상 타깃 컬럼보다 앞선 정렬 컬럼(들)을 함께 참조해야** 함.
  - 그러지 않으면 **정렬을 선언했음에도 쿼리 엔진은 대부분의 데이터 블록을 순회** 해야 함.
- **Mutability (가변성)**
  - 생성 후 **정렬 키를 바꾸는 것은 대개 가능** 하지만, 그 연산이 **테이블 전체를 정렬** 해야 할 수 있음.
    테이블 크기에 따라 **비쌀 수 있음**.

```
[Figure 8-6 재현] visit_time·page 로 정렬된 테이블 — 술어에 따라 읽는 행
──────────────────────────────────────────────────────────────────────
                        │ Visit_time        | page       | id │
   visit_time =         │                   |            |    │
   "2024-05-01"    ───► │ 2024-05-01T10:00  | home.html  | 1  │ ◄───┐
      AND          ───► │ 2024-05-01T10:03  | about.html | 2  │ ◄───┤
   page =               │ 2023-05-01T01:00  | home.html  | 3  │ ◄───┤ page =
   "home.html"          │ 2023-05-01T04:00  | about.html | 4  │ ◄───┤ "home.html"
                        │ 2022-05-01T04:00  | home.html  | 5  │ ◄───┘
                        └─────────────────────────────────────┘
    ✓ 선행 컬럼(visit_time)까지 걸면       ✗ 후행 컬럼(page)만 걸면
      상위 블록만 읽고 나머지 skip           정렬 선언에도 불구하고 전 행 순회
──────────────────────────────────────────────────────────────────────
 정렬 키가 (visit_time, page) 일 때, page 단독 필터는 데이터 스키핑 혜택을 받지 못함.
 ⇒ 이 문제를 푸는 게 곡선 정렬(Z-order) — 컬럼 간 우선순위가 없어 단독 필터에도 효과.
```

| 정렬 방식 | 배치 | 강점 | 약점 |
|---|---|---|---|
| **사전식(lexicographical)** | 선행 컬럼 우선 top-to-bottom | 선행 컬럼 필터에 매우 강함 | 후행 컬럼 단독 필터엔 무력 |
| **곡선(curved, Z-order)** | x-차원 공간 근방을 함께 배치 | **여러 컬럼 어느 쪽으로 필터해도** 블록 감소 | 재조직(compaction) 비용 |

#### 구현 예시 (Examples)

**예시 1 — GCP BigQuery clustered table (Example 8-11)**

BigQuery 는 **clustered tables** 로 Sorter 를 구현. 정렬 컬럼을 `CLUSTER BY` 로 선언:
```sql
CREATE TABLE `dedp.visits.raw_visits`
PARTITION BY DATE(event_time)      -- #50 수평 파티셔닝과 함께 씀
CLUSTER BY visit_id, page          -- #53 정렬 (사전식, 선행 컬럼 = visit_id)
```

> clustered table 은 **`visit_id` 와 `page` 를 함께** 타깃할 때 성능을 개선하지만,
> **`page` 만 필터** 할 때는 큰 도움이 안 됨 (위 Figure 8-6 의 복합 정렬 키 함정).

**예시 2 — Delta Lake Z-order compaction (Example 8-12)**

곡선 정렬이 그 문제를 해결. 곡선 분포를 만들 컬럼을 optimize API 에 넘김:
```python
DeltaTable.forPath(spark, output_dir)
  .optimize().executeZOrderBy(['visit_id', 'page'])
```

> 그 결과 Delta Lake 는 **데이터 파일을 compaction** 해, 다시 쓰인 파일 내부의 레코드를 더 잘 조직.
> 즉 Z-order 는 **쓰기 시점이 아니라 별도 최적화 잡** 으로 도는 게 일반적 — 위 "Unsorted segments" 고려사항과 직결.

```
[#50 · #52 · #53 을 함께 쓰는 전형적 레이아웃]
──────────────────────────────────────────────────────────────────────
 CREATE TABLE visits
   PARTITION BY DATE(event_time)      ─► #50 저카디널리티 축으로 위치 분리 (파티션 프루닝)
   CLUSTERED BY (user_id) INTO 50     ─► #52 고카디널리티 축을 버킷으로 (버킷 프루닝 · shuffle 제거)
   CLUSTER BY visit_id, page          ─► #53 블록 내부 정렬 (데이터 스키핑)

 쿼리 WHERE event_time = '2024-05-05' AND user_id = 'u-91823' AND page = 'home.html'
   ① 파티션 프루닝  → 2024-05-05 디렉터리만
   ② 버킷 프루닝    → hash('u-91823') % 50 = 17 → bucket_00017 만
   ③ 데이터 스키핑  → 블록 메타데이터 range 로 무관 블록 skip
──────────────────────────────────────────────────────────────────────
 세 패턴은 배타적이지 않고 층층이 쌓임 — 위치 → 그룹 → 블록 순으로 읽을 데이터를 좁혀 감.
```

<details>
<summary><b>⚠ 트러블 로그</b> — 복합 정렬 키의 후행 컬럼만 필터하면서 "클러스터링했는데 왜 스캔량이 그대로냐" 고 묻게 됨.</summary>
<div markdown="1">

**예 —** BigQuery 에서 `CLUSTER BY visit_id, page` 로 만든 3TB 테이블에
`WHERE page = 'home.html'` 만 걸었더니 스캔 바이트가 파티션 전체와 거의 같게 나오고 비용도 그대로였음.
선행 컬럼 `visit_id` 가 술어에 없어 클러스터링 블록을 좁힐 수 없기 때문.
Delta 에서 `OPTIMIZE ... ZORDER BY` 를 한 번 돌리고 끝낸 경우도 비슷 —
이후 스트리밍으로 계속 append 된 파일들은 **정렬되지 않은 세그먼트** 로 남아 혜택을 못 받음.

**반대 함정 —** 그렇다고 정렬 키를 나중에 바꾸면 **테이블 전체 재정렬** 이 필요해,
수 TB 테이블에서는 재작성 비용이 절감분을 넘길 수 있음.

**권장 —** 정렬 키는 **실제 쿼리 술어 분포를 먼저 집계해서** 정할 것.
단독 필터가 자주 오는 컬럼이 둘 이상이면 사전식 `CLUSTER BY` 가 아니라 **Z-order** 를 쓰고,
Z-order·클러스터링은 **주기적 최적화 잡으로 스케줄링** 해 정렬 안 된 세그먼트가 쌓이지 않게 할 것.

</div>
</details>

---

## 3. 요약

챕터 8 의 앞 두 절은 **"데이터를 어떻게 배치해야 빠르고 싸게 읽히나"** 를 **나누기(파티셔닝)** 와
**묶고 정렬하기(레코드 조직화)** 두 단계로 다룸.

- **파티셔닝** — 처리할 데이터 볼륨 자체를 줄이는 첫 단계.
  #50 **Horizontal Partitioner** 는 **row 전체** 를 파티션 값별 위치로 옮기며, **저카디널리티 값**(이벤트 시각 등)에 좋은 후보.
  #51 **Vertical Partitioner** 는 **속성 수준** 에서 동작해 한 row 를 여러 조각으로 쪼개 다른 곳에 저장.
- **레코드 조직화** — 파티셔닝은 훌륭한 저장 최적화지만 **성(last name)·도시** 같은 **고카디널리티 값에는 안 통함**.
  이때 더 나은 접근이 #52 **Bucket** — 비슷한 row 여러 개를 **버킷** 이라는 컨테이너로 묶음.
  추가로 #53 **Sorter** 를 활용하면 **정렬된 데이터 위에서 더 빠른 처리** 가 가능.

| 패턴 | 카테고리 | 한 줄 요약 | 핵심 트레이드오프 |
|---|---|---|---|
| #50 Horizontal Partitioner | Partitioning | 파티셔닝 컬럼에 따라 row 들을 함께 저장 | 파티션 프루닝·멱등성 토대 / <br>파티션 세분성·스큐·키 변경 비용 |
| #51 Vertical Partitioner | Partitioning | 한 row 를 컬럼 그룹별 여러 파티션으로 분할 | 불변 속성 1회 저장·조각별 정책 / <br>도메인 분할·join 필요·프로듀서 부담 |
| #52 Bucket | Records Organization | 고카디널리티 레코드를 `hash%N` 으로 묶어 배치 | 버킷 프루닝·shuffle 없는 join / <br>스키마 불변(backfill)·key 직접 접근 불리 |
| #53 Sorter | Records Organization | 데이터 블록을 디스크에 정렬해 저장 | 데이터 스키핑으로 읽기↑ / <br>쓰기 정렬 오버헤드·복합 정렬 키 제약 |

```
[챕터 8 전반부 선택 가이드 — #50~#53]
──────────────────────────────────────────────────────────────────────
 데이터를 나눌 때
   row 전체를 값별로 나눔 · 저카디널리티     ─► #50 Horizontal Partitioner (날짜·국가)
   row 를 컬럼 그룹으로 쪼갬 · 불변 속성     ─► #51 Vertical Partitioner (mutable ↔ immutable)

 파티셔닝으로 안 될 때
   고카디널리티 컬럼이 술어에 자주 등장      ─► #52 Bucket (hash(key) % N)
   같은 컬럼으로 join 도 자주 함             ─► #52 Bucket (양쪽 동일 구성 → shuffle 제거)

 블록 단위로 더 좁히고 싶을 때
   특정 컬럼으로 필터·정렬이 대부분          ─► #53 Sorter (CLUSTER BY, 사전식)
   여러 컬럼을 제각각 단독으로 필터          ─► #53 Sorter (Z-order, 곡선 정렬)
──────────────────────────────────────────────────────────────────────
 ⚠ 파티션 키는 폭증(리스팅·small files)을, 버킷은 개수 고정(변경 시 backfill)을,
   정렬은 선행 컬럼 의존과 정렬 안 된 세그먼트를 각각 조심할 것.
```

**정리** — 이 네 패턴은 **"위치를 나누고(#50·#51) → 그 안에서 그룹을 묶고(#52) → 블록을 정렬한다(#53)"** 는
**점점 국소적으로 좁혀 가는 한 축**. 배타적 선택지가 아니라 **층층이 쌓아 쓰는 도구** 라는 점이 핵심.

한 줄로 관통하는 원칙은 **"쓰기에서의 수고를 읽기에서의 절감으로 바꾸는 거래"** —
#51 은 프로듀서에 분할 로직과 다중 write 를, #52 는 backfill 없이 못 바꾸는 스키마를,
#53 은 쓰기 시점의 정렬 비용을 각각 요구하고, 그 대가로 컨슈머 (조회하는 쪽)의 스캔량을 줄여 줌.
**어떤 축으로 얼마나 좁힐지는 실제 쿼리 술어 분포** 를 보고 정할 일.

> 데이터를 잘 배치했더라도 **읽기 경로에는 아직 최적화 여지가 남음**.
> 이어지는 절에서는 **메타데이터 계층 활용(#54 Metadata Enhancer)**, **비싼 연산의 1회 구체화(#55 Dataset Materializer)**,
> **비싼 리스팅 회피(#56 Manifest)** 와 **데이터 표현 방식(#57 Normalizer / #58 Denormalizer)** 이 이어짐.

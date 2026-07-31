스토리지 계층의 레이아웃을 정의할 때 핵심은 데이터셋에 쉽게 접근할 수 있도록 나누는 최선의 방법을 찾는 것이다. 이를 위해 데이터 표현 접근 방식과 스토리지 최적화 원칙을 이해하고, 수평적 구성과 수직적 구성을 담당하는 두 가지 파티셔닝 패턴을 활용해야 한다.

## 1\. 스토리지 레이아웃 사전 고려사항 및 데이터 표현 방식

### 스토리지 최적화를 위한 핵심 원칙

-   불필요한 데이터 관련 작업을 피하기 위해 메타데이터 계층을 활용한다.
-   비용이 많이 드는 작업은 한 번만 실행하고, 후속 독자(reader)들을 위해 이를 구체화한다.
-   비용이 많이 드는 목록 작업을 피하여 데이터 준비 단계를 단순화한다.

### 데이터 표현 접근 방식

-   **정규화기 접근 방식**: 데이터 일관성을 선호한다.
-   **비정규화기 접근 방식**: 더 나은 실행 시간을 위해 일관성을 절충한다.

## 2\. 수평 파티셔너 (Horizontal Partitioner)

구현의 단순성과 데이터 엔지니어링 초기부터 오랜 기간 누려온 인기 덕분에 가장 일반적으로 사용되는 패턴이다.

### (1) 문제 상황

이전 나흘(4일)의 롤링 집계를 계산하는 배치 잡을 구축하여 수개월간 정상 운영했으나, 저장소 계층에 데이터가 증가하면서 성능이 저하되었다. 주요 원인은 나흘보다 오래된 레코드를 무시하는 필터링 작업의 실행 시간이 증가했기 때문이다. 클러스터에 컴퓨팅 파워를 추가해 임시 조치했으나 비용이 증가하였다. 비용을 낮추면서 최신 데이터 처리 실행 시간을 줄일 수 있는 접근 방식이 필요하다.

### (2) 해결책

롤링 집계는 전체 데이터셋의 일부만 사용하는 증분 데이터 처리의 예이다. 수평 파티셔너 패턴은 분산 키(distribution key)라고도 불리는 파티셔닝 속성을 식별하여, 물리적으로 격리된 저장 공간에 데이터셋을 나누어 저장함으로써 실행 시간과 비용의 균형을 맞춘다.

#### 파티셔닝 키의 유형

1.  **시간 기반 파티션**
    -   **잡 실행 컨텍스트**: 파티셔닝이 작업의 실행 시간에 의존하며, 파티션 값은 모든 레코드에서 동일하다 (예: 2024-12-31 실행 잡의 레코드는 모두 해당 일자 파티션에 위치).
    -   **데이터셋 (이벤트 시간)**: 이벤트 시간 관점에서 추론한다. 지연 데이터(Late Data) 현상으로 인해 파티션된 데이터셋 내에 다른 파티션의 값이 포함될 수 있다.
2.  **비즈니스 키 및 중첩 파티셔닝**
    -   고객 ID, 파트너 ID, 지리적 지역 등 비즈니스 키를 사용할 수 있다.
    -   시간 기반 속성과 비즈니스 기반 속성을 결합하여 중첩 파티셔닝 스키마를 구성할 수 있다.
    -   _예제 structure_: visits/2024/05/france, india, poland, usa

#### 파티션 설정 및 메타데이터 관리

-   **선언적 방식**: Databricks나 GCP BigQuery의 CREATE TABLE ... PARTITIONED BY 문처럼 프로듀서가 파티션 값 정의를 몰라도 수집 시 처리된다.
-   **동적 방식**: Apache Spark의 partitionBy 메서드(기존/계산된 컬럼 활용) 또는 Apache Kafka의 사용자 정의 파티셔너 클래스를 활용한다.
-   **파티션 메타데이터**: 일부 데이터 스토어는 마지막 갱신 시간, 레코드 수, 생성 시간 등을 관리한다.
    -   GCP BigQuery: INFORMATION\_SCHEMA.PARTITIONS 뷰
    -   Databricks: DESCRIBE TABLE EXTENDED
    -   Apache Iceberg: SELECT \* FROM a\_catalog.a\_namespace.a\_table.partitions
-   **멱등성 활용**: 수평 파티셔닝은 빠른 메타데이터 정리기 패턴 등을 통해 멱등성 파이프라인 구축의 핵심 구성 요소 역할을 한다.

### (3) 결과 및 단점 (Trade-offs)

수평 파티셔너의 가장 큰 단점은 정적 특성이다.

-   **세분화와 메타데이터 오버헤드**: 파티션은 동일한 값을 공유하는 물리적 위치이므로 파티션이 너무 많으면 데이터베이스에 부정적 영향을 미친다. (예: 매일 100만 고유 유저 이름으로 파티셔닝 시 100만 개 파티션이 생성되어 느린 파티션 목록 작업과 작은 파일 읽기 문제가 발생함). 따라서 카디널리티가 낮은 속성(시간/일 단위 이벤트 시간)을 사용하는 것이 좋은 경험 법칙이며, IoT 기기 ID 같은 고카디널리티 속성은 버킷 디자인 패턴을 사용하는 것이 적합하다.
-   **스큐 (Data Skew)**: 불균형한 파티션은 지연의 원인이 된다. 마이크로배치 스트림 처리 모델에서는 블로킹 방식으로 동작하므로 불균형한 파티션이 완료될 때까지 전체 마이크로배치가 대기해야 한다. 이를 완화하기 위해 스큐된 파티션의 추가 레코드를 별도의 버퍼에 저장하고 다음 마이크로배치에서 처리하는 **백프레셔 메커니즘**을 적용할 수 있다. 이는 스큐 파티션의 전달 지연은 늘리지만 타 작업의 실시간성을 보장한다.
-   **가변성 (Mutability)**: 파티션 키 변경 시 모든 데이터를 이동해야 하므로 비용과 시간이 많이 소요된다. Apache Iceberg 등은 메타데이터 계층에서 파티셔닝 스키마 변경(파티션 진화)을 지원하여 기존 파일 변경 없이 파티션 진화 이후 생성된 레코드부터 새 구성을 적용한다.

> **NOTE: 수직 파티셔닝 vs 샤딩** 샤딩(Sharding)은 데이터셋을 여러 장비로 나누는 것으로 물리적인 데이터 분할을 포함한다. 수평 파티셔닝은 장비 간 데이터 이동을 요구하지 않으며, 샤딩은 하드웨어 계층에 기반한 수평 파티셔닝의 특별한 유형이다.

### (4) 예제 코드

#### Apache Spark (파티셔닝 컬럼 생성)

```
partitioned_users = (input_users
    .withColumn('year', functions.year('change_date'))
    .withColumn('month', functions.month('change_date'))
    .withColumn('day', functions.day('change_date'))
    .withColumn('hour', functions.hour('change_date')))

(partitioned_users.write.mode('overwrite').format('delta')
    .partitionBy('year', 'month', 'day', 'hour').save(output_dir))
```

#### Apache Kafka (사용자 정의 파티셔너)

_(주의: 모든 코드는 복잡성을 증가시키므로 단순함을 유지하고 기본 파티셔너 사용을 권장함)_

```
public class RangePartitioner implements Partitioner {
    private static final int DEFAULT_PARTITION = 1;
    private final static Map<String, Integer> RANGES_PER_PARTITIONS = new HashMap<>();
    static {
        RANGES_PER_PARTITIONS.put("A", 0);
        RANGES_PER_PARTITIONS.put("B", 0);
    }

    @Override
    public int partition(String topic, Object key, byte[] keyBytes,
                         Object value, byte[] valueBytes, Cluster cluster) {
        String keyAsString = key.toString();
        return RANGES_PER_PARTITIONS.getOrDefault(keyAsString, DEFAULT_PARTITION);
    }
}

// 프로듀서 설정
Properties props = new Properties();
props.put("partitioner.class", "com.waitingforcode.RangePartitioner");
```

#### PostgreSQL (날짜 시간 범위 파티셔닝)

```
CREATE TABLE visits_all (
    visit_id CHAR(36) NOT NULL,
    event_time TIMESTAMP NOT NULL,
    user_id TEXT NOT NULL,
    page VARCHAR(20) NULL,
    PRIMARY KEY(visit_id, event_time)
) PARTITION BY RANGE(event_time);

CREATE TABLE visits_all_20231124 PARTITION OF visits_all
FOR VALUES FROM('2023-11-24 00:00:00') TO ('2023-11-24 23:59:59');

CREATE TABLE visits_all_20231125 PARTITION OF visits_all
FOR VALUES FROM('2023-11-25 00:00:00') TO ('2023-11-25 23:59:59');
```

## 3\. 수직 파티셔너 (Vertical Partitioner)

매번 전체 레코드를 처리하는 수평 파티셔너와 달리, 각 레코드를 분할하여 별도의 부분을 다른 테이블이나 파일 위치에 작성한다.

> **NOTE: 스토리지와 보안** 7장의 수직 파티셔너가 보안에 특화되었다면, 8장의 수직 파티셔너는 데이터 스토리지에 특화된 패턴이다.

### (1) 문제 상황

웹사이트 방문 추적 파이프라인에서 방문 데이터셋은 각 방문마다 변경되는 가변 속성(방문 시간, 방문 페이지 등)과 방문 전반에 동일하게 유지되는 불변 속성(IP 주소 등)을 가진다. 불변 정보의 중복을 피하고 방문마다 한 번만 저장하는 구조가 필요하다.

### (2) 해결책

속성을 가변 그룹과 불변 그룹으로 분류하고, 분할된 레코드를 재결합할 공유 키(방문 ID / visit\_id)를 식별한다. 데이터 처리 작업은 레코드의 그룹화된 속성을 전용 위치(테이블 또는 디렉터리)에 저장한다.

#### 차이점 및 장점

-   **차이점**: 수평 파티셔닝은 전체 레코드를 다른 위치로 이동시키지만, 수직 파티셔닝은 레코드 내부를 분할하여 다른 위치에 분리하여 쓴다.
-   **장점**: 저장 비용 최적화뿐만 아니라 레코드가 분할되었기 때문에 그룹별로 서로 다른 데이터 보존 정책이나 데이터 접근 정책을 쉽게 적용할 수 있는 유연성을 제공한다.

### (3) 결과 및 단점 (Trade-offs)

-   **도메인 분할**: 논리적으로 관련된 속성들이 서로 다른 장소에 저장되므로 전체 파악이 어려워진다. 따라서 최종 사용자를 위한 양질의 문서화 지원이 필수적이다.
-   **쿼리 복잡성**: 전체 그림을 파악하기가 수평 파티셔닝보다 어렵다. 이를 완화하기 위해 수직으로 파티션된 엔티티의 모든 테이블을 결합하는 뷰(View)를 노출할 수 있다 (데이터셋 구체화 패턴 활용).
-   **데이터 프로듀서 부담**: 프로듀서가 단순히 레코드를 쓸 수 없고 레코드 분할 로직을 직접 구현해야 하며, 이로 인해 여러 번의 쓰기 작업과 높은 네트워크 통신 비용이 발생한다.

### (4) 예제 코드

#### Apache Spark (수직 파티셔닝)

입력 데이터셋이 두 번 읽히지 않도록 persist() 및 unpersist()를 명시적으로 호출한다.

```
visits = spark_session.read.schema(visit_schema).json(input_location)
visits.persist()

# 가변 속성 저장
visits_without_user_technical_context = (visits.drop('user_id')
    .withColumn('context', F.col('context').dropFields('user'))
    .withColumn('context', F.col('context').dropFields('technical')))
visits_without_user_technical_context.write.format('delta').save(output_dir)

# 유저 관련 불변 속성 저장
(visits.selectExpr('visit_id', 'context.user.*', 'user_id').dropDuplicates()
    .write.format('delta').save(get_delta_users_table_dir()))

# 기술 컨텍스트 불변 속성 저장
(visits.selectExpr('visit_id', 'context.technical.*').dropDuplicates()
    .write.format('delta').save(get_delta_technical_table_dir()))

visits.unpersist()
```

#### INSERT INTO ... SELECT FROM 방식

기존 테이블에서 특정 컨텍스트 속성만 추출하여 수직 파티션된 테이블에 삽입한다.

```
INSERT INTO dedp.technical (visit_id, browser, browser_version, ...)
(SELECT DISTINCT visit_id, context->'technical'->>'browser',
        context->'technical'->>'browser_version', ...
FROM dedp.visits_all);
```

#### CTAS (CREATE TABLE AS SELECT) 방식

SELECT 문을 사용하여 수직으로 파티션된 테이블을 직접 생성한다.

```
CREATE TABLE dedp.technical_select AS (SELECT DISTINCT
    visit_id, context->'technical'->>'browser' AS browser,
    context->'technical'->>'browser_version' AS browser_version, ...
FROM dedp.visits_all);
```

## 레코드 구성 (Record Organization) 개요

파티셔닝은 데이터 구성의 첫 단계이지만 레코드 전체나 일부를 다른 위치로 이동시키는 기초적인 접근 방식이다. 카디널리티가 높은 속성은 수평 파티셔닝에 적합하지 않다.

레코드 구성 패턴 범주는 레코드 코로케이션(co-location)에 대한 스마트한 최적화를 적용하여 수평 파티셔너의 높은 카디널리티 문제를 해결한다.

## 4\. 버킷 (Bucket)

고유 사용자 ID와 같이 높은 카디널리티를 가진 컬럼에 대한 접근 성능을 개선하는 패턴이다. 동일한 저장 공간에 레코드를 코로케이션하는 대신, 레코드 그룹을 코로케이션한다.

### (1) 문제 상황

-   데이터셋의 쿼리 술어(predicate)로 자주 사용되는 비즈니스 속성이 존재함.
-   해당 속성을 파티셔닝 컬럼으로 사용하고자 하나, 카디널리티가 너무 높아 과도한 파티션이 생성되고 데이터 스토어의 메타데이터 제한에 도달함.
-   작업의 80%가 해당 속성에 의존하므로 저장을 최적화해야 함.

### (2) 해결책

표면적으로는 전용 위치에 레코드를 저장하지만, 수평 파티셔너와 달리 동일한 저장 영역에 서로 다른 값들을 코로케이션한다.

#### 구현 단계

1.  **버킷 컬럼 정의**: 데이터 분석 단계에서 버킷팅에 사용할 컬럼을 선정한다. 수평/수직 파티션된 데이터셋의 경우, 이 속성은 보조 그룹화 키 집합으로 간주된다.
2.  **버킷 수 설정**: 카디널리티에 맞춰 버킷 수를 결정한다. 높은 버킷 수는 '많은 수의 작은 버킷'을, 낮은 버킷 수는 '적은 수의 큰 버킷'을 의미한다.
3.  **모듈러 해싱 적용**: 각 키의 버킷 번호는 hash(key) % buckets number 공식을 통해 계산되어 레코드가 그룹화된다.

#### 버킷 패턴이 제공하는 2가지 최적화 기법

-   **버킷 프루닝 (Bucket Pruning)**: 쿼리에서 버킷 컬럼이 술어로 사용될 때, 쿼리 실행 엔진이 버킷팅 알고리즘을 직접 적용하여 필요한 키가 없는 버킷을 모두 건너뛴다. 필터링 성능이 향상된다.
-   **네트워크 교환(셔플) 제거**: 조인(JOIN) 양쪽 테이블이 동일한 버킷팅 구성을 사용하는 경우 적용된다. 상관된 레코드가 동일한 조인 프로세스에 직접 적재되므로, 네트워크 셔플 없이 분산 조인 및 집계를 수행할 수 있다.

#### 기술적 배경

역사적으로 아파치 하이브(Apache Hive)를 통해 보급되었으며, 현재는 아파치 스파크(Apache Spark), AWS 아테나(Athena) 등 최신 데이터 솔루션에 통합되어 있다.

### (3) 결과 및 한계 (Trade-offs)

-   **가변성 (Mutability)**: 버킷팅 스키마는 불변이다. 컬럼이나 버킷 크기를 변경하려면 데이터셋 전체를 백필링(backfilling)해야 하는 고비용 작업이 필요하다.
-   **버킷 크기 결정의 어려움**: 미래 데이터 볼륨을 예측해 적절한 버킷 크기를 설정하기 어렵다. 현재 볼륨에 맞추면 미래에 버킷이 커지고, 미래 볼륨을 과도하게 예측하면 너무 많은 버킷이 생성되는 함정이 존재한다.

### (4) 예제 코드

#### AWS 아테나 (Athena)

아테나는 S3 데이터에 기존 버킷팅 로직만 적용하는 논리적 쿼리 서비스이므로, 버킷팅된 테이블에 INSERT INTO 쿼리를 직접 실행하면 오류가 발생한다.

```
CREATE EXTERNAL TABLE visits (...) ...
CLUSTERED BY (`user_id`) INTO 50 BUCKETS
TBLPROPERTIES ('bucketing_format' = 'spark')
```

#### 아파치 스파크 (Apache Spark)

bucketBy 함수를 통해 모듈 기반 알고리즘을 적용한 버킷형 테이블을 생성한다.

```
input_dataset.write.bucketBy(50, 'user_id').saveAsTable(table_name)
```

## 5\. 정렬기 (Sorter)

레코드 그룹을 코로케이션하는 버킷 외에, 데이터의 **저장 순서**를 활용하여 쿼리와 관련 없는 데이터 블록을 제거하는 최적화 기법이다.

### (1) 문제 상황

-   주간 테이블 저장 및 멱등성 유지 관리 방식(빠른 메타데이터 정리기 패턴)을 사용 중이나, 쿼리 실행 시간이 개선되지 않음.
-   멱등성 전략을 유지하면서 쿼리 지연을 줄여야함.
-   대부분의 쿼리가 이벤트 시간 컬럼 등으로 필터링되거나 정렬되는 패턴을 보임.

### (2) 해결책

자주 정렬/필터링되는 컬럼을 정렬 컬럼으로 식별하고, 테이블 생성 시 이를 선언한다. 데이터베이스는 정렬 선언에 따라 작성된 레코드를 순서대로 구성한다.

-   **메타데이터 건너뛰기**: 저장소가 이미 정렬되어 있으므로, 정렬 컬럼 대상 쿼리는 메타데이터(파일별 최소/최대 범위 등)를 확인하여 관련 없는 데이터 블록 전체를 읽지 않고 건너뛸 수 있다.

#### 곡선 정렬 (Z-order)

기존의 단일 축 기준 상-하 정렬(사전식 정렬)의 변형으로, x차원 공간에서 레코드들을 다차원적으로 코로케이션한다.

-   **블록 읽기 감소**: 사전식 정렬 대비 여러 컬럼에 대해 훨씬 적은 수의 데이터 블록만 읽고 처리할 수 있다 (예: 특정 두 컬럼 검색 시 사전식 정렬이 9개 블록을 읽을 때 Z-order는 7개만 읽음).
-   **지원 기술**: 아파치 아이스버그(Apache Iceberg), 델타 레이크(Delta Lake)가 기본 구현하고 있으며, 아마존 레드시프트(Interleaved Sort Key), GCP 빅쿼리(Clustered Table), 스노우플레이크(Snowflake) 등에서도 유사 개념을 활용한다.

> **NOTE: 정렬 vs 클러스터링**
> 
> Z-order는 관련 레코드를 동일 파일에 모으므로 클러스터링으로도 불린다. 그러나 디스크 상에서 데이터를 정렬하여 이를 달성하므로 본질적으로 정렬기 패턴의 일종으로 분류된다.

### (3) 결과 및 한계 (Trade-offs)

-   **작성자(Writer) 성능 저하 및 정렬되지 않은 세그먼트**: 레코드 정렬 작업으로 인해 쓰기 시 추가 오버헤드가 발생한다. 즉각 정렬되지 않은 블록이 생길 수 있으며, 이를 처리하기 위해 정렬 작업을 쓰기 프로세스 내에 통합하거나 외부에서 정렬 작업을 예약(scheduling)해야 한다.
-   **복합 정렬 키의 선두 컬럼 제약**: 사전식 정렬에서 복합 키(예: visit\_time, page)를 사용할 경우, 쿼리가 반드시 선두 컬럼(visit\_time)을 포함해야만 정렬 혜택을 받는다. 후속 컬럼(page)만으로 쿼리하면 선두 컬럼이 없으므로 대다수 블록을 스캔하게 된다. (Z-order는 이 한계를 완화할 수 있다.)
-   **가변성 (Mutability)**: 생성 후 정렬 키를 변경하려면 전체 테이블을 다시 정렬해야 하므로 높은 비용이 발생한다.

### (4) 예제 코드

#### GCP 빅쿼리 (BigQuery)

파티셔닝과 클러스터링을 함께 적용하여 정렬기 패턴을 구현한다.

```
CREATE TABLE `dedp.visits.raw_visits`
PARTITION BY DATE(event_time)
CLUSTER BY visit_id, page
```

(단, 빅쿼리의 클러스터 테이블도 visit\_id 없이 page 컬럼으로만 필터링하는 쿼리에는 최적화 효과가 제한적이다.)

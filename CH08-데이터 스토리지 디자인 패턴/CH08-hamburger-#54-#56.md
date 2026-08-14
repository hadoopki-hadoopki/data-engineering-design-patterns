## **패턴 #54: 메타데이터 강화기**

데이터를 쿼리 엔진에 적재하기 전에 관련 없는 데이터 파일이나 블록을 건너뛰어(Pruning) 조회 성능을 최적화하고 비용을 줄이는 패턴이다.

### 1\. 문제 상황

-   기존에 JSON 데이터셋을 이벤트 시간 기준으로 수평 파티셔닝하여 배치 작업 속도를 개선함.
-   새로 고용된 데이터 분석가들이 동일하게 파티션된 데이터셋을 사용하지만, 실제로는 파티션 내에서도 극히 일부분(작은 부분집합)의 레코드만 다룸.
-   사용량 기반 과금형 쿼리 서비스를 이용하는 환경에서, 쿼리가 항상 전체 파티션 데이터셋을 엔진에 적재한 뒤 필터링 로직을 적용함.
-   이로 인해 쿼리 실행 지연과 클라우드 비용 증가 문제가 발생함.
-   **목적**: 데이터셋을 쿼리 엔진에 적재하기 전에 필터링 로직을 먼저 적용할 수 있는 구조가 필요함.

### 2\. 해결책

처리를 시작하기 전, 파일이나 데이터베이스에 저장된 레코드 통계(최솟값, 최댓값, NULL 개수 등)를 수집 및 유지하여 관련 없는 파일을 미리 건너뛴다.

#### (1) 컬럼형 파일 형식 (Apache Parquet)

-   컬럼형 파일 형식은 데이터 파일 끝에 추가 메타데이터가 담긴 푸터(Footer)를 포함한다.
-   데이터 생성 시 푸터에 컬럼별 통계(예: min, max, nulls)가 자동으로 계산되어 저장된다.
-   쿼리 실행 엔진은 WHERE age > 50과 같은 조건이 들어올 때, 실제 데이터 블록을 열기 전에 푸터 메타데이터만 확인하여 조건에 해당하지 않는 파일을 스캔 대상에서 즉시 제외한다.
-   푸터의 크기는 데이터 블록보다 훨씬 작기 때문에 필터링에 걸리는 시간이 대폭 단축된다.

#### (2) 데이터 레이크하우스 테이블 형식 (Delta Lake, Apache Iceberg, Apache Hudi)

-   아파치 파케이 파일 자체의 메타데이터 외에도, 커밋 로그(Commit Log)에 레코드 수, 컬럼별 최솟값/최댓값, NULL 개수 등의 추가 메타데이터를 저장한다.
-   이를 통해 존재하지 않는 값 필터링이나 요소 개수 집계 쿼리를 매우 빠르게 처리한다.

#### (3) 관계형 데이터베이스 및 데이터 웨어하우스 (RDBMS / DW)

-   쿼리 플래너가 가장 효율적인 실행 계획을 작성할 수 있도록 별도의 통계 테이블을 유지 및 활용한다.

### 3\. 결과 및 단점 (Trade-offs)

-   **쓰기 시점 오버헤드**: 파케이 같은 컬럼형 파일을 작성할 때 컬럼별 통계를 실시간으로 계산해야 하므로 쓰기 처리 시간이 약간 증가할 수 있다.
-   **오래된 통계 (Outdated Stats) 문제**: RDBMS/DW에서는 데이터가 변경되어도 통계가 즉시 자동 갱신되지 않고 특정 수정 임계값에 도달해야만 갱신될 수 있다. 시간이 지나며 통계가 낙후되면 쿼리 플래너가 비효율적인 실행 계획을 수립할 수 있다.
-   **수동 새로 고침 오버헤드**: 통계 낙후를 완화하기 위해 ANALYZE TABLE과 같은 수동 새로 고침 명령을 실행해야 하며, 이는 임시적인 읽기 오버헤드를 발생시킨다.

### 4\. 예제 코드

(1) Apache Spark에서 Parquet 작성

```
input_dataset.write.mode('overwrite').parquet(path=get_parquet_dir())
```

(2) Docker를 활용한 Parquet 메타데이터 분석

```
docker run --rm -v "./output-parquet:/tmp/parquet" \
  hangxie/parquet-tools:v1.20.7 meta \
  /tmp/parquet/part-00001-3c52ae6f-aeea-4364-aac3-7fc69d63e898-c000.snappy.parquet
```

-   출력된 메타데이터 통계 예시 (Id 컬럼):

```
"NumRowGroups": 1, 
{
  "PathInSchema": ["Id"], 
  "Type": "BYTE_ARRAY",
  "Encodings": ["PLAIN", "RLE", "BIT_PACKED"], 
  "CompressedSize": 180463,
  "UncompressedSize": 200035, 
  "NumValues": 5000,
  "NullCount": 0, 
  "MaxValue": "fffbe4f8-8d88-43d2-a9a5-54bf536de75b",
  "MinValue": "0018e1dc-1b80-4410-92f6-5261d2dadf35",
  "CompressionCodec": "SNAPPY"
}
```

(3) Delta Lake 커밋 로그 메타데이터 예시

```
{
  "commitInfo": {
    "timestamp": 1716954694590,
    "operation": "WRITE",
    "operationMetrics": {
      "numFiles": "1",
      "numOutputRows": "6100",
      "numOutputBytes": "50437"
    }
  }
}
{
  "add": {
    "path": "part-...-c000.snappy.parquet",
    "size": 50437,
    "stats": "{\"numRecords\":6100,\"minValues\":{\"type\":\"galaxy\",\"full_name\":\"APPLE iPhone 11 (White, 64 GB)\",\"version\":\"Android 10\"},\"maxValues\":{\"type\":\"mac\",\"full_name\":\"Yoga 7i (14\\\" Intel) 2 in 1 Lapto \",\"version\":\"v17169535721658688\"},\"nullCount\":{\"type\":0,\"full_name\":0,\"version\":0}}"
  }
}
```

## **패턴 #55: 데이터셋 구체화기** 

복잡한 연산(셔플, CPU 집약적 변환, 복수 테이블 JOIN 등)이 포함된 반복 쿼리의 결과를 미리 계산하여 구체화된 테이블이나 뷰로 저장함으로써 접근 성능을 대폭 높이는 패턴이다.

### 1\. 문제 상황

-   지난 3주간의 데이터를 얻기 위해 여러 파티션 테이블을 매번 결합하여 쿼리해야 하는 복잡한 프로세스가 존재함.
-   단순 일반 뷰(View)를 생성해 제공했으나, 일반 뷰는 호출될 때마다 기본 SQL 쿼리를 매번 다시 재실행하므로 조회 지연 시간이 줄어들지 않고 사용자 불만이 지속됨.

### 2\. 해결책

미리 쿼리(SELECT, JOIN, UNION 등)를 통해 연산된 결과를 **구체화된 뷰(Materialized View)** 또는 **구체화된 테이블(Materialized Table)** 형태로 물리적으로 저장한다.

#### (1) 구체화된 뷰 (Materialized View)

-   Amazon Redshift (AUTO REFRESH YES), GCP BigQuery, Databricks, Snowflake 등 대부분의 현대 DW 솔루션에서 지원한다.
-   자동 새로 고침(Auto Refresh)을 지원하지만, 기본 테이블 변경 직후 즉시 갱신되지는 않으며 데이터베이스의 현재 워크로드나 새로 고칠 데이터 크기에 따라 실행 시점이 결정되므로 예측이 어려울 수 있다.

#### (2) 구체화된 테이블 (Materialized Table)

-   자동 새로 고침 기능이 없어 엔지니어가 직접 새로 고침 로직을 구현하고 관리해야 한다.
-   하지만 구체화된 뷰에서는 적용하기 어려운 파티셔닝, 버킷팅, 정렬(Sort) 등의 저장소 최적화 기법을 직접 적용할 수 있어 한층 더 뛰어난 운영 유연성을 제공한다.

### 3\. 결과 및 고려사항 (Trade-offs)

-   **새로 고침 비용 (Refresh Cost)**: 뷰를 새로 고칠 때마다 원본 생성 쿼리가 재실행되므로 계산 자원이 많이 소모된다.
    -   **대안 (증분 새로 고침)**: 전체를 다시 계산하지 않고 최신 변경 사항/신규 레코드만 통합하는 **증분 새로 고침(Incremental Refresh)** 기법을 적용할 수 있다. (Databricks, BigQuery 등에서 지원하나 모든 SQL 연산을 지원하지는 않음)
-   **데이터 접근 제어 한계**: 여러 원본 테이블이 결합된 형태이므로, 개별 테이블에 적용되던 일관된 데이터 관리가 어려워질 수 있다. 접근 제어를 계속 거부하거나 세밀한 접근자 패턴을 구현해야 한다.
-   **스토리지 오버헤드**: 데이터를 중복 저장하므로 저장 용량 비용이 늘어난다. 용량이 부담된다면 데이터의 일부만 구체화하고 나머지는 재계산하는 **혼합 구현(Hybrid)** 방식을 선택할 수 있다.

### 4\. 예제 코드

(1) GCP BigQuery: 자동 새로 고침 구체화된 뷰

```
REFRESH MATERIALIZED VIEW dedp.windowed_visits WITH DATA;
```

(3) 데이터셋 구체화기 패턴의 증분 버전 (MERGE 방식)  
입력 테이블의 작성 시간 컬럼(insertion\_time)을 기반으로 이전 실행 이후 추가된 신규 레코드만 쿼리하여 기존 데이터셋과 병합(MERGE)한다. (증분 로더 + 병합기 패턴 결합)

```
MERGE INTO dedp.visits_counter AS target
USING (
    -- 이전 실행 시점(2024-11-09T03:27:32) 이후에 삽입된 데이터만 조회
    SELECT user_id, COUNT(*) AS visits 
    FROM dedp.visits
    WHERE insertion_time > '2024-11-09T03:27:32' 
    GROUP BY user_id
) AS input
ON target.user_id = input.user_id
WHEN MATCHED THEN 
    UPDATE SET count = count + input.visits
WHEN NOT MATCHED THEN 
    INSERT (user_id, count) VALUES (input.user_id, input.visits);
```

## **패턴 #56: 매니페스트**

읽기 접근 성능은 데이터 목록 작성(Listing) 작업과 밀접한 관련이 있다. 객체 스토리지(Object Storage) 환경에서는 목록화 작업 시 수많은 API 호출이 발생하므로 속도가 저하될 수 있다. 매니페스트 패턴은 이러한 파일 목록화 작업을 사전 기록 및 단순화하여 성능을 최적화하는 패턴이다.

### 1\. 문제 상황

-   객체 스토리지에 아파치 파케이(Apache Parquet) 데이터셋을 생성하여 배치 작업의 실행 시간과 클라우드 요금을 단축함.
-   해당 파케이 데이터셋을 데이터 분석가 팀이 사용할 수 있도록 데이터 웨어하우스(DW) 계층으로 노출하려 함.
-   그러나 테스트 진행 시 객체 스토리지에서 적재할 파일들을 목록화(Listing)하는 단계에서 많은 시간과 비용이 소요되어 전체 쿼리 실행 속도가 크게 저하됨.

### 2\. 해결책

반복되는 파일 목록화 문제를 해결하기 위해, 파일을 한 번만 목록화하거나 데이터 프로듀서(Data Producer)가 생성 시점에 파일명을 미리 메타데이터로 기록하여 읽기 시점의 목록화 작업을 제거한다.

#### (1) 테이블 파일 형식 (Delta Lake, Apache Iceberg, Apache Hudi)

-   주어진 트랜잭션 내에서 생성된 파일 목록을 메타데이터 위치의 커밋 로그(Commit Log)에 작성함.
-   데이터를 읽는 독자(Reader)는 스토리지 전체를 목록화하지 않고, 커밋/로그 파일(매니페스트 파일 역할)만 읽어서 필요한 파일 목록을 빠르게 확보함.

#### (2) 수동 생성 매니페스트

-   자동으로 관리되는 매니페스트 대안으로 사전 목록 작업이 필요할 때 수동으로 생성함.
-   여러 독자가 동일한 데이터셋에서 작업하는 팬아웃(Fan-out) 패턴 환경에서 유용함.

#### (3) 쓰기 및 데이터 로딩 시 활용

-   **Amazon Redshift**: COPY 명령어를 실행할 때 로드할 파일의 전용 목록이 담긴 매니페스트 파일을 활용함.
-   **GCP Storage Transfer Service**: 다른 클라우드 스토리지에서 GCS로 파일을 복사할 때 매니페스트 목록에 의존함.

### 3\. 결과 및 고려사항 (Trade-offs)

-   **복잡성 (Complexity)**:
    -   파이프라인에 매니페스트 생성 단계를 추가해야 하므로 실행 플로우가 약간 복잡해짐.
    -   하지만 최근 작성된 파일을 목록화하는 간단한 작업이므로, 느리고 예측할 수 없는 스토리지 목록화 작업을 반복하는 것보다 관리상 유리함.
-   **파일 크기 (Size)**:
    -   작은 파일이 많거나 연속적인 스트리밍 작업 환경에서는 매니페스트 파일 용량이 몇 기가바이트(GB) 수준으로 커질 수 있음.
    -   따라서 최대 크기 제한이나 보존(Retention) 구성을 적용해야 함.
    -   _(사례)_ 초기 아파치 스파크 구조적 스트리밍(Spark Structured Streaming)에서는 새 파일 생성 시마다 매니페스트에 경로를 추가하다가 파일이 너무 커져 작업을 재시작하지 못하는 문제가 발생했었으며, 이후 수정됨.

### 4\. 예제 코드

(1) Delta Lake 매니페스트 생성 및 BigQuery 외부 테이블 연동

```
# 델타 레이크 테이블에서 매니페스트 파일 생성 (generate 함수 호출)
devices_table = DeltaTable.forPath(spark_session, DemoConfiguration.DEVICES_TABLE)
devices_table.generate('symlink_format_manifest')
```

```
-- 빅쿼리(BigQuery)에서 생성된 매니페스트를 참조하는 외부 테이블 생성
CREATE EXTERNAL TABLE IF NOT EXISTS `dedp.visits.devices`
...
OPTIONS (
  hive_partition_uri_prefix = "gc://devices",
  uris = ['gc://devices/_symlink_format_manifest/*/manifest'],
  file_set_spec_type = 'NEW_LINE_DELIMITED_MANIFEST',
  format="PARQUET"
);
```

(2) Amazon Redshift COPY 명령 및 매니페스트 구조

```
-- Amazon Redshift 데이터 로딩
COPY customer
FROM 's3://devices/manifest_20250601_1031'
...
```

```
/* manifest_20250601_1031 매니페스트 파일 내부 예시 */
MANIFEST;
# manifest_20250601_1031
{"entries": [
  {"url":"s3://devices/dataset_1", "mandatory":true},
  {"url":"s3://devices/dataset_2", "mandatory":true}
]}
```

데이터 스토리지는 단순히 물리적 구성을 최적화하는 것 외에도, 어떤 속성들을 함께 저장하고 어떤 구조의 테이블로 표현할지 결정하는 **데이터 표현** 방식이 매우 중요하다.

## **패턴 #57: 정규화기**

정규화기 패턴은 정보의 중복을 제거(디커플링)하여 데이터셋의 일관성을 유지하고 저장 공간 및 갱신 효율성을 높이는 패턴이다.

### 1\. 문제 상황

-   visits 테이블에 방문 시간, 방문 페이지 같은 이벤트성 데이터뿐만 아니라 장치 이름, OS 이름, OS 버전 등 불변 속성이 함께 저장됨.
-   각 방문 레코드마다 동일한 불변 속성이 반복 저장되어 저장 공간이 낭비됨.
-   해당 속성이 변경되거나 수정될 때 전체 레코드를 수정해야 하므로 갱신 연산 속도가 저하됨.

### 2\. 해결책

분리되는 각 정보를 단 한 번만 표현하여 데이터 중복을 줄인다. 대표적인 구현 방법으로 정규형(Normal Form, NF)과 스노우플레이크 스키마(Snowflake Schema)가 있다.

#### (1) 정규화 상위 수준 설계 3단계

1.  **비즈니스 엔티티 정의**: 데이터 모델에 포함될 용어 목록 작성 (visits, devices, browsers, link referrals 등).
2.  **비즈니스 엔티티 설명**: 각 엔티티의 속성 정의 (예: 브라우저 엔티티 $\\rightarrow$ 브라우저 이름, 버전).
3.  **비즈니스 엔티티 간 관계 정의**: 엔티티 간의 의존성 정의 (예: 방문은 브라우저 가용성에 의존하고, 브라우저는 OS 가용성에 의존).

#### (2) 정규형 (NF) 기반 접근법

쓰기 연산이 빈번한 트랜잭션 워크로드에서 중복을 줄이고 데이터 품질을 높이기 위해 아래 규칙을 준수한다.

-   **제1정규형 (1NF)**: 모든 컬럼은 반복되지 않는 원자적(Atomic) 값을 가져야 하며, 기본키(PK)로 각 레코드를 고유하게 식별할 수 있어야 함.
-   **제2정규형 (2NF)**: 제1정규형을 만족하고, 기본키가 아닌 모든 속성은 기본키 전체에만 의존해야 함.
-   **제3정규형 (3NF)**: 제2정규형을 만족하고, 기본키가 아닌 속성들 간의 이행적 의존성(Transitive Dependency)이 없어야 함.

#### (3) 정규형 위반 및 해결 예시

1.  **제1정규형 위반**: comments 컬럼에 배열 형식으로 여러 값이 중복 저장됨.
    -   _해결_: 댓글 전용 games\_comments 테이블로 분리 추출.

| Name (PK) | Comments |
| --- | --- |
| Puzzle Tour | \["...", "..."\] |
| Runner | \["..."\] |

2. **제2정규형 위반**: 복합 기본키(Name, Platform)를 가진 구조에서 Platform language 속성이 Platform에만 부분 의존함.

-   _해결_: 플랫폼과 플랫폼 언어를 저장하는 별도 테이블 생성.

| Name (PK) | Platform (PK) | Release year | Platform language |
| --- | --- | --- | --- |
| Puzzle Tour | iOS | 2023 | Swift |
| Puzzle Tour | Android | 2024 | Kotlin |
| Runner | Android | 2024 | Kotlin |

3. **제3정규형 위반**: 기본키(Name)가 아닌 Studio country 속성이 다른 일반 속성인 Studio에 의존함(이행적 의존성).

-   _해결_: studios 테이블을 신설하여 스튜디오 관련 정보를 분리.

| Name (PK) | Studio | Studio country |
| --- | --- | --- |
| Puzzle Tour | Studio A | Italy |
| Runner | Studio B | Portugal |

#### (4) 스노우플레이크 모델 (Snowflake Model)

-   분석 워크로드에서 사용되는 차원 모델(Dimensional Model)의 확장 형태임.
-   하나의 팩트 테이블(Fact Table)이 여러 차원 테이블(Dimension Table)과 연결되고, 이 차원 테이블들이 다시 **하위 차원 테이블**들로 다단계 정규화되는 구조임.
-   날짜 차원이 주, 월, 연도 등의 하위 차원 테이블로 분리되는 것이 대표적 예시임.

### 3\. 결과 및 고려사항 (Trade-offs)

-   **데이터 일관성 (Data Consistency)**:
    -   정규화기 패턴의 최우선 목표는 성능 최적화가 아닌 **데이터 일관성 확보**임.
    -   데이터 수정 시 단 한 곳만 갱신하면 되므로 일관성을 쉽게 유지할 수 있음.
-   **쿼리 비용 및 JOIN 오버헤드 (Query Cost)**:
    -   데이터를 여러 테이블로 분할하므로, 조회 시 다수의 JOIN 연산이 필요함.
    -   분산 환경에서는 네트워크를 통한 데이터 셔플(Shuffle)이 발생하여 쿼리 비용과 지연 시간이 크게 증가함.
-   **완화 기법**:
    -   **로컬 조인**: 작은 차원/엔티티 테이블을 큰 테이블과 동일한 컴퓨팅 노드에 배치하여 네트워크 트래픽을 줄임.
    -   **브로드캐스트(Broadcast) 조인**: 작은 테이블을 모든 컴퓨팅 노드로 복제 전송하여 데이터 분산 비용을 회피함.
    -   _(참고)_ 큰 테이블을 브로드캐스팅할 때는 필터링을 통해 크기를 줄이거나, 아파치 스파크의 spark.sql.autoBroadcastJoinThreshold 속성 등을 활용해 제어함.
-   **아카이브 및 변경 관리**:
    -   엔티티/차원 테이블은 시간에 따라 변화함 (예: 제품의 연도별 가격 변동).
    -   과거 특정 시점의 상태를 조회해야 할 경우 SCD(Slowly Changing Dimension) 기법을 활용하여 관리함.

### 4\. 예제 코드 및 구조

**(1) 방문 이벤트 정규화 스키마 구조**

방문 데이터를 정규화하면 다음과 같이 엔티티별로 테이블이 분리된다.

-   **Visits** (기본키: visit\_id, pages\_id, user\_id, event\_time)
-   **Users** (id, login)
-   **Pages** (id, page\_categories\_id, name, url) $\\rightarrow$ **Page\_categories** (id, name, url)
-   **Visits\_contexts** (visit\_id, ads\_id, browsers\_id, devices\_id, referral, user\_ip, user\_connected\_since)
    -   **Ads** (id, name)
    -   **Browsers** (id, name, version)
    -   **Devices** (id, type, version)

**(2) 정규화된 데이터셋 조인 쿼리 오버헤드 (PySpark)**

정규화 단계가 깊어질수록 데이터를 복원하기 위해 많은 조인 연산이 필요해진다.

```
# Fully Normalized Dataset Join
context = (visits_context
    .join(ads, visits_context.ads_id == ads.id, 'left_outer').drop('id')
    .join(browser, visits_context.browsers_id == browser.id, 'left_outer').drop('id')
    .join(device, visits_context.devices_id == device.id, 'left_outer').drop('id'))

page_with_category = (pages.withColumnRenamed('id', 'page_id')
    .join(categories, pages.page_categories_id == categories.id, 'left_outer')
    .drop('id').withColumnRenamed('page_id', 'id'))

full_visit = (visits
    .join(context, visits.visit_id_event == context.visit_id, 'left_outer')
    .drop('visit_id_event')
    .join(users, visits.users_id == users.id, 'left_outer').drop('id')
    .join(page_with_category, visits.pages_id == page_with_category.id, 'left_outer')
    .drop('id').withColumnRenamed('visit_id', 'id'))
```

**(3) 스노우플레이크 스키마 조회 쿼리 오버헤드 (PySpark)**

```
# Snowflake Schema Query Overhead
page_w_category = dim_page.join(dim_page_category,
    dim_page.dim_page_category_id == dim_page_category.page_category_id,
    'left_outer')

date_with_month_and_quarter = (dim_date
    .join(dim_date_month, dim_date.dim_month_id == dim_date_month.month_id, 'left_outer')
    .join(dim_date_quarter, dim_date.dim_quarter_id == dim_date_quarter.quarter_id, 'left_outer'))

full_visit = (fact_visit
    .join(page_w_category, fact_visit.dim_page_id == page_w_category.page_id, 'left_outer')
    .join(date_with_month_and_quarter))
```

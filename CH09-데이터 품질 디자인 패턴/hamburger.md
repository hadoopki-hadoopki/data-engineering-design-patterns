# 데이터 품질 디자인 패턴

데이터셋의 신뢰성은 불완전함, 부정확함, 불일치를 방지하는 품질 확보에서 시작된다. 데이터 품질 디자인 패턴은 크게 세 가지 범주로 나뉜다.

-   **다운스트림 컨슈머 보호**: 저품질 데이터가 컨슈머에게 노출되지 않도록 사전에 차단한다.
-   **스키마 수준의 문제 해결**: 스키마 수정 및 파이프라인 진화에 따른 치명적 실패를 방지한다.
-   **지속적인 관찰(Observability)**: 오늘 설정한 품질 규칙이 미래 데이터에도 유효한지 지속적으로 검찰하고 제어한다.

# 2\. 감사-쓰기-감사-배포 (Audit-Write-Audit-Publish, AWAP)

AWAP 패턴은 데이터 흐름에 제어 장치(Assertion)를 설정하여, 데이터셋이 기대 조건에 부합하지 않을 경우 전체 파이프라인 실행을 중단하거나 제어하는 데이터 품질 보호 패턴이다.

### 1\. 문제 상황

-   일일 배치 ETL 잡에서 고유 방문자 수가 50% 감소하는 집계 오류가 발생하여 진행 중이던 마케팅 캠페인이 중단되는 사고가 발생했다.
-   소스 데이터 유입 단계나 변환 로직에서 품질 검증이 제대로 이루어지지 않아 저품질 데이터가 그대로 최종 스토리지에 적재된 것이 원인이었다.
-   향후 유사한 데이터 오류가 발생했을 때 다운스트림으로 전파되지 않도록 자동화된 데이터 제어 장치가 필요한 상황이다.

### 2\. 해결책

넷플릭스(Netflix)가 공개했던 기존 **WAP(Write-Audit-Publish)** 패턴을 발전시켜, 데이터 변환 전(입력)과 **후(출력)** 모두에 가벼운 유효성 검사 단계(Audit steps)를 배치한다.

```
[입력 원천] ──> (첫 번째 감사) ──> [추출/변환] ──> [스테이징 스토리지] ──> (두 번째 감사) ──> [최종 적재/배포]
```

#### (1) 두 가지 감사 태스크 (Audit Tasks)

-   **첫 번째 감사 (입력 감사)**
    -   데이터 변환 작업 시작 전, 유입된 원천 데이터셋을 분석한다.
    -   파일 형식 유효성, 최소/최대 파일 크기, 최소 줄 수, 필수 스키마 존재 여부 등 빠르게 처리할 수 있는 메타데이터 중심의 검사를 수행한다.
-   **두 번째 감사 (출력/변환 감사)**
    -   변환이 완료된 데이터셋을 대상으로 실행한다.
    -   특정 컬럼의 NULL 비율, 필수 컬럼 존재 여부, 값의 유효 범위, 독자성(Uniqueness) 등 실제 데이터 레코드 및 전체 데이터셋 수준의 유효성을 검증한다.

#### (2) 단위 테스트(Unit Test)와의 차이점

-   **단위 테스트**: 코드가 예상 입력에 대해 올바르게 작동하는지 확인하는 **정적** 검증 기법이다.
-   **AWAP 감사 단계**: 실제 운영 환경에서 지속적으로 유입되는 동적 데이터를 대상으로 실행되는 **실시간 확장형** 검증 기법이다.

#### (3) 감사 실패 시 데이터 처리 전략

-   **데이터 디스패칭 (Data Dispatching)**
    -   검증을 통과한 유효 레코드만 최종 스토리지로 승격(Publish)한다.
    -   유효하지 않은 레코드는 격리된 별도 스토리지(데드 레터와 유사한 명시적 제어 공간)로 보관한다.
-   **비차단 감사 (Non-blocking Audit)**
    -   약간의 결함이 있더라도 우선 최종 스토리지로 승격시킨다.
    -   대신 해당 데이터셋에 결함 항목과 신뢰성 레벨을 주석/메타데이터로 표시하여, 컨슈머가 허용 임계값에 따라 선택적으로 사용하도록 처리한다.

#### (4) 스트리밍 환경에서의 AWAP 적용

-   **윈도 기반 (Window-based)**: 처리 시간 윈도(Window) 동안 레코드를 버퍼링한 후, 윈도가 닫히는 시점에 감사 함수를 실행하여 통과 여부를 결정한다.
-   **스테이징 기반 (Staging-based)**: 스트리밍 작업 결과를 먼저 스테이징 계층(Delta Lake 등)에 기록하고, 별도의 감사 잡이 해당 스테이징 테이블을 검증한 뒤 최종 출력 위치로 승격한다.

### 3\. 결과 및 고려사항 (Trade-offs)

-   **컴퓨팅 비용 증가**:
    -   단순 메타데이터 검사는 저렴하지만, 전체 레코드를 스캔하는 깊은 검사는 컴퓨팅 자원 소비를 늘린다.
-   **규칙 포괄성의 한계**:
    -   데이터셋의 성격은 시간에 따라 변화하므로, 현재 작성한 비즈니스 규칙이 미래의 모든 데이터 변형을 100% 포괄할 수는 없다.
-   **스트리밍 지연 발생**:
    -   스트리밍 데이터 버퍼링 및 윈도 누적 시간만큼 다운스트림 데이터 제공이 지연된다.
-   **오탐(False Positive) 가능성**:
    -   예기치 않은 마케팅 성공으로 트래픽이 폭증하여 데이터 크기가 갑자기 커진 경우처럼, 정상적인 비즈니스 성장도 감사 실패로 감지될 수 있으므로 경고 발생 시 추가 조사가 필요하다.

### 4\. 예제 코드

#### (1) Airflow 기반 배치 파이프라인 (Python)

```
# 1. 입력 데이터 감사
audit_file_to_load = PythonOperator(
    task_id='audit_file_to_load',
    python_callable=local_validate_the_file_before_processing
)

# 2. 데이터 변환
transform_file = PythonOperator(
    task_id='transform_file',
    python_callable=flatten_input_visits_to_csv
)

# 3. 변환된 데이터 감사
audit_transformed_file = PythonOperator(
    task_id='audit_transformed_file',
    python_callable=local_validate_flatten_visits
)

# 4. 최종 스토리지 적재 (Publish)
load_flattened_visits_to_final_table = PostgresOperator(
    task_id='load_flattened_visits_to_final_table',
    sql='sql/load_file_to_visits_table.sql'
)

# 태스크 실행 순서 정의
next_partition_sensor >> audit_file_to_load >> transform_file >> audit_transformed_file >> load_flattened_visits_to_final_table
```

#### (2) 입력 및 출력 유효성 검사 로직

```
# 입력 데이터 검사 (파일 크기, 줄 수, JSON 유효성)
def local_validate_the_file_before_processing():
    validation_errors = []
    if f_size < min_size:
        validation_errors.append(f'File is too small. Expected at least {min_size} bytes but got {f_size}')
    if lines < min_lines:
        validation_errors.append(f'File is too short. Expected at least {min_lines} lines but got {lines}')
    if invalid_json_line:
        validation_errors.append(f'File contains some invalid JSON lines. Line {invalid_json_line_number}')
    
    if validation_errors:
        raise Exception('Audit failed for the file:\n' + '\n'.join(validation_errors))

# 처리된 데이터 검사 (Pandas 이용 NULL 값 검사)
def local_validate_flatten_visits():
    required_columns = ['visit_id', 'event_time', 'user_id', 'page', 'ip', 'login', 'browser', 'device_type']
    visits = pandas.read_csv(partition_file, sep=';', header=0)
    
    cols_w_nulls = []
    for validated_column in required_columns:
        if visits[validated_column].isnull().any():
            cols_w_nulls.append(validated_column)
            
    if cols_w_nulls:
        raise Exception('Found nulls in not nullable columns: ' + ','.join(cols_w_nulls))
```

#### (3) Apache Spark Structural Streaming & Delta Lake 스테이징 예제

```
# 1. 카프카에서 읽어와 스테이징 테이블에 쓰기
visits = (spark_session.readStream
    .option('kafka.bootstrap.servers', 'localhost:9094')
    .option('subscribe', 'visits')
    .format('kafka').load()
    .selectExpr('CAST(value AS STRING)')
    .select(F.from_json('value', get_visit_event_schema()).alias('visit'))
    .selectExpr('visit.*')
)

write_query = (visits.writeStream
    .trigger(processingTime='15 seconds')
    .option('checkpointLocation', checkpoint_dir)
    .foreachBatch(write_dataset_to_staging_table).start()
)

# 2. 스테이징 테이블에서 감사 표현식을 적용하여 검증 후 스트리밍
visits_staging = (spark_session.readStream.format('delta')
    .option('maxBytesPerTrigger', 20000000)
    .table(get_staging_visits_table())
    .withColumn('is_valid', row_validation_expression)
)

write_query_publish = (visits_staging.writeStream
    .trigger(processingTime='30 seconds')
    .option('checkpointLocation', checkpoint_dir)
)
```

# 3\. 제약 조건 적용자 

## 1\. 개념 및 문제 상황

-   **개념**: 데이터 유효성 검사를 애플리케이션 파이프라인 코드(AWAP 패턴 등)에서 직접 구현하는 대신, 데이터베이스나 스토리지 형식 자체에 품질 관리 업무를 위임하는 선언적 접근 방식이다.
-   **문제 상황**:
    -   배치 파이프라인이 정상 작동하다가 필수 필드에서 무작위로 NULL 값이 유입되는 오류가 발생한다.
    -   이미 데이터 처리 잡이 복잡하여 내부 로직에 유효성 검사 복잡성을 추가하고 싶지 않으며, 데이터 품질 오류가 발생할 때 로딩 프로세스를 자동으로 실패하게 만드는 대안이 필요하다.

## 2\. 해결책 및 제약 조건의 종류

유효성 확인 책임을 데이터베이스/스토리지에 위임한다. 제품 팀이나 법규 등 비즈니스 요구사항에 따라 검증할 속성을 식별하고 제약 조건 규칙을 할당한다.

-   **유형 제약 (Type Constraints)**
    -   주어진 속성의 모든 값이 항상 동일한 데이터 유형임을 보장한다.
    -   컨슈머의 데이터 처리 프로세스를 크게 단순화한다.
-   **NULL 허용 제약 (Nullable Constraints)**
    -   속성의 누락 가능 여부(NotNull 또는 Nullable)를 정의한다.
    -   Not-null 정의 시 값이 누락된 레코드를 거부하고, Nullable 설정 시 누락된 값을 수용한다.
-   **값 제약 (Value Constraints)**
    -   허용되는 단일 값, 값의 집합, 또는 비교 연산자 및 표현식을 기준으로 검증한다.
    -   예: 삽입 값 $x$가 미래 시간이 되지 않도록 $x \\le \\text{NOW}()$ 설정, 또는 $x \\text{ BETWEEN } 1901 \\text{ AND } 2000$ 설정.
-   **무결성 제약 (Integrity Constraints)**
    -   정규화된 트랜잭션 DB 등에서 한 테이블의 값이 다른 테이블의 실제 값을 참조하도록 보장한다.
    -   예: 존재하지 않는 페이지를 참조하는 웹사이트 방문 레코드는 방문 테이블에 추가되지 않도록 차단한다.

## 3\. 구현 기술 및 코드 예제

### (1) Delta Lake

Delta Lake는 CHECK 연산자와 NOT NULL 제약 조건을 지원한다.

```
-- 테이블 생성 시 NOT NULL 제약 적용
CREATE TABLE default.visits (
    visit_id STRING NOT NULL,
    event_time TIMESTAMP NOT NULL
) USING delta;

-- CHECK 연산자를 통한 값 제약 조건 추가
ALTER TABLE default.visits ADD CONSTRAINT
    event_time_not_in_the_future CHECK (event_time < NOW() + INTERVAL "1 SECOND");
```

-   **동작**: 위반 레코드 삽입 시 DELTA\_VIOLATE\_CONSTRAINT\_WITH\_VALUES 또는 DELTA\_NOT\_NULL\_CONSTRAINT\_VIOLATED 오류가 발생하고, 해당 트랜잭션의 어떤 레코드도 테이블에 기록되지 않는다.

### (2) Protobuf 및 protovalidate

Protobuf는 네이티브 유형 제약을 지원하며, protovalidate 라이브러리를 설치하여 값 제약 조건으로 확장할 수 있다.

```
message Visit {
    // 최소 길이 제약
    string visit_id = 1 [(buf.validate.field).string.min_len = 5];
    
    // 현재 시간 이전 제약 및 필수값 설정
    google.protobuf.Timestamp event_time = 2 [
        (buf.validate.field).timestamp.lt_now = true,
        (buf.validate.field).required = true
    ];
    
    string user_id = 3 [(buf.validate.field).required = true];
    
    // 표현식(CEL)을 이용한 확장자 검사
    string page = 4 [(buf.validate.field).cel = {
        message: "Page cannot end with an html extension"
        expression: "this.endswith('html') == false"
    }, (buf.validate.field).required = true];
}
```

-   **동작**: 규칙 위반 시 애플리케이션에서 ValidationError가 발생한다.

## 4\. 결과 및 고려사항 (Trade-offs)

-   **All-or-Nothing 시맨틱**: 입력 레코드가 규칙을 준수하지 않으면 전체 트랜잭션이 거부되며, 보통 첫 오류 위치에서 멈춘다. 전체 오류 목록을 파악하려면 프로듀서 측에서 별도 검증을 수행해야 할 수 있다.
-   **데이터 프로듀서 교대**: 제약 조건이 프로듀서 지향적이므로, 컨슈머별로 다른 데이터 요구사항(예: 특정 컨슈머에게만 필요한 Nullable 필드)을 모두 충족하지 못해 컨슈머가 별도 필터링 로직을 구현해야 할 수 있다.
-   **제약 조건 지원의 한계**: 특히 테이블 파일 형식은 무결성 제약 조건을 지원하지 않는 경우가 많아, AWAP 패턴보다 유연성이 떨어진다.

# 4\. 스키마 호환성 적용자 (Schema Compatibility Enforcer)

## 1\. 개념 및 문제 상황

-   **개념**: 시간에 따라 변화하는 데이터셋에서 데이터 프로듀서가 중단을 일으키는 스키마 변경을 하지 못하도록 호환성 규칙을 강제하는 패턴이다.
-   **문제 상황**:
    -   데이터 생성 팀에서 특정 필드가 불필요하다고 판단하여 사전 공유 없이 삭제했다.
    -   이로 인해 해당 필드를 참조하던 다운스트림 데이터 애플리케이션 및 세션화 잡이 지속적으로 실패하는 문제가 발생한다.

## 2\. 세 가지 스키마 호환성 적용 모드

1.  **외부 서비스나 라이브러리를 통한 적용**
    -   Apache Kafka의 Schema Registry처럼 각 스키마에 버전을 지정하고 구성된 규칙에 따라 변경 유효성을 검사한다.
    -   Apache Avro의 SchemaValidator 같은 라이브러리를 활용할 수도 있다.
2.  **삽입 시 암시적 적용**
    -   테이블 파일 형식이나 관계형 DB에서 새 테이블 생성 시 제약 조건을 정의하고, 이를 위반하는 레코드 쓰기를 암시적으로 거부한다.
3.  **DDL에 대한 이벤트 주도 적용**
    -   PostgreSQL, SQL Server 등에서 DROP COLUMN, RENAME COLUMN 같은 DDL 작업 커밋 전 이벤트 트리거(SQL 함수)를 실행하여 검증한다.
    -   세밀한 제어가 필요 없다면 사용자에게 ALTER TABLE 권한을 부여하지 않아 스키마 수정을 사전에 방지한다.

## 3\. 스키마 호환성 모드의 종류

| 호환 모드 | 허용된 행동 | 시맨틱 |
| --- | --- | --- |
| 하위 호환성 (Backward) | 필드 삭제, 선택적 필드 추가 | 새로운 버전을 사용하는 컨슈머가 오래된 버전으로 생성된 데이터를 조회할 수 있다. |
| 상위 호환성 (Forward) | 필드 추가, 선택적 필드 삭제 | 오래된 버전을 사용하는 컨슈머가 새로운 버전으로 생성된 데이터를 조회할 수 있다. |
| 완전 호환성 (Full) | 선택적 필드 추가, 선택적 필드 삭제 | 구/신 버전 컨슈머 모두 상대방 버전으로 생성된 데이터를 서로 조회할 수 있다. |

### 전이적(Transitive) 호환성의 파괴 사례

호환성은 전이적일 수도 있고 비전이적일 수도 있다. 비전이적 하위 호환성 설정 시 발생할 수 있는 문제는 다음과 같다.

1.  **v0 스키마**: order\_id LONG REQUIRED
2.  **v1 스키마**: amount DOUBLE DEFAULT 0.0 추가 (선택적 필드 추가이므로 v0과 하위 호환됨).
3.  **v2 스키마**: 분석팀의 요청으로 기본값을 제거하고 amount DOUBLE REQUIRED로 변경.

-   **문제 점검**: v2를 사용하는 최신 컨슈머는 v0에서 생성된 데이터를 조회할 때 필수 항목인 amount 필드가 없어 오류가 발생한다. 즉, v1-v2 및 v0-v1 간은 호환되지만 v0-v2 간의 **전이적 하위 호환성**은 파괴된다.

## 4\. 결과 및 고려사항 (Trade-offs)

-   **상호 작용 오버헤드**: 스키마 레지스트리 등 외부 컴포넌트를 거쳐 레코드별 유효성을 검사하는 과정에서 성능 오버헤드가 발생한다.
-   **스키마 진화의 복잡성**: 스키마 호환성 규칙을 준수해야 하므로 단순한 필드 이름 변경조차 어려워진다. (보통 '새 필드 추가' 후 '기존 필드 사용 중단' 단계를 거쳐야 함).

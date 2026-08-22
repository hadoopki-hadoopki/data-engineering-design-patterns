
## Pattern: Audit-Write-Audit-Publish (AWAP)

### (5) 최신트렌드

**목적: 요즘 실무에서 audit 로직을 뭘로 짜고, 왜 그게 더 낫다고 느끼는지.**

### 1. Great Expectations (GX)

- 정체: 데이터 검증 규칙("expectation")을 선언적으로 정의하고, 파이프라인 어디서든 실행할 수 있게 해주는 오픈소스 데이터 품질 프레임워크
- 이전 한계: 책 예시처럼 `if f_size < min_size: ...` 식으로 검증 로직을 직접 파이썬 코드로 하나하나 짜야 했다. 검증 규칙이 늘어날수록 코드가 지저분해지고, 팀마다 검증 스타일이 제각각이었다
- 요즘 쓰는 이유

> 예전엔 audit 함수 하나 짤 때마다 `if`문 쌓고 `validation_errors.append` 반복하는 게 일이었다. GX는 이걸 `expect_column_values_to_not_be_null("user_id")` 처럼 한 줄 선언으로 끝낸다. 검증 규칙이 코드가 아니라 설정(config)처럼 관리되니까, 이 required_columns 11개 체크하는 로직도 실제로는 몇 줄이면 끝난다. 게다가 실행 결과가 자동으로 HTML 리포트로 나와서, "오늘 audit에서 뭐가 걸렸는지" 팀 전체가 같이 보기 편하다.

### 2. dbt tests

- 정체: dbt(변환 파이프라인 도구) 안에서 SQL 모델 옆에 YAML로 테스트 규칙을 같이 선언하는 기능 (not_null, unique, accepted_values 등 내장 테스트 제공)
- 이전 한계: 예전엔 변환 로직(SQL)과 검증 로직(Python audit 함수)이 완전히 분리된 코드베이스에 있어서, 이 컬럼이 무슨 검증을 받는지 한눈에 안 보였다
- 요즘 쓰는 이유

> dbt 쓰면 `visits.sql` 모델 옆에 `schema.yml`에서 `user_id: tests: [not_null]`처럼 바로 붙여둔다. 코드랑 테스트가 같은 프로젝트, 같은 파일 구조 안에 있으니까 "이 컬럼이 뭘 검증받는지" 찾으려고 다른 코드베이스 뒤질 필요가 없다. dbt run이 끝나면 dbt test가 바로 이어서 도니까, 이게 사실상 Write → Audit(출력) 흐름을 그대로 자동화해주는 셈이다.

### 3. Delta Lake / Iceberg의 네이티브 제약조건 (constraints)

- 정체: 테이블 자체에 `NOT NULL`, `CHECK` 제약을 걸어서, INSERT 시점에 DB 엔진이 자동으로 거부하게 만드는 기능
- 이전 한계: 책 예시의 CSV는 "constraintless 포맷"이라, NULL 체크를 파이프라인 코드(audit 함수)에서 직접 짜야만 했다
- 요즘 쓰는 이유

> Delta Lake 테이블에 `ALTER TABLE visits ADD CONSTRAINT ...`로 제약을 걸어두면, 애초에 파이프라인 코드에 NULL 체크 로직을 안 짜도 된다. DB가 알아서 막아준다. 이게 바로 다음에 배울 Constraints Enforcer 패턴인데, 요즘은 AWAP(코드로 직접 검증)과 Constraints Enforcer(DB가 대신 검증)를 같이 쓰는 게 표준이다 — 쉬운 제약은 DB한테 맡기고, 복잡한 비즈니스 로직(row 수 급감 감지 같은)만 AWAP 코드로 남긴다.

### 실무 선호 정리

- **Great Expectations**: 검증 규칙이 많고 복잡하며, 여러 팀이 공통으로 쓰는 데이터 품질 표준을 만들고 싶을 때
- **dbt tests**: 이미 dbt로 변환 파이프라인을 짜고 있는 팀 — 추가 도구 없이 자연스럽게 붙일 수 있어서 진입장벽이 제일 낮음
- **DB 네이티브 제약(Delta/Iceberg)**: 단순한 규칙(NOT NULL, 값 범위)이고, DB 레벨에서 무조건 막고 싶을 때 — 성능도 제일 좋음 (파이프라인 코드가 매번 전체 스캔 안 해도 됨)

### 엔지니어 독백

> 결국 이 셋 다 하는 일은 책에서 배운 "audit 단계를 어디에 두느냐"는 판단과 똑같다. 다른 게 있다면 예전엔 그 audit 로직을 무조건 내가 파이썬으로 짜야 했는데, 지금은 선언적 도구(GX, dbt)나 DB 자체 기능(constraint)으로 위임할 수 있는 선택지가 많아졌다는 거다.
> 
> 신입 때 헷갈리기 쉬운 게 "그럼 이제 AWAP 개념 자체는 필요 없나?"인데, 반대다. GX든 dbt test든 내부적으로 실행되는 시점은 여전히 Audit-Write-Audit-Publish의 어느 단계인지 판단해야 한다. 도구가 검증 코드 작성을 편하게 해줄 뿐이지, "이 검증을 입력에 걸지 출력에 걸지, 실패하면 막을지 흘려보낼지"는 여전히 엔지니어가 설계해야 하는 부분이다.





## Great Expectations / dbt tests / DB Constraints — 검증 가능 범위 상세 (번호 정리)

**결론: NULL 체크는 극히 일부 예시일 뿐이고, 실제로는 컬럼 단위 · row 단위 · 테이블 전체 단위 · 테이블 간 관계까지 커스텀 검증이 가능하다.**

### 검증 레벨 4단계 (공통 개념)

1. 컬럼 값 검증 — 하나의 값이 조건을 만족하는가
2. row 단위 비즈니스 로직 검증 — 여러 컬럼 값의 조합이 말이 되는가
3. 데이터셋 전체 통계 검증 — 분포, 비율, row 수 등
4. 테이블 간 관계 검증 — 참조 무결성, 다른 테이블과의 정합성

---

### 1. Great Expectations

**1-1. 컬럼 값 검증**
```python
expect_column_values_to_not_be_null("user_id")
expect_column_values_to_be_between("browser_version", min_value=1, max_value=200)
expect_column_values_to_match_regex("ip", r'^\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}$')
expect_column_values_to_be_in_set("device_type", ["mobile", "desktop", "tablet"])
expect_column_values_to_be_unique("visit_id")
```

- NULL 체크뿐 아니라 값 범위, 정규식 패턴(IP 형식 등), 허용된 값 집합(enum처럼), 유니크 여부까지 커버

**1-2. row 단위 로직**
```python
expect_column_pair_values_A_to_be_greater_than_B("event_time", "created_at")
```

- 예: event_time이 created_at보다 항상 이후여야 한다는 컬럼 간 관계 검증

**1-3. 데이터셋 전체 통계**
```python
expect_table_row_count_to_be_between(min_value=10000, max_value=1000000)
expect_column_mean_to_be_between("browser_version", min_value=80, max_value=140)
expect_column_proportion_of_unique_values_to_be_between("user_id", min_value=0.3, max_value=0.9)
```

- row 수 급감/급증 감지(문제상황에서 겪은 케이스), 평균값 이상치 감지, 유니크 비율 감지까지 가능

**1-4. 커스텀 SQL/Python 함수**
```python
def expect_page_to_not_contain_deprecated_paths(column):
    return not column.str.contains("/old-blog/")
```

- 내장 expectation 200개 이상 + 커스텀 expectation도 파이썬 함수로 직접 정의 가능

---

### 2. dbt tests

**2-1. 내장(generic) 테스트 4종**
```yaml
columns:
  - name: user_id
    tests:
      - not_null
      - unique
  - name: device_type
    tests:
      - accepted_values:
          values: ['mobile', 'desktop', 'tablet']
  - name: dim_page_id
    tests:
      - relationships:
          to: ref('dim_page')
          field: page_id
```

- `relationships` 테스트가 핵심 — 테이블 간 참조 무결성(레벨 4) 검증. fact_visit의 dim_page_id가 dim_page 테이블에 실제로 존재하는 값인지 체크


**2-2. 커스텀(singular) 테스트 — 원하는 SQL 아무거나**
```sql
-- tests/assert_visit_time_not_future.sql
-- 이 쿼리가 row를 하나라도 반환하면 테스트 실패
SELECT visit_id
FROM {{ ref('visits') }}
WHERE event_time > CURRENT_TIMESTAMP
```

- SQL로 표현 가능한 거의 모든 로직을 테스트로 만들 수 있음. row 수 비교, 컬럼 간 계산 검증, 시계열 이상치 등 전부 SQL 한 방으로 커스텀 가능
---

### 3. DB 네이티브 제약 (Delta Lake CHECK constraint)
```sql
ALTER TABLE default.visits ADD CONSTRAINT
    event_time_not_in_the_future CHECK (event_time < NOW() + INTERVAL '1 SECOND');

ALTER TABLE default.visits ADD CONSTRAINT
    device_type_valid CHECK (device_type IN ('mobile', 'desktop', 'tablet'));
```

- CHECK 절 안에 SQL 표현식을 넣을 수 있어서, 값 범위 · enum · 컬럼 간 비교 등 레벨 1~2 수준은 DB가 INSERT 시점에 바로 막아줌
- 레벨 3(통계) · 레벨 4(테이블 간 관계)는 DB 제약만으로는 표현 안 됨 → GX나 dbt test 영역

---

### 레벨별 도구 매핑 정리

|레벨|예시|1. Great Expectations|2. dbt tests|3. DB 제약|
|---|---|---|---|---|
|레벨1. 컬럼 값|NULL, 범위, 정규식, enum|가능|가능|가능|
|레벨2. row 단위 로직|컬럼 간 비교|가능|가능 (커스텀 SQL)|가능 (CHECK)|
|레벨3. 데이터셋 통계|row 수 급감, 평균 이상치|가능|가능 (커스텀 SQL)|불가능|
|레벨4. 테이블 간 관계|참조 무결성|가능 (커스텀)|가능 (relationships)|제한적 (FK 지원 DB에 한함)|

### 엔지니어 독백

> 책 예시가 NULL 체크만 보여준 건 "제일 이해하기 쉬운 예시라서"지, AWAP이 원래 그 정도만 하는 패턴이 아니다. 실무에서 진짜 골치 아픈 버그는 대부분 레벨3(통계 이상치)이나 레벨4(테이블 간 정합성 깨짐)에서 터진다. 신입 때 NULL 체크만 잔뜩 짜놓고 "이 정도면 충분하다" 생각하기 쉬운데, 진짜 사고는 "값은 다 있는데 숫자가 이상하다"거나 "fact 테이블 참조 키가 dimension에 없는 유령 값이다" 같은 데서 난다.
> 
> 그래서 실무에서 팀 규모가 커지면 dbt test의 relationships부터 깐다. star schema, snowflake schema 다 결국 fact-dimension 관계가 정합성의 핵심인데, 이게 깨지는 걸 사람이 매번 눈으로 확인할 수 없으니까 이 테스트 하나로 자동 감시를 건다.

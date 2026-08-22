
# 패턴 #58: 역정규화기 (Denormalizer)
## (1) 문제상황

**핵심 고통: 조회 한 번에 테이블 8개를 join하느라 쿼리가 안 끝난다.**

- 등장 컴포넌트 
    - **data warehouse**: 분석용 쿼리를 돌리는 대용량 저장·쿼리 엔진 (예: BigQuery, Redshift, Snowflake).
    - **정규화 모델(relational/snowflake schema)**: 중복을 없애려고 정보를 여러 테이블로 쪼개 둔 구조. 바로 앞 Normalizer 패턴의 결과물.
    - **join**: 쪼개진 테이블을 다시 붙여서 원래 한 줄의 전체 그림을 복원하는 연산. 이게 이 문제의 주범.
- 엔지니어 독백

> 처음엔 정규화가 정답이었다. 중복 없고, 업데이트 한 방이면 끝. device 이름 하나 바꿔도 dimension 테이블 한 행만 고치면 됐으니까.
> 
> 그런데 제품이 대박났다. 데이터 양이 폭발하면서 분석팀이 들고 일어났다. "쿼리가 2분, 10분씩 걸린다"고.
> 까보니 원인이 명확했다. 전체 쿼리의 80%가 테이블 8개를 전부 join하고 있었다. visit 한 줄의 전체 그림을 보려면 user, device, page, page_category, date, month, quarter... 다 붙여야 했으니까.
> 
> join이 왜 느리냐면 — 붙일 테이블들이 물리적으로 다른 노드에 흩어져 있어서, join 때마다 노드 간에 데이터를 네트워크로 실어 나른다. 테이블이 많아질수록 이 왕복이 기하급수로 늘어난다.(셔플)
> 
> 급한 불은 클러스터에 컴퓨트를 더 붙여서 껐다. 근데 그건 돈으로 때우는 거지 해결이 아니다. 비용은 오르고 근본 원인은 그대로다.
> 
> 그래서 발상을 뒤집었다. 매 쿼리마다 join할 거면, 아예 미리 다 붙여서 한 줄로 저장해두면 안 되나?

- 이 "미리 다 붙여 한 줄로 펴는" 발상이 바로 **Denormalizer**.
- 대신 새 문제가 따라온다: 중복이 다시 생긴다. 정규화로 없앴던 중복이 부활하고, 이로 인해 업데이트 비용·저장 공간 문제가 생긴다. (이건 (3)결과에서 다룸)
- 한 줄 정리
    - Normalizer는 **일관성**을 위해 쪼갰고, Denormalizer는 **조회 속도**를 위해 다시 붙인다. 둘은 정반대 트레이드오프.



## (2)솔루션

**핵심 메커니즘: join으로 흩어진 테이블들을 쓰기 시점에 미리 합쳐서, 한 row 안에 필요한 값을 전부 넣어버린다.**

값을 한 row 안에 넣는 방식은 두 가지다.

### 1. 일반 컬럼으로 펴기 (flat columns)

- 처리 내용: join된 다른 테이블의 컬럼을 각각 최상위 컬럼으로 그대로 복사해 넣는다.
- 사용자는 `SELECT page_name FROM visits_flat` 처럼 컬럼에 바로 접근한다. 중첩 구조를 뚫고 들어갈 필요가 없다.
- **One Big Table (OBT)** 방식: [1]visit + [2]user + [3]device → 한 테이블

|visit_id|user_id|user_name|device_id|device_full_name|visit_time|visited_page|
|---|---|---|---|---|---|---|
|1|409|user ABC|10000|local computer|2024-07-01T09:00:00Z|home.html|


### 2. 중첩 구조로 넣기 (nested STRUCT)

- `STRUCT` : 컬럼 하나 안에 여러 필드를 담는 복합 타입
	- 딕셔너리 형태 
	- BigQuery, Delta Lake 다 지원
	- ex) {device_id: 10000, device_full_name: "local computer"}
- 처리 내용:  여러 컬럼 내용 →  하나의 STRUCT 타입 컬럼 안에 통째로.

- 조회방법 : 점(`.`)으로 필드를 파고 들어감
	- `visit.device.device_full_name` 처럼
	- `visit.`  : 테이블
	- `device.` : 컬럼
	- `device_full_name.` : STRUCT 컬럼 값의 '내부필드' ← 키값이 아니라 내부필드라고 명칭함 
- 컬럼 하나가 곧 작은 서브 테이블인 셈.

| visit_id | visit_time           | device (STRUCT)                                        |
| -------- | -------------------- | ------------------------------------------------------ |
| 1        | 2024-07-01T09:00:00Z | {device_id: 10000, device_full_name: "local computer"} |


> Q.Struct 타입의 수정은 어떻게 하나?  내부필드 개별로 수정이 가능한지?, 전체값 덮어쓰기 방식인지? 
**결론: 쿼리 문법상으로는 필드 하나만 지정해서 수정 가능하다. 그런데 저장소 내부 동작은 그 row가 통째로 다시 쓰인다.**

1. 쿼리 레벨 — 필드 단위로 UPDATE 가능
Delta Lake는 점 표기법으로 STRUCT 내부 필드를 직접 타겟팅해서 UPDATE를 허용한다.
```sql
UPDATE visits_flat
SET device.device_full_name = 'updated computer'
WHERE visit_id = 1;
```
- `device.device_full_name`만 지정 — `device_id`는 안 건드림
- 문법상으로는 STRUCT 전체를 다시 쓸 필요 없이 필드 하나만 골라 수정 가능

2. 저장 레벨 — 실제로는 row/파일 전체가 다시 써진다
```
왜 그런가:

Parquet(Delta Lake의 물리 저장 포맷) 자체가 불변(immutable) 구조
→ 파일 안의 특정 값 하나만 콕 집어서 in-place로 고치는 게 원천적으로 불가능
→ UPDATE 명령이 들어오면:
   1. 해당 row가 들어있는 파일을 찾음
   2. 그 파일 전체를 읽음
   3. 메모리상에서 device.device_full_name 값만 바꿈
   4. 파일 전체를 새로 씀 (새 Parquet 파일)
   5. 기존 파일은 안 쓰는 것으로 표시 (Delta Lake transaction log에 기록)
```
- 이 방식을 **copy-on-write**라고 부른다. "필드 하나만 바꿨다"고 느껴지지만, 물리적으로는 그 필드가 속한 파일(보통 수천~수만 row)이 통째로 재작성된다.
**엔지니어 독백**
> 처음엔 "필드 하나만 UPDATE 치는 거니까 가볍겠지" 오해하기 쉽다. 실제로는 Delta Lake든 Iceberg든 Parquet 기반은 다 이 방식이다. UPDATE 문 하나가 파일 하나(보통 128MB~1GB 단위)를 통째로 다시 쓸 수도 있다.
> 
> 그래서 STRUCT 필드를 자주, 소량씩 UPDATE 쳐야 하는 워크로드라면 Delta Lake보다 파티셔닝을 잘게 쪼개서 재작성 범위를 줄이거나, 애초에 자주 바뀌는 필드는 STRUCT 안에 넣지 말고 별도 테이블로 분리하는 걸 고려해야 한다. STRUCT는 "같이 조회되는" 속성을 묶기 좋은 거지, "자주 개별 수정되는" 속성을 묶기엔 안 맞는다.


> Q.조회비용이 비쌀것 같은데?
**결론: UPDATE(쓰기) 비용은 비쌀 수 있지만, SELECT(조회) 비용은 오히려 낮다.** 방금 걱정한 지점이 정확히 반대다 — 비싼 건 쓰기, 싼 건 읽기.

- STRUCT 내부 필드도 각각 **독립된 컬럼처럼 별도로 저장**된다. `device`라는 박스 하나로 뭉쳐서 저장되는 게 아니다.
```
연산       | 비용                          | 이유
-----------+-------------------------------+------------------------------
SELECT     | 낮음 (컬럼 프루닝)              | 필요한 필드만 디스크에서 읽음
UPDATE     | 높음 (copy-on-write)          | 파일 전체를 재작성해야 함
```

**엔지니어 독백**
> 헷갈리는 게 당연하다. "박스 안에 들어있으니까 박스 전체를 읽어야 하지 않나" 싶은데, Parquet은 그렇게 안 만들어져 있다. STRUCT여도 내부 필드 하나하나가 물리적으로 분리된 컬럼처럼 저장되기 때문에, 읽기는 여전히 필요한 것만 선택적으로 가능하다.
> 
> 실무에서 STRUCT 쓰는 이유가 사실 이거다 — "논리적으로는 묶어서 보기 좋게, 물리적으로는 펴서 읽는 것처럼 빠르게" 두 마리 토끼를 잡으려고 쓰는 거다. 비싼 건 조회가 아니라 UPDATE 쪽이니, STRUCT 설계할 때는 "이 필드들이 자주 같이 조회되는가"만 신경 쓰면 되고 "조회가 느려질까"는 걱정 안 해도 된다.




### One Big Table vs Star Schema — 어디까지 flatten하나

둘 다 Denormalizer지만, "얼마나 철저하게" 붙이냐가 다르다.

|구분|One Big Table|Star Schema|
|---|---|---|
|구조|관련 테이블 전부를 fact 테이블 하나로 완전히 흡수|fact 테이블(중심) + dimension 테이블(주변)로 여전히 분리|
|dimension 중첩|없음 (다 fact에 흡수됨)|dimension이 또 다른 dimension을 참조하지 않음 (snowflake와 이 점에서 다름) — 딱 1단계까지만 분리|
|조회 시 join|0번|필요 (단, snowflake보다 훨씬 적음 — 1단계뿐이므로)|
|저장 중복|최대|OBT보다 적음|

```
Star schema 구조:

dim_date        dim_page
	\              /
	 \            /
	  fact_visit  ←── dim_page_category (dim_page의 자식, join 1단계 더 필요)
	 /
dim_user
```

- Star schema는 snowflake schema(Normalizer 패턴)에서 "dimension이 또 다른 dimension을 참조하는" 깊은 계층만 잘라낸 것. 완전히 정규화하지도, 완전히 OBT로 뭉치지도 않은 절충안.

### 실무 선호 — 상황별

- **One Big Table 선호 상황**: 특정 도메인(예: "이 visit 이벤트")에 관련된 속성들이 명확하게 하나로 묶이고, 조회 패턴이 거의 항상 "그 도메인 전체를 한 번에 본다"일 때. 분석가가 join 없이 즉시 조회해야 하는 대시보드/리포트용 테이블에 많이 씀.
    - 주의: 서로 관련 없는 속성(예: 유저의 즐겨찾는 색상 + 이번 visit 정보)까지 억지로 한 테이블에 넣으면 안티패턴이 된다. "테이블 이름을 지어봤을 때 and/with가 여러 번 들어가면 위험 신호"라는 게 책의 기준.
- **Star schema 선호 상황**: dimension 데이터가 자주 바뀌고(예: page_category 재분류), 그 변경을 한 곳에서만 관리하고 싶을 때. join이 1단계뿐이라 완전 정규화보다는 훨씬 싸다. BI 툴(Tableau, Looker 등)이 star schema를 기본 전제로 동작하는 경우도 많아서 실무에서 표준으로 제일 많이 씀.

두 패턴은 배타적이지 않다. Normalizer로 정확한 snowflake schema를 원본으로 유지하면서, 그 위에 조회용 OBT를 파생시켜 둘 다 노출하는 것도 실무에서 흔한 구성이다. (동기화는 Chapter 6의 sequence 패턴으로 처리)


## (3)결과

**결론: 반정규화 →  조회 속도를 얻는 대신, 데이터 일관성(consistency)을 잃는다.**

### 단점 1: 업데이트 비용 증가

- 배경: 정규화 상태 — 값 하나가 테이블 한 곳에만 존재
- 문제: flatten 후 — 같은 값이 여러 row에 중복 저장 → 값 하나 바뀌면 관련 row 전부 수정 필요
- 대응
    - denormalized 테이블을 "특정 시점 스냅샷"으로 정의 → 업데이트 자체를 안 함
    - 항상 최신 상태 필요하면 → 비싼 업데이트 비용 그냥 감수

### 단점 2: 저장 공간 증가

- 배경: join된 값들이 매 row마다 반복 저장
- 문제: page_name, category_name 같은 문자열이 방문 건수만큼 중복 → 저장 공간 증가
- 대응: dictionary encoding
    - 실제 값 ↔ 압축된 표현(정수 등) 매핑 테이블 생성
    - 컬럼에는 압축값만 저장 (예: `{1: "긴 문자열...", 2: "다른 긴 문자열..."}`)
    - 부가 효과: "이 값 존재하나?" 체크 시 전체 데이터 안 읽고 매핑 테이블만 확인 → 성능도 개선

### 단점 3: One Big Table 안티패턴화

- 배경: OBT 취지 — 관련 속성을 한 테이블에 모으기
- 문제: 무관한 속성까지 욕심내서 다 넣음 → 밖에서 뭐가 들었는지 알 수 없는 "쓰레기봉투 테이블" 됨
    - 예: visit 테이블에 유저 과거 주문 목록, 즐겨찾는 색상까지 추가 → visit과 무관
- 대응: 테이블 이름 자가진단
    - 이름에 "and", "with"가 여러 번 들어가야 설명됨 → 무관한 속성 억지로 묶은 신호
    - 도메인 범위 먼저 정의 → 그 범위 안 속성만 flatten

### 엔지니어 독백

> Denormalizer 처음 쓸 때 제일 흔한 실수 — "일단 다 붙이고 보자." 관련 없는 속성까지 욱여넣으면, 나중에 유지보수하는 사람이 "이 컬럼 왜 여기 있지?" 하며 헤맨다.
> 
> 업데이트 비용 문제는 직접 겪어봐야 감 온다. dim 테이블 하나 고치는 배치가 갑자기 OBT 테이블 수백만 row를 재작성해야 하면, 그제야 "이거 스냅샷으로 관리했어야 했나" 깨닫는다. 그래서 설계 단계에서 "이 OBT가 최신 상태 유지해야 하나, 하루 단위 스냅샷이어도 되나"부터 정하고 시작해야 한다. 나중에 바꾸려면 파이프라인 다시 짜야 한다.


## (4)예시

- **예시1: One Big Table 쓰기/읽기 (Example 8-25)**
- **예시2: Star Schema 쓰기/읽기 (Example 8-26)**
### <예시1>
### 1단계: dim_page + dim_page_category join (page 관련 정보 flatten)

```python
page_w_category = dim_page.join(
    dim_page_category,
    dim_page.dim_page_category_id == dim_page_category.page_category_id,
    'left_outer'
)

page_w_category.show()

page_id | page_name  | page_category_id | page_category_id | category_name
--------+------------+-------------------+-------------------+---------------
    100 | home.html  |                 5 |                 5 | landing
    101 | product.html|                6 |                 6 | catalog
```

- 이 조각이 하는 일: `dim_page`와 `dim_page_category` 두 dimension 테이블을 미리 합쳐서, page 이름과 category 이름을 한 row에 같이 담는다.
- 핵심 문법
    - `.join(대상, 조건, 방식)` — Spark DataFrame의 join 메서드. 첫 인자는 합칠 대상 DataFrame, 둘째는 join 조건, 셋째는 join 종류.
    - `'left_outer'` — left outer join. 왼쪽(`dim_page`)의 모든 row는 무조건 살아남고, 오른쪽(`dim_page_category`)에 매칭되는 값이 없으면 해당 컬럼은 NULL로 채워진다. dimension 테이블끼리 join할 때, page는 있는데 category가 아직 안 채워진 경우를 보호하려고 outer를 씀.

### 2단계: dim_date + dim_date_month + dim_date_quarter join (날짜 관련 정보 flatten)

```python
date_w_month_quarter = (dim_date
    .join(dim_date_month, dim_date.dim_month_id == dim_date_month.month_id, 'left_outer')
    .join(dim_date_quarter, dim_date.dim_quarter_id == dim_date_quarter.quarter_id, 'left_outer'))
    

date_w_month_quarter.show()

date_id  | full_date  | month | quarter
---------+------------+-------+---------
20240701 | 2024-07-01 |     7 | Q3
20240702 | 2024-07-02 |     7 | Q3
```

- 이 조각이 하는 일: `dim_date`를 기준으로 월(`dim_date_month`), 분기(`dim_date_quarter`) 정보까지 순서대로 이어붙여서 하나의 날짜 정보 DataFrame으로 만든다.
- 핵심 문법
    - `.join().join()` 체이닝 — join 결과가 다시 DataFrame이므로, 그 위에 바로 다음 join을 이어 붙일 수 있다. 1단계는 join 1번, 이 단계는 join 2번 연속.
    - 여기서 `dim_date`는 원래 snowflake schema처럼 month, quarter가 각각 별도 테이블로 정규화돼 있던 상태였다는 게 전제다. 이 join으로 그걸 다시 하나로 합치는 것.

1단계에서 page 관련 정보를, 2단계에서 date 관련 정보를 각각 별도로 미리 flatten해뒀다. 3단계에서 이 둘을 fact 테이블과 최종 결합한다.


### 3단계: fact_visit과 최종 결합 (One Big Table 완성)
```python
full_visit = (fact_visit
    .join(page_w_category, fact_visit.dim_page_id == page_w_category.page_id, 'left_outer')
    .join(date_w_month_quarter, fact_visit.dim_date_id == date_w_month_quarter.date_id, 'left_outer')
)


full_visit.show()
visit_id | page_name  | category_name | full_date  | month | quarter
---------+------------+----------------+------------+-------+---------
       1 | home.html  | landing        | 2024-07-01 |     7 | Q3
       2 | product.html| catalog       | 2024-07-01 |     7 | Q3
```

- 이 조각이 하는 일: 사실 테이블 `fact_visit`에, 1단계에서 만든 page 정보와 2단계에서 만든 date 정보를 각각 join해서 최종적으로 "한 row에 모든 정보가 다 담긴" DataFrame을 만든다.
- 핵심 문법
    - `fact_visit.dim_page_id == page_w_category.page_id` — join 조건. fact 테이블의 참조 키(dim_page_id)와 dimension 쪽의 실제 키(page_id)를 매칭.
    - 이 시점에서 `full_visit`은 join이 총 4번(1단계 1번 + 2단계 2번 + 3단계 2번... 정확히는 1단계1+2단계2+3단계2=5번) 이미 일어난 결과물이지만, **이건 지금 딱 한 번(쓰기 시점)만 발생**한다는 게 핵심.

### 4단계: One Big Table로 저장 (쓰기 비용을 여기서 딱 한 번 지불)

```python
full_visit.write.mode('overwrite').format('delta').save(get_one_big_table_dir())


full_visit.write.mode('overwrite').format('delta').save(...)
>>> 실행 완료. 반환값 없음.
>>> 디스크 상태: /path/to/one_big_table/ 안에 Delta 파일들(.parquet + _delta_log) 생성됨
```

- 이 조각이 하는 일: 지금까지 join으로 완성한 `full_visit` DataFrame을 Delta Lake 포맷으로 디스크에 저장한다.
- 핵심 문법
    - `.write` — DataFrame을 저장 모드로 전환하는 진입점.
    - `.mode('overwrite')` — 기존에 같은 경로에 저장된 데이터가 있으면 덮어쓴다. (추가 append가 아님)
    - `.format('delta')` — 저장 포맷을 Delta Lake로 지정. Parquet 기반이지만 트랜잭션 로그가 추가된 포맷.
    - `.save(경로)` — 실제 저장 위치.

### 5단계: 분석가의 조회 (join 없이, 최종 사용 시나리오)

```python
visits_table = spark_session.read.format('delta').load(get_one_big_table_dir())


visits_table = spark_session.read.format('delta').load(...)
visits_table.show()

visit_id | page_name  | category_name | full_date  | month | quarter
---------+------------+----------------+------------+-------+---------
       1 | home.html  | landing        | 2024-07-01 |     7 | Q3
       2 | product.html| catalog       | 2024-07-01 |     7 | Q3
```

- 이 조각이 하는 일: 저장해둔 One Big Table을 그냥 읽어오기만 한다. 여기엔 join이 전혀 없다.
- 앞 단계(1~4단계)에서 미리 join을 다 해뒀기 때문에, 5단계(실제 분석가가 매번 실행하는 조회)는 비용이 거의 안 든다. 1~4단계는 하루 1번 배치로 도는 비용, 5단계는 분석가가 하루 수백 번 실행하는 비용 — 이 비대칭이 Denormalizer의 핵심 이득이다.

### 최종 요약

```
한 줄 요약: dimension 테이블들을 미리 join해서 flat한 One Big Table로 저장하고, 조회는 join 없이 읽기만 한다.

실행 순서: 1단계(page flatten) → 2단계(date flatten) → 3단계(fact와 결합) → 4단계(저장, 비용 지불 시점) → 5단계(조회, 비용 없음)

핵심 라인: 4단계의 .write.mode('overwrite').format('delta').save(...) — 여기가 join 비용을 "매 조회"에서 "매 쓰기"로 옮기는 지점.
```


### <예시2>
### 1단계: page + page_category join → dim_page(star schema용) 저장
```python
page_with_category = dim_page.join(dim_page_category,
    dim_page.dim_page_category_id == dim_page_category.page_category_id,
    'left_outer').dropDuplicates()

page_with_category.write.mode('overwrite').format('delta').save(output_page)
```

- 이 조각이 하는 일: page와 category를 미리 join해서, star schema의 `dim_page` 테이블(= category_name까지 이미 펴진 상태)을 만들어 저장한다.
- 핵심 문법
    - `.dropDuplicates()` — join 결과에서 완전히 동일한 row가 여러 개 생겼을 때 중복을 제거. join 특성상 1:N 관계면 같은 row가 반복될 수 있어서, dimension 테이블은 유니크해야 하므로 넣는다.
    - `.write...save(output_page)` — 이 결과 자체가 star schema의 최종 `dim_page` 테이블. One Big Table과 달리 fact와 아직 합치지 않고 별도 테이블로 저장한다는 게 핵심 차이.

**실행 결과**
```
page_with_category.show()

page_id | page_name   | page_category_id | category_name
--------+-------------+-------------------+---------------
    100 | home.html   |                 5 | landing
    101 | product.html|                 6 | catalog

>>> 저장 완료. output_page 경로에 Delta 파일 생성됨.
```


### 2단계: date + month + quarter join → dim_date(star schema용) 저장
```python
date_with_month_and_quarter = (dim_date
    .join(dim_date_month, dim_date.dim_month_id == dim_date_month.month_id, 'left_outer')
    .join(dim_date_quarter, dim_date.dim_quarter_id == dim_date_quarter.quarter_id, 'left_outer')).dropDuplicates()

date_with_month_and_quarter.write.mode('overwrite').format('delta').save(output_date_dir)
```
- 이 조각이 하는 일: 1단계와 같은 논리로, date/month/quarter를 미리 합쳐서 star schema의 `dim_date` 테이블을 만들어 저장한다.
- 1단계와 이어지는 관계: 1단계는 page 쪽 dimension, 2단계는 date 쪽 dimension을 각각 독립적으로 준비하는 과정. 아직 fact_visit과는 무관.

**실행 결과**

```
date_with_month_and_quarter.show()

date_id  | full_date  | month | quarter
---------+------------+-------+---------
20240701 | 2024-07-01 |     7 | Q3
20240702 | 2024-07-02 |     7 | Q3

>>> 저장 완료. output_date_dir 경로에 Delta 파일 생성됨.
```


### 3단계: 원본 visits JSON 읽고, fact_visit 생성용 컬럼 계산
```python
visits_dataset = (spark_session.read
    .schema('visit_id STRING, event_time TIMESTAMP, page STRING')
    .format('json').load(input_visits_dir))

fact_visit = (visits_dataset.selectExpr(
    'visit_id',
    'HASH(page) AS dim_page_id',
    'HASH(TO_DATE(event_time)) AS dim_date_id',
    'DATE_FORMAT(event_time, "HH:mm:ss") AS event_time'
))
```
- 이 조각이 하는 일: 원본 visit 로그(JSON)를 읽어서, 그 값을 기준으로 `dim_page_id`, `dim_date_id`라는 참조 키를 새로 만들어 fact 테이블을 구성한다.
- 핵심 문법
    - `.schema('...')` — 읽을 때 컬럼명과 타입을 명시적으로 지정. JSON은 스키마 추론(infer)에 비용이 들기 때문에 미리 스키마를 박아주는 것.
    - `.selectExpr(...)` — SQL 표현식 문자열을 그대로 넣어서 컬럼을 선택/계산하는 메서드. `select()` + SQL 문법 혼합 버전.
    - `HASH(page)` — page 문자열 값을 해시 함수로 정수 ID로 변환. dim_page 쪽의 실제 `page_id`와 나중에 매칭시키기 위한 인위적인 키 생성 방식. (양쪽에서 동일한 HASH 로직을 써야 값이 맞아떨어진다.)
    - `TO_DATE(event_time)` — timestamp에서 날짜 부분만 추출. 날짜만 남겨서 dim_date와 매칭할 키를 만드는 용도.
    - `DATE_FORMAT(event_time, "HH:mm:ss")` — timestamp에서 시:분:초 문자열만 추출해 별도 컬럼으로 보존.

**실행 결과**
```
visits_dataset.show()  (원본)

visit_id | event_time            | page
---------+------------------------+-----------
       1 | 2024-07-01T09:00:00Z  | home.html

fact_visit.show()  (가공 후)

visit_id | dim_page_id | dim_date_id | event_time
---------+-------------+-------------+------------
       1 |  -845123001 |   582910447 |   09:00:00

※ dim_page_id, dim_date_id는 HASH() 결과값 — 실제로는 위 값과 다른 임의의 정수가 나옴 (해시 함수 특성상 예측 불가한 정수)
```

### 4단계: fact_visit 저장
```python
fact_visit.write.mode('overwrite').format('delta').save(output_visits_dir)
```
- 이 조각이 하는 일: 3단계에서 만든 fact_visit을 디스크에 저장. 이 시점에 star schema를 이루는 3개 테이블(dim_page, dim_date, fact_visit)이 모두 준비 완료.

**실행 결과**
```
>>> 실행 완료. 반환값 없음.
>>> 디스크 상태: output_visits_dir 경로에 Delta 파일 생성됨
```


### 5단계: 분석가의 조회 — join이 여전히 필요 (One Big Table과의 결정적 차이)
```python
fact_visit = spark_session.read.format('delta').load(output_visits_dir)
dim_date = spark_session.read.format('delta').load(output_date_dir)
dim_page = spark_session.read.format('delta').load(output_page_dir)

full_visit = (fact_visit
    .join(dim_date, fact_visit.dim_date_id == dim_date.date_id, 'left_outer')
    .join(dim_page, [fact_visit.dim_page_id == dim_page.page_id], 'left_outer'))
```

- 이 조각이 하는 일: 저장해둔 3개 테이블을 각각 읽어서, 조회 시점에 join으로 다시 합친다.
- One Big Table과 비교되는 핵심 지점: One Big Table은 조회에 join이 0번이었는데, star schema는 조회할 때도 join이 2번 남아있다. 대신 쓰기 시점에서 fact와 dimension을 아예 합쳐버리지 않았으므로, dim_page나 dim_date 값이 바뀌어도 해당 테이블 1곳만 업데이트하면 된다 — One Big Table보다 업데이트 비용이 싸다.

**실행 결과**

```
full_visit.show()  → 1개 row 결과

visit_id       : 1
dim_page_id    : -845123001
dim_date_id    : 582910447
event_time     : 09:00:00
date_id        : 582910447
full_date      : 2024-07-01
month          : 7
quarter        : Q3
page_id        : -845123001
page_name      : home.html
category_name  : landing
```

### 최종 요약
```
한 줄 요약: page/date 관련 dimension은 미리 합쳐서 별도 저장하고, fact는 참조 키만 들고 별도 저장한 뒤, 조회 시점에 fact-dimension만 join한다.

실행 순서: 1단계(dim_page 준비/저장) → 2단계(dim_date 준비/저장) → 3단계(fact 계산) → 4단계(fact 저장) → 5단계(조회 시 join 2회)

핵심 라인: 3단계의 HASH(page) AS dim_page_id — One Big Table처럼 완전히 합치지 않고, "참조 키만 계산해서 넣는다"는 star schema 특유의 지점.
```


## (5)최신트렌드

**목적: 요즘 실무에서 뭘 쓰고, 왜 그게 더 낫다고 느끼는지.** 메커니즘 나열이 아니라 체감 이점 위주.

### 1. Iceberg / Delta Lake의 Nested Schema Evolution
- 정체: STRUCT 내부에 필드를 자유롭게 추가/삭제/이름변경할 수 있게 해주는 lakehouse 테이블 포맷 기능
- 이전 한계: 예전 Hive/Parquet-only 환경에서는 STRUCT 내부 필드 하나 추가하려면 테이블 전체를 재작성하거나, 심하면 새 테이블을 파야 했다
- 요즘 쓰는 이유
> STRUCT를 실무에서 안 쓴 이유가 사실 "나중에 필드 하나 추가하기 무섭다"였다. 
  Iceberg는 스키마 변경을 메타데이터 레벨에서 처리해서, `device` STRUCT 안에 `device_os_version` 필드 하나 추가해도 기존 데이터 재작성 없이 바로 반영된다. 이게 되니까 이제 STRUCT를 OBT 설계에 훨씬 편하게 넣는다. 예전엔 "필드 나중에 바뀌면 골치아프다"는 이유로 무조건 flat 컬럼으로만 갔는데, 지금은 그 걱정이 많이 줄었다.
 
→ 별도의 메타데이터 레이어(로그처럼 append-only로 쌓이는 구조)에서 스키마 변경 이력을 관리하는 방식이다

### 2. dbt의 wide table / mart 레이어 자동화
- 정체: SQL 기반 변환(transform) 도구. 정규화된 원본(staging) 위에 star schema나 OBT(mart)를 코드로 정의하고 자동 실행하는 프레임워크
- 이전 한계: 예전엔 이 join 파이프라인을 Spark 스크립트나 Airflow DAG로 사람이 직접 짜고 관리했다. join 순서, 의존성, 재실행 로직을 다 수동으로 챙겨야 했다
- 요즘 쓰는 이유
> dbt 쓰면 지금 이 세션에서 짠 것 같은 join 체인을 SQL 파일 하나로 선언만 하면 된다. `ref()`로 staging 테이블을 참조하면 dbt가 알아서 의존성 순서(DAG)를 계산해서 실행한다. 예전엔 "이 배치가 dim_page 먼저 돌고 fact가 그다음 돌아야 하는데" 이런 순서를 직접 Airflow에서 관리했는데, 이제 그게 훨씬 선언적으로 바뀌었다. star schema/OBT 만드는 파이프라인 자체가 유지보수하기 쉬워진 게 체감 이점.


### 3. BigQuery / Databricks의 Materialized View
- 정체: 쿼리 결과를 자동으로 캐싱해서 저장해두고, 원본 데이터가 바뀌면 증분(incremental)으로만 갱신하는 기능
- 이전 한계: 예전엔 이 문서에서 본 것처럼 "join 결과를 매일 배치로 전체 재작성"하는 방식이 기본이었다. 데이터가 조금만 바뀌어도 전체를 다시 join해야 했다
- 요즘 쓰는 이유
>  `write.mode('overwrite')`로 매번 전체를 다시 만드는 대신, materialized view는 바뀐 부분만 증분으로 갱신한다. 
>  예전엔 "OBT 하나 갱신하는데 30분씩 걸린다"가 흔한 불만이었는데, 
>  증분 갱신되면 몇 분으로 줄어든다. 다만 이게 모든 쿼리 패턴에 다 맞는 건 아니라서, 여전히 배치 join 파이프라인을 직접 짜는 경우도 많다 — materialized view는 쿼리가 비교적 단순하고 예측 가능할 때 유리하다.

### 실무 선호 정리
- **Iceberg/Delta 위에 STRUCT 쓰는 OBT**: 스키마가 계속 진화할 걸 아는 상황 (product가 계속 커지는 스타트업 등)
- **dbt 기반 star schema mart**: 여러 팀이 같이 쓰는 성숙한 데이터 웨어하우스, 거버넌스와 재사용성이 중요할 때
- **materialized view**: 쿼리 패턴이 고정적이고, 갱신 지연을 최소화해야 할 때 (대시보드 등)

### 엔지니어 독백
> 지금 이 3개(Iceberg, dbt, materialized view) 다 결국 하는 일은 똑같다 — "join 비용을 조회에서 쓰기로 옮긴다"는 Denormalizer의 본질은 안 바뀌었다. 달라진 건 그 join 파이프라인을 얼마나 편하게, 얼마나 적은 재작성 비용으로 유지보수하냐는 도구적인 부분이다.
> 
> 신입 때는 "요즘은 다 자동화됐으니 Denormalizer 원리를 몰라도 되지 않나" 싶을 수 있는데, 반대다. dbt든 materialized view든 결국 내부적으로 저 join 로직을 실행하는 거라, 뭐가 느린지 왜 느린지 디버깅하려면 지금 배운 원리를 알아야 한다. 도구는 바뀌어도 join 비용을 어디로 옮기는지에 대한 판단은 엔지니어 몫이다.


- 01.STRUCT 타입
- 02.star스키마 vs snowflake 스키마



# 01.Struct 타입

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




<br><br><br><br><br><br>





# 02.Star스키마 vs snowflake 스키마 비교

- Star schema : 1번 join으로 끝나도록 반정규화 함
- Snowflake schema : 정규화 끝까지 함 
	- snowflake DW와 상관없음




## Star Schema vs Snowflake Schema — 테이블 설명 포함 재작성

### 공통 테이블: fact_visit, dim_date

- **fact_visit**: visit이 발생할 때마다 한 row씩 쌓이는 사실 테이블. 언제, 어느 페이지에서 방문이 일어났는지를 참조 키(ID)로 기록한다.
- **dim_date**: 날짜 관련 속성(월, 분기 등)을 담은 dimension 테이블. fact_visit이 이 테이블을 참조한다.

두 테이블은 star든 snowflake든 동일하다.

```
SELECT * FROM fact_visit;

 visit_id | dim_date_id | dim_page_id | visit_time
----------+-------------+--------------+-----------
        1 |    20240701 |          100 | 09:00:00

SELECT * FROM dim_date;

 date_id  | full_date  | month | quarter
----------+------------+-------+---------
 20240701 | 2024-07-01 |     7 | Q3
```

### 갈라지는 지점: dim_page 이후

- **dim_page**: 페이지 관련 속성(페이지명, 카테고리 등)을 담은 dimension 테이블. fact_visit이 이 테이블을 참조한다. star와 snowflake의 차이가 여기서부터 생긴다.
- **dim_page_category**: 페이지 카테고리 이름을 담은 dimension 테이블. snowflake schema에만 별도로 존재하고, dim_page가 이 테이블을 참조한다.
- star schema : `dim_page` 테이블에서 다 때려박기(반정규화), 조인 1번으로 끝냄
- snowflake schema : `dim_page`, `dim_page_category`로 정규화 끝장보기 

```
[STAR SCHEMA]                          [SNOWFLAKE SCHEMA]

dim_page 테이블                          dim_page 테이블
SELECT * FROM dim_page;                 SELECT * FROM dim_page;

 page_id | page_name | category_name    page_id | page_name | page_category_id
---------+-----------+---------------   --------+-----------+-------------------
     100 | home.html | landing              100 | home.html |                 5

(더 이상 참조할 테이블 없음)              dim_page_category 테이블 (별도 존재)
                                         SELECT * FROM dim_page_category;

                                      page_category_id | category_name
                                    -------------------+---------------
                                                     5 | landing
```

### 구조도 나란히 비교

```
[STAR SCHEMA]                          [SNOWFLAKE SCHEMA]

dim_date                                dim_date
   \                                       \
    fact_visit                              fact_visit
   /                                       /
dim_page ── (끝, 더 참조 없음)          dim_page ──▶ dim_page_category
```

### join 흐름 나란히 비교 — category_name 하나 얻는 데 걸리는 join 수

```
[STAR SCHEMA]                                    [SNOWFLAKE SCHEMA]

fact_visit.dim_page_id(100)                      fact_visit.dim_page_id(100)
  ──join 1회──▶ dim_page.page_id(100)               ──join 1회──▶ dim_page.page_id(100)
  결과: category_name 즉시 확보                       dim_page.page_category_id(5)
                                                     ──join 1회──▶ dim_page_category.page_category_id(5)
  총 join 1회                                        결과: category_name 확보
                                                     총 join 2회
```

### 최종 비교표

```
항목                         | Star Schema              | Snowflake Schema
------------------------------+---------------------------+---------------------------
소속 패턴                     | Denormalizer (조회속도 우선)| Normalizer (일관성 우선)
dim_page 안 category_name    | 있음 (flatten됨)          | 없음, page_category_id만
dim_page_category 테이블 존재 | 없음                      | 있음
category_name 조회 join 횟수 | 1회                       | 2회
category_name 중복 저장       | 있음 (page마다 반복)       | 없음 (category당 1행)
category_name 수정 비용       | dim_page의 관련 row 여러 개| dim_page_category 1행만
```

### 엔지니어 독백

> star는 "조회할 때 빠르게"가 목적이라 category까지 dim_page 안에 미리 붙여놨다. 대신 category_name이 바뀌면 dim_page의 관련 row를 다 찾아 고쳐야 한다.
> 
> snowflake는 반대로 "수정할 때 한 곳만"이 목적이다. category_name은 dim_page_category 딱 한 곳에만 있으니 거기만 고치면 끝. 대신 조회할 때마다 join이 하나 더 붙는다.
> 
> 실무에서는 dimension의 row 개수로 갈린다. dim_page_category처럼 row가 몇십~몇백 개뿐이고 잘 안 바뀌는 건 그냥 star로 펴서 넣는다. join 하나 아끼는 게 이득이 더 크다.



### 언제 뭘 쓰나 — 실무 기준

> 주니어 때 내가 헷갈렸던 지점이 이거였다. "정규화가 이론적으로 더 깔끔한데 왜 다들 star를 쓰지?" 답은 간단하다. **분석 쿼리는 읽기(read)가 압도적으로 많고, 그 읽기 속도가 곧 분석가의 생산성이다.** join 2번, 3번 걸리는 snowflake는 분석가 입장에서 매번 고통이다.
> 
> 그래서 실무 기본값은 star다. dimension 테이블 몇 개 안 되고(수십~수백 row), 자주 안 바뀌는 건 그냥 펴서 넣는다. Kimball 방법론(데이터 웨어하우스 모델링의 사실상 표준)도 star를 기본으로 밀고, Looker/Tableau 같은 BI 툴도 star가 있다고 가정하고 동작한다.
> 
> Snowflake는 반대로 아주 정교한 계층 데이터(예: 조직도, 상품 카테고리 트리처럼 깊이가 있는 구조)를 다룰 때, 혹은 update가 자주 일어나는 dimension을 한 곳에서만 관리해야 할 때 쓴다. 근데 이것도 요즘은 굳이 snowflake로 안 가고 다른 방법으로 우회하는 추세다.

### 좀 더 진보된 방식 — 요즘 실무에서 쓰는 것

> star도 결국 "join이 몇 번이냐"의 트레이드오프일 뿐이지, 근본적으로 "정규화냐 비정규화냐" 이분법 안에 갇혀 있다. 요즘 컬럼형 스토리지(Parquet, Delta Lake, Iceberg 같은)에서는 여기서 한 단계 더 나간다.
> 
> **STRUCT/ARRAY 같은 중첩 타입을 쓰는 One Big Table.** dim_page 정보를 별도 테이블로 안 두고, fact_visit 안에 `page STRUCT<page_id, page_name, category_name>` 처럼 통째로 박아 넣는다. join 자체가 아예 없다. 컬럼형 포맷은 이런 중첩 구조를 읽을 때도 필요한 필드만 골라 읽을 수 있어서(컬럼 프루닝), star schema처럼 컬럼이 다 펴져 있을 때와 성능 차이가 거의 안 난다.
> 
> 그래서 요즘 흐름은 이렇다: **원본은 정규화(snowflake 또는 3NF)로 안전하게 유지하고, 그 위에 배치/스트리밍 파이프라인으로 One Big Table을 파생시켜 분석가에게 노출한다.** 데이터 일관성은 원본 레이어가 책임지고, 조회 속도는 파생 레이어가 책임지는 식으로 관심사를 분리하는 거다. 이게 Normalizer + Denormalizer를 같이 쓰는 실무 패턴이고, 지금 우리가 배우는 두 패턴이 왜 "배타적이지 않다"고 하는지 이유가 여기 있다.

**한 줄 팁**: 주니어 때는 "지금 짜는 테이블이 분석가가 자주 조회할 테이블이냐, 아니면 시스템 내부 원본 데이터냐"부터 물어봐라. 전자면 star나 OBT, 후자면 정규화. 이게 90%는 답을 준다.


## Star vs Snowflake — 등장 순서와 Snowflake(DW) 연관성

### 등장 순서: Star가 먼저다

> 순서가 사용자 예상과 반대다. **Star schema가 먼저 나왔고, Snowflake schema는 그 이후에 등장한 변형이다.**
> 
> Star schema는 1990년대 Ralph Kimball이 데이터 웨어하우스 차원 모델링(dimensional modeling) 방법론을 정립하면서 표준으로 자리잡았다. 목적이 처음부터 "분석가가 빠르게 조회하게 하자"였기 때문에, 처음부터 dimension을 얕게(1단계) 설계했다.
> 
> Snowflake schema는 star의 파생형으로, dimension 테이블도 정규화 원칙(중복 제거, 저장 공간 절약)을 적용하고 싶은 사람들이 dimension을 더 잘게 쪼갠 것이다. "star가 불편해서 star 다음에 star를 대체하러 나온 게 아니라", star를 좀 더 저장 공간 효율적으로 만들고 싶을 때 선택하는 대안으로 나온 것에 가깝다. 대체 관계가 아니라 **트레이드오프가 다른 선택지**로 같이 존재해온 것.

### Snowflake(DW 제품)와 Snowflake schema — 연관성 있나?

> 없다. 순전히 이름만 겹치는 우연이다.
> 
> Snowflake schema라는 용어는 1990년대 데이터 웨어하우스 모델링 이론에서 나온 것이고, dimension 테이블들이 눈송이(snowflake)처럼 가지를 뻗는 모양이라서 붙은 이름이다.
> 
> Snowflake(회사, 클라우드 DW 제품)는 2012년에 설립된 회사고, 이름을 그 눈송이 모양 콘셉트에서 따왔을 가능성은 있지만(회사 자체가 눈 관련 브랜딩을 종종 씀), **기술적으로 Snowflake 제품이 snowflake schema를 강제하거나 그 구조로 동작하는 건 전혀 아니다.** Snowflake 안에서 star schema를 쓰든 snowflake schema를 쓰든 OBT를 쓰든 자유고, 오히려 실무에서는 Snowflake(제품) 위에서도 star나 OBT가 훨씬 많이 쓰인다.

**한 줄 정리**: 순서는 star → snowflake(변형). Snowflake(제품)는 이름만 같을 뿐 snowflake schema와 기술적 연관은 없다.





## 원본 정규화 + 파생 OBT 구조 — 실제 예시

**도메인: 이커머스 주문(order) 데이터.** raw 레이어는 정규화로, 분석용 레이어는 OBT로 분리하는 흐름을 실제 테이블로 보여준다.

### 1단계: 원본 레이어 (정규화, 시스템이 직접 쓰는 테이블)

- **orders**: 주문 하나당 한 row. 결제/배송 시스템이 직접 insert/update하는 원본 테이블.
- **customers**: 고객 정보. 고객 정보가 바뀌면(주소 변경 등) 여기 딱 1행만 고치면 된다.
- **products**: 상품 정보. 가격이나 상품명이 바뀌면 여기 1행만 고치면 된다.

```
SELECT * FROM orders;

 order_id | customer_id | product_id | order_time          | quantity
----------+-------------+------------+----------------------+---------
     5001 |         900 |         77 | 2026-08-20T10:00:00Z |        2

SELECT * FROM customers;

 customer_id | customer_name | email
--------------+----------------+------------------
          900 | Jane Kim       | jane@example.com

SELECT * FROM products;

 product_id | product_name | price
------------+---------------+-------
         77 | Wireless Mouse|  29.99
```

- 이 3개 테이블은 시스템(주문 서비스, 결제 서비스)이 실시간으로 insert/update하는 곳이라, **정합성이 최우선**이다. product 가격이 바뀌면 products 딱 1행만 고치면 모든 orders가 자동으로 최신 가격을 참조한다.

### 2단계: 파생 레이어를 만드는 배치 파이프라인

- **orders_flat_daily_batch (Airflow DAG 등)**: 매일 새벽, 원본 3개 테이블을 join해서 분석용 OBT를 새로 만들어 쓰는 배치 잡.

```
파이프라인 흐름 (하루 1번 실행):

orders ─┐
customers ─┼─ join 3개 테이블 ─▶ orders_flat 테이블에 overwrite 저장
products ──┘
```

- 이 join 비용은 하루 딱 1번만 발생한다. 분석가가 오늘 하루 몇 번을 조회하든 이 비용은 다시 지불하지 않는다.

### 3단계: 분석 레이어 (OBT, 분석가가 조회하는 테이블)

```
SELECT * FROM orders_flat;

 order_id | customer_name | email             | product_name    | price | quantity | order_time
----------+-----------------+-------------------+------------------+-------+----------+----------------------
     5001 | Jane Kim        | jane@example.com  | Wireless Mouse   | 29.99 |        2 | 2026-08-20T10:00:00Z
```

- 분석가는 Looker/Tableau에서 이 테이블 하나만 붙여서 join 없이 바로 "고객명별 매출", "상품별 판매량" 같은 걸 조회한다.

### 관심사 분리 — 누가 뭘 책임지나

```
              [정합성 책임]                        [조회 속도 책임]
                    │                                    │
     orders / customers / products    ──batch join──▶  orders_flat
     (시스템이 직접 쓰는 원본)          (하루 1번,          (분석가가 매일 수백 번
                                        비용 지불)           조회, join 없음)
```

### 엔지니어 독백

> 만약 orders_flat을 시스템이 직접 쓰게 하면 어떻게 될까 상상해봐라. product 가격 하나 바뀔 때마다, 그 product가 들어간 orders_flat의 row를 전부 찾아서 UPDATE 쳐야 한다. 수백만 건이 될 수도 있는 row를. 끔찍하다.
> 
> 그래서 시스템(쓰기 트래픽이 많은 쪽)은 무조건 정규화된 원본을 건드리게 하고, 분석가(읽기 트래픽이 많은 쪽)는 절대 원본을 직접 join하지 못하게 파생 테이블만 보게 만든다. 이 파생 테이블이 하루 지난 데이터라도(배치 주기 때문에) 대부분의 분석 케이스에서는 문제가 안 된다 — 어제 매출 보는데 1시간 지연이 무슨 상관이겠나.
> 
> 실시간성이 필요하면 배치 대신 Structured Streaming으로 이 파이프라인을 돌리면 되고, 구조 자체는 똑같다. "원본은 정규화, 파생은 비정규화"라는 원칙만 지키면 된다.



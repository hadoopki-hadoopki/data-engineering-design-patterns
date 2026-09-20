**Chapter 10: 데이터 관찰성 (Data Observability) 디자인 패턴**

| 섹션                         | 패턴                              | 페이지     |
| -------------------------- | ------------------------------- | ------- |
| 10.1 관찰성 (Observability)   | Flow Interruption Detector(#65) | 308~311 |
| 10.1 관찰성 (Observability)   | Skew Detector(#66)              | 312~315 |
| 10.1 관찰성 (Observability)   | Lag Detector(#67)               | 316~319 |
| 10.2 데이터 계보 (Data Lineage) | Dataset Tracker(#68)            | 320~326 |
| 10.2 데이터 계보 (Data Lineage) | Fine-Grained Tracker(#69)       | 327~330 |

**이전 챕터와의 연결점**

- Chapter 9는 "데이터 자체"가 정확한지(스키마, 값 품질)를 다룸
- Chapter 10은 "파이프라인이 애초에 정상적으로 돌고 있는가"를 다룸 — 데이터 값은 멀쩡해도, 파이프라인 흐름이 끊기거나 늦어지는 문제는 Chapter 9 패턴들로 못 잡음


# 패턴#66 스큐 탐지기 (Skew Detector)

**결론: 이전 실행과 지금 실행의 데이터 볼륨을 비교한다. 차이가 크면 막는다.**(급증,급락)

전체 흐름 먼저 그림으로 보자:

```
[D-1 파티션]              [D 파티션]
 file_size = 120MB   vs   file_size = 55MB
        \                    /
         size_ratio = 55 / 120 = 0.458
                    |
        0.458 < 0.5(임계치) → Exception 발생
                    |
        compare_volumes 태스크 실패 → transform_file, load 태스크 실행 안 됨
```

## (1) 문제상황

- 핵심 고통: 
	- 반쪽짜리 데이터셋이 그대로 처리됨
- 패턴#65 흐름 중단 탐지기(Flow Interruption Detector, 마지막 커밋 시각으로 흐름 끊김만 감지하는 패턴)는 이 상황을 통과시킴
- 이유: 
	- 흐름 자체는 있었음. 줄어든 건 볼륨뿐
- 실제 상황
    - 원래 하루치 파일 크기: 120MB
    - 오늘 들어온 파일 크기: 55MB
    - 원인: 데이터 제공사 쪽 생성 오류
    - 임계치가 "±50%" 이런 식으로 양방향
- 그래서 힘들어지는 지점
    - 잡은 성공으로 끝남
    - 로그도 정상
    - 하지만 컨슈머 집계 결과는 반토막
    - 다운스트림에서 항의 들어옴


## (2) 솔루션

**주요컨셉:** 
**비교 기준 시점을 정하고, 그 사이 데이터양 차이에 허용 범위를 두고, 범위를 벗어나면 막는다.**

3단계로 나뉘어.

1. 비교 윈도우 정하기
    - 어떤 시점끼리 비교할지 결정
    - 예: 일배치면 "오늘 vs 어제" 파티션
2. 허용 임계치 정하기
    - 예: 50%로 잡으면 전날 대비 ±50% 이내는 정상
    - 과거 변동폭 데이터를 보거나, 비즈니스 담당자한테 직접 물어서 정함
3. 계산 방식 선택 (둘 중 하나)
    - **윈도우 대 윈도우 비교**
        - 두 시점 값을 그냥 비율로 나눔
        - 예: `size_ratio = 오늘_파일크기 / 어제_파일크기`
        - 배치, 스트리밍 둘 다 적용 가능
    - **표준편차 비율(stddev/avg)**
        - 파티션마다 평균 대비 얼마나 벗어났는지 계산
        - 파티션 구조를 가진 저장소(Kafka 토픽, PostgreSQL 파티션 테이블)에 유용
        - 이런 저장소는 보통 `STDDEV`, `AVG` 함수를 기본 제공해서 바로 계산식에 넣을 수 있음

- 위치 팁
    - 스큐 탐지기는 Audit-Write-Audit-Publish(입력·출력 데이터셋의 필수 컬럼 값을 검증해 실패 시 파이프라인을 중단시키는 패턴)의 **1차 Audit 단계 가드**로 넣기 좋음
    - 데이터가 온전한 상태인지부터 걸러주는 역할



## (3) 결과

패턴을 적용해도 실무에서는 세 가지 까다로운 지점이 남아.

### **1. 시즌성(Seasonality)**

- 임계치가 항상 맞는 건 아님
- 마케팅 캠페인 진행 중이면 평소보다 50% 더 많은 레코드가 정상적으로 들어올 수 있음
- 여름/겨울처럼 계절 영향받는 비즈니스도 동일한 문제 발생
- 해결 방향: 비즈니스 지식으로 비교 공식을 짜고, 변동 폭 큰 특정 기간은 예외 처리

### **2. 커뮤니케이션 문제**

- 임계치를 잘 잡아도 False Positive(가짜 경보) 여지는 남음
- 예: 성공적인 마케팅 캠페인 하나로 임계치를 초과하는 데이터가 유입될 수 있음
- 이건 기술로 못 풀어
- 타 부서와 사전 커뮤니케이션으로 풀어야 하는 문제

### **3. Fatality loop(연쇄 실패)**

```
D일:   정상 120MB → 오류로 40MB만 생성
       size_ratio = 40/120 = 0.33 → 실패

D+1일: 정상 복구, 120MB 생성
       하지만 비교 대상이 D일(40MB)
       size_ratio = 120/40 = 3.0 → 또 실패
```

- D일에 데이터가 줄어서 스큐 검증 실패
- D+1일에 정상 데이터가 들어와도, D일 대비 3배가 됐으니 또 "스큐"로 판정
- 근본 해결: 데이터 제공사가 D일 이슈를 고치는 것
- 급하면 비교 대상을 "바로 전날"이 아니라 **가장 최근 성공한 실행**으로 바꿔서 회피



> 발생을 감지만하고 원인은 따로 찾아야 하겠구나 너무 다양해서
> 
- 원인은 케이스마다 다 다름
    - 데이터 제공사 생성 오류
    - 마케팅 캠페인으로 인한 정상 증가
    - 배치 스케줄 자체가 밀린 경우
    - 상류 파이프라인이 일부만 돌고 끝난 경우
- 그래서 실무에서는 항상 이렇게 이어짐
    - 스큐 탐지기 → 알람만 울림
    - 알람 뜨면 사람이 원인 파악 (로그 확인, 담당 부서 문의, 데이터 제공사 확인)
    - 원인 확정되면 그때 대응 (재처리, 예외 처리, 임계치 조정)
- 즉 이 패턴의 역할 범위는 "이상 여부 판정"까지고, "원인 규명 + 대응"은 사람 몫




## (4)예시

> 방문 로그(visits)를 다루는 팀이 있어. 
> JSON 파일이 하루 단위로 들어오고, 
> Airflow DAG가 이걸 읽어서 PostgreSQL 테이블에 적재해. 
> 여기에 스큐 탐지기를 3가지 방식으로 붙여볼게.


> 세 예시는 같은 패턴이지만 비교하는 영역이 서로 달라.

```
[같은 시점, 파티션끼리 비교]          [다른 시점, 전체 볼륨끼리 비교]

기술1: PostgreSQL  ─┐
                    ├─ 같은 영역, 저장소만 다름
기술2: Kafka       ─┘

기술3: Airflow ─────────────────────────── 비교하는 영역 자체가 다름
```


### 기술1: PostgreSQL — 파티션 간 불균형 감지

앞서 말한 `visits` 테이블은 실제로는 날짜별 여러 파티션(`visits_all_range_1`, `_2`, `_3`...)으로 쪼개져서 저장돼.

이 예시로 확인할 것: 같은 시점 안에서, 파티션 개수와 무관하게 쏠림 하나를 잡아내는 표준편차 방식. 이게 기술2, 기술3의 기준이 됨.

**1단계.**

- 파티션이 늘어날수록 "1번 vs 2번", "1번 vs 3번" 식 짝비교는 조합이 폭발함
- 전체 평균만 보면, 파티션 하나가 텅 비어도 나머지가 커버해서 정상처럼 보임

**2단계.**

- 파티션 몇 개든 흩어진 정도를 숫자 하나로 요약할 방법이 필요
- 그 방법이 표준편차(STDDEV)

**3단계.**

```
정상: 10,000 / 10,200 / 9,800 / 10,000
→ AVG=10,000, STDDEV≈141, 비율≈1.4%

이상: 10,000 / 10,200 / 9,800 / 500
→ AVG=7,625, STDDEV≈4,116, 비율≈54%
```

**4단계.**

한 줄 요약: `pg_stat_user_tables`(PostgreSQL 내장 테이블 통계 뷰)에서 파티션별 row 수를 모아 stddev/avg 비율을 계산하는 쿼리

```sql
SELECT
  NOW() AS "time",
  (STDDEV(n_live_tup) / AVG(n_live_tup)) * 100 AS value
FROM pg_catalog.pg_stat_user_tables
WHERE relname != 'visits_all_range'
  AND relname LIKE 'visits_all_range_%';
```

★ 핵심: `STDDEV(n_live_tup) / AVG(n_live_tup)` — n_live_tup(파티션별 row 수)의 편차를 평균으로 나눠서, 파티션 개수와 무관한 비율값 하나로 만듦


### 기술2: Apache Kafka — 브로커 파티션 쏠림 감지

> 앞서 본 팀이 방문 로그를 Kafka `visits` 토픽으로도 흘려보낸다고 해보자.
> 이 토픽도 여러 파티션에 데이터가 분산 저장돼.
> 이 예시로 확인할 것: 기술1과 비교하는 영역은 완전히 같아. 같은 시점, 파티션끼리.
> 저장소만 PostgreSQL에서 Kafka로 바뀌었을 때, 계산식이 그대로 재사용되는지 확인하는 자리야.

**1단계.**
- 프로듀서가 파티셔닝 키를 잘못 잡으면 특정 파티션 하나에만 데이터가 몰릴 수 있음
- 몰린 파티션은 컨슈머가 처리 지연을 겪고, 나머지 파티션은 놀게 됨
- 기술1과 똑같은 문제 구조. 다만 데이터가 테이블 통계가 아니라 Prometheus 메트릭(`kafka_log_size`)으로 존재한다는 차이만 있음


**2단계.**
- 필요조건도 기술1과 동일 — 표준편차/평균 비율
- 용어부터 정리: `kafka_log_size{topic='visits'}`는 Kafka 브로커가 Prometheus로 내보내는 메트릭. `{topic='visits'}`는 라벨 필터로 visits 토픽 값만 골라냄
- `sum(...) by (partition)`은 파티션별로 그룹 지어 합산하는 함수
- `stddev(...)`, `avg(...)`는 기술1에서 본 것과 동일한 표준편차, 평균 함수


**3단계.**
실행 순서:
```
1. kafka_log_size{topic='visits'}로 원본 메트릭 값들을 가져옴 (파티션별로 여러 개)
2. sum(...) by (partition)으로 파티션별 하나의 숫자로 묶음
3. 묶인 값으로 stddev(...) 계산, 별도로 avg(...) 계산
4. 편차값 / 평균값 → 비율, *100으로 퍼센트 변환
```


**4단계.**
한 줄 요약: Kafka `visits` 토픽의 파티션별 쏠림 정도를 Prometheus에서 조회하는 쿼리(PromQL)
```promql
stddev(sum(kafka_log_size{topic='visits'}) by (partition))
/ avg(kafka_log_size{topic='visits'}) * 100
```


★ 핵심: `sum(...) by (partition)` — 이게 없으면 안 됨.
Prometheus는 파티션마다 값을 별도 시계열로 저장하기 때문에, 먼저 파티션 단위로 묶어야 stddev·avg가 계산 기준을 잡을 수 있음



### 기술3: Apache Airflow — 어제 대비 오늘 전체 볼륨 급변 감지

> 이제 그 팀의 원래 파이프라인, JSON 파일을 읽어 PostgreSQL에 적재하는 Airflow DAG로 돌아가자.
> 
> 이 예시로 확인할 것: 앞 두 예시(기술1, 기술2)와 비교하는 영역 자체가 다름. "같은 시점, 파티션끼리"가 아니라 "다른 시점, 전체 볼륨끼리".


**1단계.**
- 풀어야 할 문제: 어제 파일은 120MB였는데, 오늘 파일이 55MB로 들어옴
- 비교 대상이 파티션 여러 개가 아니라 딱 두 시점(어제, 오늘)뿐이라 표준편차를 쓸 이유가 없음


**2단계.**
- 필요한 건 그냥 두 시점 값의 비율
- 그 비율이 정해둔 범위(예: 0.5~1.5)를 벗어나면, 다음 단계(변환, 적재) 실행 전에 막아야 함
- 안 그러면 반쪽 데이터가 이미 테이블에 들어간 뒤에야 알게 됨


**3단계.**
- 시나리오: `next_partition_sensor`(FileSensor, 다음 파티션 파일 도착을 감시하는 센서)가 먼저 대기.
- 통과하면 `compare_volumes`(PythonOperator, 오늘 파일과 어제 파일 크기를 비교하는 태스크)가 실행됨.

이 코드가 하는 일: 이전 DAG 실행(`previous_dag_run`)의 파일 크기와 현재 실행의 파일 크기를 비교. 비율(`size_ratio`)이 0.5~1.5 범위를 벗어나면 예외를 발생시켜 파이프라인 중단.

```python
next_partition_sensor = FileSensor(...)

def compare_volumes():
    context = get_current_context()
    previous_dag_run = DagRun.get_previous_dagrun(context['dag_run'])
    if previous_dag_run:
        previous_execution_date = previous_dag_run.execution_date
        current_file_path = get_full_path(context['logical_date'], 'json')
        current_file_size = os.path.getsize(current_file_path)
        previous_file_path = get_full_path(previous_execution_date, 'json')
        previous_file_size = os.path.getsize(previous_file_path)
        size_ratio = current_file_size / previous_file_size
        if size_ratio > 1.5 or size_ratio < 0.5:
            raise Exception(f'Unexpected file size detected for the...')

volume_comparator = PythonOperator(
    task_id='compare_volumes',
    python_callable=compare_volumes
)
transform_file = PythonOperator(...)
load_flattened_visits_to_final_table = PostgresOperator(...)

(next_partition_sensor >> volume_comparator >> transform_file
 >> load_flattened_visits_to_final_table)
```

★ 핵심: `if size_ratio > 1.5 or size_ratio < 0.5: raise Exception(...)`

- 이 라인에서 예외가 나는 순간, 하위 태스크(`transform_file`, `load_flattened_visits_to_final_table`)는 실행 자체가 안 됨.
- Airflow의 태스크 의존성(`>>`)이 곧 "가드" 역할을 함.

### 예시 분류 요약표

|예시|비교하는 영역|저장소/도구|계산 방식|
|---|---|---|---|
|기술1 PostgreSQL|같은 시점, 파티션끼리|테이블 통계|stddev/avg|
|기술2 Kafka|같은 시점, 파티션끼리 (기술1과 동일)|Prometheus 메트릭|stddev/avg|
|기술3 Airflow|다른 시점, 전체 볼륨|파일 시스템 + DAG|size_ratio|

엔지니어 독백:
형, 이 세 예시를 계산식 세 개로 외우면 나중에 헷갈려.
"비교하는 영역이 두 가지고, 그 중 하나(파티션끼리 비교)를 저장소 두 개에서 재확인한 것"으로 묶어서 기억해.

그럼 새로운 저장소(예: MongoDB, Cassandra)를 만나도 "아 이것도 파티션 구조니까 stddev/avg 방식이 되겠구나"라고 바로 판단할 수 있어.
계산식을 외우는 게 아니라 "이게 어느 비교인지" 판단하는 게 실무에서 훨씬 중요해.



## (5)최신트렌드

결론: 요즘은 임계값을 손으로 정하지 않고, 과거 볼륨 이력을 학습해 동적 임계값을 만드는 데이터 관측성 도구에 맡기는 게 대세다. (3)결과에서 나온 계절성 문제를 정적 50% 룰로는 못 풀기 때문.

### 1. Elementary (dbt 네이티브 관측성 도구)

개요
- dbt 프로젝트에 패키지로 설치하는 오픈소스 관측성 도구
- 매 실행마다 테이블 row 수, freshness를 자동 수집

기존 방식의 한계
- 책 예시처럼 Airflow에 비교 함수를 직접 짜면 파이프라인마다 코드 중복 발생
- 임계값 50%가 하드코딩으로 박힘

도입 이유
- volume anomaly 테스트가 과거 이력의 평균·표준편차로 이상 범위를 자동 계산
- 요일별 계절성 옵션 지원, "월요일은 월요일끼리 비교" 가능
- dbt 쓰는 팀이면 YAML 몇 줄로 끝, 별도 인프라 불필요

### 2. Soda Core / Great Expectations (데이터 품질 프레임워크)

개요
- 파이프라인 중간에 검증 단계로 끼워 넣는 데이터 품질 검사 프레임워크

기존 방식의 한계
- compare_volumes 같은 커스텀 함수는 검사 로직·알림·리포트를 전부 직접 구현해야 함

도입 이유
- 선언형으로 검사를 정의함 (Soda의 row_count 변화율 체크, GX의 expect_table_row_count_to_be_between)
- 실패 시 파이프라인 중단과 알림까지 프레임워크가 처리
- AWAP의 Audit 단계에 그대로 꽂기 좋음

적합 상황
- dbt를 안 쓰는 팀
- Airflow/Spark 파이프라인에 검증 태스크로 삽입할 때

### 3. Monte Carlo, Datadog 등 (상용 관측성 SaaS)

개요
- 웨어하우스 메타데이터를 읽어 전체 테이블의 볼륨·freshness를 ML로 모니터링하는 상용 서비스

기존 방식의 한계
- 오픈소스 도구는 감시할 테이블을 하나하나 등록해야 함
- 테이블이 수백 개면 등록 관리 자체가 불가능해짐

도입 이유
- 연결만 하면 전체 테이블에 이상 탐지가 자동 적용됨
- 계절성·트렌드를 모델이 학습해 false positive 감소, 마케팅 캠페인 같은 급증에도 적응

적합 상황
- 테이블 수백 개 이상의 대규모 조직
- 관측성에 예산을 쓸 수 있는 곳

### 실무 선호 정리
- dbt 팀이면 Elementary가 사실상 표준, 도입 비용 최저
- dbt 없는 팀은 Soda Core 또는 GX를 Audit 단계로 삽입
- 대규모 조직은 Monte Carlo류 SaaS, 테이블 등록 관리가 일이 되는 시점부터
- 공통 방향은 정적 임계값에서 이력 기반 동적 임계값으로의 이동, 책의 표준편차 비율 계산을 도구가 자동화한 것

엔지니어 독백:
>50% 임계값 하드코딩으로 시작하면 처음 한 달은 잘 돌아. 
>그런데 블랙프라이데이에 알람 폭탄 맞고, 연휴에 또 맞고, 결국 알람을 꺼버리는 팀을 여럿 봤다. 
>알람은 꺼지는 순간부터 없는 거랑 같다. 
>그래서 요즘은 처음부터 Elementary 같은 걸로 요일별 이력 비교를 걸어두는 걸 추천해. 
>
>그리고 도구가 뭐든 간에 producer 팀이랑 슬랙 채널 하나 파두는 게 
>제일 싸고 효과 좋은 스큐 대응책이다. 
>캠페인 일정 공유받는 것만으로 false positive 절반은 줄어.







# 패턴#67 지연 탐지기 (Lag Detector)

- 챕터 10 데이터 관측성 중 시간 탐지기(Time Detectors) 그룹의 첫 패턴
- 시간 탐지기: 데이터가 아닌 시간(지연)을 지표로 처리 계층의 문제를 잡는 패턴 그룹

## (1)문제상황

결론: consumer가 producer보다 얼마나 뒤처져 있는지(lag)를 측정하지 않으면, 처리 속도 저하를 소비자 컴플레인으로 처음 알게 된다. 스케일링 판단의 근거 지표 자체가 없는 게 핵심 고통.

등장 컴포넌트:

- 데이터 producer: Kafka topic에 방문 이벤트를 넣는 업스트림 서비스
- Spark Structured Streaming 잡: 그 topic을 계속 읽어 처리하는 스트리밍 consumer
- 다운스트림 consumer: 처리 결과를 받아 쓰는 팀

발생 상황 (시간순):

1. 일주일 전부터 producer 측 데이터량이 30% 증가
2. 증가를 공지하는 이메일이 왔지만 놓침
3. 스트리밍 잡은 늘어난 입력을 기존 리소스로 처리 → 점점 뒤처짐
4. 다운스트림 팀 컴플레인: "데이터가 예전보다 늦게 도착해요"
5. 그제야 상황 파악, "이번이 마지막"이라고 약속한 상태

Pain point:

- 잡은 죽지 않고 돌아가니 에러 알람이 없음 → 지연은 조용히 누적됨
- 스케일링 전략을 세우고 싶어도 "지금 얼마나 뒤처졌는지" 수치가 없음
- 측정이 안 되면 스케일 아웃 시점도, 효과 검증도 불가능 → 지연 탐지기가 푸는 지점

용어 정리:

- lag: consumer가 마지막으로 처리한 지점과 스토리지에 존재하는 최신 지점의 차이
- lag가 커진다는 건 데이터 freshness 저하, 나아가 데이터 미가용의 전조 지표



## (2)솔루션

주요컨셉: 엔지니어가 "스토리지의 최신 지점 - consumer가 처리한 지점"의 차이를 lag 지표로 정의하고, 이를 Prometheus 같은 메트릭 저장소로 push해 알람 기준으로 삼는다.

전체 흐름:

```
producer ──▶ 스토리지 (Kafka topic / Delta 테이블)
                │
                ├─ 최신 지점: offset 5,000
                │
consumer ──▶ 처리 지점: offset 4,200
                │
                └─▶ lag = 5,000 - 4,200 = 800
                        │
                        ▼
                  모니터링 시스템 (알람 기준)
```

### 1단계. lag 단위 정의
- lag를 "무엇의 차이"로 잴지 결정하는 단계. 데이터 스토어마다 진행 상황을 기록하는 방식이 달라서 단위도 달라짐
- Kafka topic: 레코드 위치(offset) 또는 레코드 append 시각
- Delta Lake 테이블: commit 번호(version)
- 시간 파티셔닝된 스토어: 파티션 타임스탬프 (예: dt=2026-09-18까지 처리했는데 최신은 dt=2026-09-19)

### 2단계. 비교식 정의
- lag = 스토리지의 최신 단위 - consumer가 마지막으로 처리한 단위
- 실제 값 대입: 최신 offset 5,000, 처리 offset 4,200 → lag 800건
- Delta라면: 최신 version 152, 읽은 version 149 → lag 3 commit

### 3단계. 파티션별 결과 집계 전략
- Kafka처럼 파티셔닝된 스토어는 lag가 파티션마다 따로 나옴 → 하나의 지표로 합치는 전략 필요
- 선택지 두 가지:

1. MAX 집계
    - 파티션 중 최악의 lag 하나만 봄
    - 단 하나의 파티션만 뒤처져도 잡아냄 → 최악 시나리오 감시용
2. 백분위(P90, P95)
    - "파티션의 90%는 lag가 X 이하"를 보장하는 값
    - 전체적인 처리 성능 파악용. 단, 나머지 10%는 더 클 수 있음

- 실무 선호: 둘을 같이 씀. P90으로 전반적 추세를, MAX로 최악 케이스를 각각 감시

평균의 함정 (Average Trap):

- 7개 파티션 lag가 10, 5, 30, 2, 3, 5, 3초라고 하면
- 평균 = 8초 → "잘 돌고 있네"로 오판
- P90 = 18초 → 실제로는 90% 데이터가 18초까지 걸림
- 소수의 큰 lag를 평균이 희석시킴 → 관측성에서는 평균 대신 백분위를 쓰는 이유

정리:

|단계|결정할 것|예시|
|---|---|---|
|1. lag 단위|스토어별 진행 단위|Kafka offset, Delta version|
|2. 비교식|최신 - 처리 지점|5,000 - 4,200 = 800|
|3. 집계 전략|MAX / 백분위 / 병행|P90 + MAX 병행|

핵심 흐름 재확인: 단위를 정하고 → 차이를 계산하고 → 파티션 결과를 집계해 알람 기준으로 삼는다.



## (3)결과

결론: consumer의 처리 속도 저하를 컴플레인 전에 수치로 잡을 수 있게 되는 대신, lag 수치가 "consumer 탓"이 아닌 경우까지 consumer 문제처럼 보이는 착시를 감수해야 한다.

### 단점 1. 데이터 스큐가 만드는 착시

배경:

- 3단계에서 MAX 집계를 선택하면 최악의 파티션 하나가 전체 지표를 대표함
- 그런데 파티션별 lag 차이는 consumer 성능 말고 데이터 분포 때문에도 생김

문제:

- producer가 파티션 키를 잘못 잡아 특정 파티션에만 데이터가 쏠리면, consumer는 그 파티션을 당연히 느리게 처리함
- 실제 값 대입: partition 0~5는 lag 50건인데 partition 6만 lag 4,000건 → MAX 지표는 4,000
- 알람이 울리고 "consumer가 느리다"로 오판 → 스케일 아웃을 해도 파티션 6은 여전히 한 consumer가 처리하므로 해결 안 됨

대응:

- consumer를 키우는 게 아니라 쓰기 단계에서 데이터 분배를 고침
- 예: Kafka producer의 파티션 키를 쏠린 키(특정 대형 고객 ID)에서 더 고르게 퍼지는 키로 변경
- 진단 순서: MAX 알람 발생 → 파티션별 lag 분포 확인 → 한 파티션만 튀면 producer 쪽 분배 문제, 전체가 고르게 높으면 consumer 처리량 문제

엔지니어 독백:

lag 알람 받고 반사적으로 executor부터 늘리는 신입들이 많은데, 나도 그랬다. 파티션별 그래프를 먼저 봐라. 한 놈만 솟아 있으면 그건 니 잡 문제가 아니라 업스트림이 데이터를 몰아준 거다. 그때 스케일 아웃 해봤자 돈만 나가고 lag는 그대로다. Grafana 대시보드 만들 때 합계 지표 하나만 두지 말고 파티션별 분해 그래프를 꼭 옆에 붙여둬. 알람은 MAX로 받되, 원인 판단은 분포로 하는 거다.


### Lag가 발생하는 상황

> skew 말고 lag가 발생하는 상황은?

결론: 스큐 말고는 크게 4가지 계열이다 — 입력 증가, consumer 처리력 저하, sink 병목, 운영 이벤트. 실무에서 제일 흔한 건 consumer 자체가 아니라 sink(쓰기 대상) 병목이다.

### 1. 입력량 증가 (producer 쪽)
- 이 패턴의 문제상황 그 자체. 캠페인, 신규 서비스 연동, 업스트림 백필로 유입량 급증
- consumer는 그대로인데 들어오는 속도가 처리 속도를 넘어서면 lag는 선형으로 누적
- 특징: 파티션 전체가 고르게 lag 상승


### 2. consumer 처리력 저하
- 로직 변경으로 레코드당 처리 비용 증가 (예: 배포에서 외부 API 호출 추가 → 건당 5ms가 50ms로)
- JVM GC pause, 메모리 부족으로 인한 spill
- poison pill: 특정 레코드가 파싱 실패 → 재시도 루프에 갇혀 해당 파티션 진행 정지
- 특징: 배포 시점, 특정 레코드 도착 시점과 lag 상승 시점이 일치


### 3. sink 병목 (쓰기 대상 쪽)
- consumer가 읽고 변환은 빠른데 쓰는 곳이 못 받아주는 경우
- 예: 적재 대상 PostgreSQL에 락 경합, Elasticsearch 인덱싱 지연, S3 쓰기 시 small files 폭증으로 commit 시간 증가
- 스트리밍 프레임워크는 sink가 느리면 backpressure로 읽기 속도를 줄임 → 결과적으로 소스 lag 증가
- 특징: consumer CPU는 놀고 있는데 lag만 증가. 배치당 처리 시간 중 write 단계가 지배적


### 4. 운영 이벤트
- consumer 재시작·배포 중 다운타임 동안 쌓인 데이터 (일시적, 자연 해소)
- Kafka consumer group rebalancing이 잦아 처리 중단 반복
- 체크포인트 저장소(S3, HDFS) 지연으로 마이크로배치 간격 자체가 늘어짐
- 클러스터 리소스 경합: 같은 클러스터의 다른 잡이 리소스를 점유


### 진단 관점 정리

|관찰|의심 지점|
|---|---|
|전 파티션 고르게 상승|입력 증가 또는 sink 병목|
|한 파티션만 상승|스큐 또는 poison pill|
|배포 직후 상승|처리 로직 변경|
|톱니 모양(쌓였다 해소 반복)|재시작, rebalancing|
|CPU 낮은데 lag 상승|sink 병목|

엔지니어 독백:

lag 원인 찾을 때 "읽기가 느린가"부터 보는데, 경험상 범인은 쓰기 쪽인 경우가 더 많았다. 배치당 duration을 read/process/write로 쪼개서 메트릭을 남겨두면 어디서 먹는지 바로 보인다. 이거 안 해두면 매번 추측으로 executor만 늘리게 된다.



## (4)예시

엔지니어 독백:

실무에서 이 패턴은 두 가지 형태로 쓴다. 계속 도는 스트리밍 잡이면 프레임워크의 리스너에 lag 계산을 심어서 마이크로배치마다 자동으로 메트릭이 나가게 만들고, 스케줄 배치 잡이면 producer와 consumer가 각자 "내 진행 지점"을 메트릭으로 내보내고 비교는 Grafana 쿼리에서 한다. 코드에 알람 로직을 넣는 게 아니라, 코드는 숫자만 내보내고 판단은 모니터링 계층에 맡기는 구조다.

### 예시 1. Kafka + Spark Structured Streaming: 리스너로 offset lag 내보내기

구성: Kafka topic 'visits'를 읽는 Spark Structured Streaming 잡에 리스너를 달아, 마이크로배치가 끝날 때마다 lag를 Prometheus Pushgateway로 push

#### 1단계. 리스너에서 offset 두 종류 읽기
```python
class BatchCompletionSlaListener(StreamingQueryListener):
    def onQueryProgress(self, event: "QueryProgressEvent") -> None:
        latest_offsets_per_partition = self._read_last_available_offsets()
        visits_end_offsets = json.loads(event.progress.sources[0].endOffset)
        visits_offsets_per_partition: Dict[str, int] = visits_end_offsets['visits']
```

- 이 조각: 마이크로배치 완료 시점에 "Kafka의 최신 offset"과 "잡이 처리한 offset"을 파티션별로 가져옴
- 핵심 문법:
    - StreamingQueryListener:
        - Spark가 제공하는 훅 클래스
        - 상속해서 spark.streams.addListener()로 등록하면 스트리밍 잡의 생명주기 이벤트마다 Spark가 알아서 호출해줌
        - 처리 로직을 건드리지 않고 관측 코드를 붙일 수 있는 이유
    - onQueryProgress:
        - 마이크로배치 하나가 끝날 때마다 호출되는 메서드
        - event 객체에 그 배치의 진행 정보가 담겨 옴
    - event.progress.sources[0].endOffset:
        - 이 배치가 어디까지 처리했는지를 담은 JSON 문자열
        - 파싱하면 {"visits": {"0": 4200, "1": 4180, ...}} 형태로 파티션별 처리 offset이 나옴
    - _read_last_available_offsets():
        - Kafka에 직접 물어봐서 각 파티션의 최신 offset을 가져오는 헬퍼
        - 각주: Kafka AdminClient/consumer API 사용, 전체 코드는 책 GitHub repo

#### 2단계. lag 계산 후 Prometheus로 push

```python
registry = CollectorRegistry()
metrics_gauge = Gauge('visits_reader_lag', '...', registry=registry,
                      labelnames=['partition'])
for partition, value in visits_offsets_per_partition.items():
    lag = latest_offsets_per_partition[partition] - value
    metrics_gauge.labels(partition=partition).set(lag)
push_to_gateway('localhost:9091', job='...', registry=registry)
```

- 이 조각: 1단계에서 가져온 두 offset의 차이(lag)를 파티션별로 계산해 Pushgateway로 전송
- 핵심 문법:
    - CollectorRegistry:
        - prometheus_client 라이브러리에서 메트릭들을 담는 컨테이너
        - push할 메트릭 묶음의 단위
    - Gauge:
        - 오르내릴 수 있는 값 타입의 메트릭
        - lag는 늘었다 줄었다 하므로 Gauge가 맞음 (누적만 되는 값은 Counter)
    - labelnames=['partition']:
        - 같은 메트릭 이름에 파티션 번호를 라벨로 붙여 파티션별로 따로 저장
        - (3)결과에서 말한 "파티션별 분해 그래프"가 가능한 이유가 이 라벨
    - push_to_gateway('localhost:9091', ...):
        - Pushgateway 주소로 전송
        - 9091은 Pushgateway 기본 포트
- 단계 연결: 1단계가 재료(offset 두 종류)를 모으고, 2단계가 그 차이를 계산해 내보냄. 이후 알람은 Grafana에서 max(visits_reader_lag) 같은 쿼리로 처리

핵심 라인: lag = latest_offsets_per_partition[partition] - value — (2)솔루션의 비교식 "최신 - 처리 지점"이 코드로 구현된 지점



### 예시 2. Delta Lake: producer/consumer가 각자 version을 내보내기

구성: 스케줄로 도는 배치형 잡. consumer는 availableNow 트리거로 "지금 있는 것만 처리하고 종료", producer와 consumer가 각각 자기 version을 push하고 비교는 모니터링에서

#### 1단계. availableNow 트리거로 배치형 스트리밍 잡 실행
```python
visits_stream = spark_session.readStream.table('default.visits')
console_printer = (visits_stream.writeStream.trigger(availableNow=True)
    .option('checkpointLocation', checkpoint_dir)
    .option('truncate', False).format('console'))
console_printer.start().awaitTermination()
```
- 이 조각: Delta 테이블을 스트리밍으로 읽되, 현재 시점까지 쌓인 데이터만 처리하고 스스로 종료하는 잡
- 핵심 문법:
    - trigger(availableNow=True):
        - "지금 available한 데이터까지만 처리하고 종료"하는 트리거 옵션
        - 스케줄 배치처럼 돌리면서도 스트리밍 API를 쓰는 방식
    - checkpointLocation:
        - 어디까지 읽었는지(Delta version)를 Spark가 자동 기록하는 경로
        - 진행 지점을 수동 관리할 필요가 없어짐 — 이 예시에서 굳이 스트리밍 API를 쓴 이유
    - awaitTermination():
        - 잡이 끝날 때까지 대기

#### 2단계. consumer가 "마지막으로 읽은 version" push
```python
last_version = query.lastProgress["sources"][0]["endOffset"]["reservoirVersion"]
registry = CollectorRegistry()
metrics_gauge = Gauge('visits_reader_version',
                      'Last read version of the visits table', registry=registry)
metrics_gauge.set(last_version)
push_to_gateway('localhost:9091', job='visits_reader_version', registry=registry)
```

- 이 조각: 잡 종료 후 진행 정보에서 마지막으로 읽은 Delta version을 꺼내 push
- 핵심 문법:
    - query.lastProgress:
        - 마지막 마이크로배치의 진행 정보 dict
    - reservoirVersion:
        - Delta 소스의 endOffset 안에서 "읽기가 도달한 Delta 테이블 version"을 뜻하는 필드명
- 단계 연결: 1단계 잡이 끝나야 lastProgress에 최종 version이 담기고, 2단계가 그걸 내보냄


#### 3단계. producer가 "마지막으로 쓴 version" push
```python
# ... 데이터 생성 단계, 트랜잭션 commit
last_written_version = (spark_session.sql('DESCRIBE HISTORY default.visits')
    .selectExpr('MAX(version) AS last_version').collect()[0].last_version)
```

- 이 조각: producer가 쓰기 commit 직후, 테이블의 최신 version을 조회 (이후 동일하게 Gauge로 push)
- 핵심 문법:
    - DESCRIBE HISTORY:
        - Delta 테이블의 commit 이력(version, timestamp, operation)을 반환하는 Delta SQL 명령
    - MAX(version):
        - 이력 중 가장 최신 commit 번호 = 방금 쓴 version
- 단계 연결: 2단계(consumer)와 3단계(producer)가 각자 자기 지점을 push → 모니터링에서 visits_writer_version - visits_reader_version 차이가 임계값을 넘으면 lag 알람

핵심 라인: 두 Gauge의 차이 비교식 자체. 실제 값 대입: writer version 152, reader version 149 → lag 3 commit

### 실제 사용 시나리오
```promql
# Grafana 알람 조건 (Prometheus 쿼리)
visits_writer_version - visits_reader_version > 5

# Kafka 쪽 알람: 최악 파티션 기준
max(visits_reader_lag) > 1000
```
- 잡 코드는 숫자만 내보내고, "5 commit 이상 벌어지면 알람" 같은 판단은 전부 이 쿼리 계층에서 함

정리:
- 한 줄 요약: consumer(그리고 필요 시 producer)가 진행 지점을 메트릭으로 push하고, 차이 계산과 알람은 Prometheus/Grafana가 담당
- 실행 순서: 리스너/잡 종료 시점에 지점 수집 → Gauge로 push → 모니터링 쿼리로 차이 감시
- 핵심: Kafka는 offset 차이를 잡 안에서 계산, Delta는 version을 양쪽이 따로 내보내 밖에서 비교

엔지니어 독백:
Pushgateway 쓸 때 하나 조심할 게, 잡이 죽어도 마지막 push된 값이 계속 남아 있다는 거다. lag 800 찍고 잡이 죽으면 대시보드엔 영원히 800이 떠 있어서 "lag가 안정적이네"로 착각하기 딱 좋다. 그래서 lag 메트릭만 보지 말고 흐름 중단 탐지기에서 배운 last update time 메트릭을 같이 띄워놔야 한다. "값이 얼마냐"와 "그 값이 언제 적 거냐"는 항상 세트다.



## (5)최신트렌드

결론: 요즘 Kafka lag는 잡 안에 리스너 코드를 짜지 않고, 외부 exporter가 Kafka 메타데이터를 읽어 자동 수집하는 방식이 표준이다. 책 예시처럼 잡마다 코드를 심는 방식은 잡 수가 늘면 유지보수가 안 되기 때문.

### 1. kafka-exporter / Kafka Lag Exporter (외부 exporter)

개요

- Kafka 브로커에 붙어 consumer group의 committed offset과 파티션 최신 offset을 주기적으로 읽어 Prometheus 메트릭으로 노출하는 독립 프로세스
- 잡 코드와 완전히 분리되어 별도 컨테이너 하나로 동작

기존 방식의 한계

- 책 예시의 리스너 방식은 잡마다 계측 코드를 복붙해야 함
- 잡이 죽으면 메트릭도 같이 끊김 → 정작 제일 알고 싶은 "잡이 죽어서 lag가 쌓이는 상황"을 잡 스스로는 보고할 수 없음

도입 이유

- consumer group만 있으면 코드 0줄로 전 잡의 lag 수집
- 잡 생사와 무관하게 exporter가 살아 있는 한 lag는 계속 관측됨
- kafka_consumergroup_lag{group="visits-processor"} 형태로 바로 Grafana에 연결

### 2. Burrow (LinkedIn 오픈소스)

개요

- LinkedIn이 만든 Kafka consumer 모니터링 서비스. lag 숫자가 아니라 "이 consumer가 건강한가"를 상태(OK/WARN/ERR)로 평가

기존 방식의 한계

- lag 숫자에 임계값을 거는 방식은 topic마다 적정 임계값이 달라 관리 부담 (트래픽 큰 topic은 lag 1만도 정상, 작은 topic은 100도 비정상)

도입 이유

- offset이 전진하는 추세를 보고 판단 → "lag가 크지만 줄어드는 중"은 OK, "작지만 멈춰 있음"은 ERR
- 임계값 튜닝 없이 consumer 상태 평가 자동화

### 3. Datadog, CloudWatch(MSK) 등 관리형 스택

개요

- 상용 APM/클라우드 모니터링. Kafka 연동 시 consumer lag 대시보드와 알람이 기본 제공

기존 방식의 한계

- Prometheus + Pushgateway + Grafana 스택은 직접 설치·운영해야 하고, Pushgateway의 잔존값 문제 같은 함정도 스스로 관리

도입 이유

- AWS MSK 쓰면 CloudWatch에 MaxOffsetLag, EstimatedTimeLag 메트릭이 자동으로 옴
- EstimatedTimeLag처럼 "몇 건 밀렸나"를 "몇 초 밀렸나"로 환산해줘서 비즈니스 대화에 바로 쓰기 좋음

### 4. Spark/Flink 자체 메트릭 활용

개요

- 프레임워크가 이미 내보내는 내장 메트릭을 그대로 긁는 방식
- Flink: records-lag-max 등 Kafka connector 메트릭 기본 제공
- Spark: metrics.properties 설정으로 PrometheusServlet 노출 가능

기존 방식의 한계

- 리스너 직접 구현은 프레임워크 버전 업그레이드 때마다 깨질 위험

도입 이유

- 커스텀 코드 최소화, 프레임워크가 유지보수해주는 메트릭 사용

### 실무 선호 정리

- Kafka lag: kafka-exporter(또는 Lag Exporter)가 사실상 기본값. 코드 없이 전 consumer group 커버
- consumer 상태 평가까지 원하면: Burrow 추가
- MSK/Confluent Cloud 등 관리형 Kafka: 제공되는 기본 메트릭부터 사용
- 책 예시 같은 잡 내부 계측이 여전히 필요한 경우: Delta version처럼 외부 exporter가 없는 스토어, 또는 event time 기준 지연 같은 비즈니스 정의 lag를 잴 때
- 방향: lag 수집은 인프라 계층으로 내리고, 잡 코드에는 비즈니스 고유 지표만 남김

엔지니어 독백:

리스너로 lag 계측 직접 짜본 사람으로서 말하면, 처음엔 뿌듯한데 잡이 10개 넘어가는 순간 후회한다. exporter 하나 띄우면 끝날 일이었다. 다만 exporter가 주는 offset lag는 "몇 건"이지 "몇 분"이 아니라서, 비즈니스랑 얘기할 땐 환산이 필요하다. 그래서 나는 offset lag는 exporter에 맡기고, 잡 안에서는 "지금 처리 중인 레코드의 event time이 현재 시각보다 얼마나 과거인가" 하나만 직접 내보낸다. 이 둘이면 기술 대화와 비즈니스 대화가 다 커버된다.

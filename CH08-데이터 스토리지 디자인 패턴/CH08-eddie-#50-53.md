**Chapter 8: Data Storage Design Patterns — 전체 흐름**

**왜 이 챕터가 필요한가 (책의 문제의식)**

- 빅데이터 환경에서 쿼리/잡 결과를 2분, 심지어 10분 넘게 기다려본 경험은 다들 있음
- 이 지연을 줄이는 방법은 두 가지:
    1. **컴퓨팅 자원을 더 붙인다**: 빠르고 쉽지만, 보통 유저 컴플레인 터진 뒤에야 급하게 하는 사후 대응(retroactive) 조치. 비용도 계속 늘어남
    2. **데이터를 미리 잘 정리해둔다**: 이 챕터가 다루는 사전 대응(preemptive) 조치. 데이터 저장 구조를 잘 짜두면 실행 시간이 줄고, 문제 파악도 더 빨리 됨
- Chapter 8은 이 두 번째 접근, 즉 "데이터를 어떻게 물리적으로 배치해야 빠르고 싸게 접근할 수 있는가"를 다룸

**4개 섹션 흐름 (이렇게 진행되는 이유)**

1. **Partitioning (파티셔닝)** — 가장 기초적인 정리 방법. 데이터를 통째로(수평) 또는 쪼개서(수직) 다른 위치로 나눠 저장. 처리할 데이터 volume 자체를 줄이는 게 목적
2. **Records Organization (레코드 정리)** — 파티셔닝의 한계(고카디널리티 컬럼엔 안 먹힘)를 보완. 개별 row가 아니라 row _그룹_을 어떻게 배치/정렬할지 다룸
3. **Read Performance Optimization (읽기 성능 최적화)** — 파티셔닝/정리를 넘어서, 메타데이터 계층 활용, 비싼 연산 미리 실행해두기(materialize), listing 연산 회피 등으로 읽기 자체를 더 빠르게
4. **Data Representation (데이터 표현)** — 테이블을 정규화(일관성 우선)할지 비정규화(속도 우선)할지의 트레이드오프

**패턴 정리표**

| 섹션                   | 패턴                     | 핵심 목적                                 | 대표 트레이드오프(Gotcha)                          |
| -------------------- | ---------------------- | ------------------------------------- | ------------------------------------------ |
| Partitioning         | Horizontal Partitioner | 전체 row를 속성값 기준으로 분리 저장 → 처리량 감소       | 저카디널리티에만 적합, 파티션 키 변경 어려움, skew 발생 가능      |
| Partitioning         | Vertical Partitioner   | row를 컬럼 단위로 쪼개서 분리 저장 (가변/불변 속성 분리)   | 읽을 때 join 필요, 쿼리 복잡도 증가                    |
| Records Organization | Bucket                 | 고카디널리티 컬럼을 그룹 단위로 같은 저장 공간에 colocate  | bucket 개수 잘못 잡으면 효과 감소                     |
| Records Organization | Sorter                 | 저장 시 정렬 순서를 강제해서 조회 시 스캔 범위 축소        | 정렬 유지 비용, 업데이트 시 재정렬 필요                    |
| Read Performance     | Metadata Enhancer      | 메타데이터 계층에 통계 추가해서 불필요한 데이터 접근 회피      | 메타데이터 생성 자체가 쓰기 단계에 오버헤드 추가                |
| Read Performance     | Dataset Materializer   | 복잡한 쿼리 결과를 테이블/뷰로 미리 구체화(materialize) | refresh 비용, 데이터 관리(retention/access 정책) 부담 |
| Read Performance     | Manifest               | listing 연산 자체를 회피                     | manifest 파일이 너무 커질 수 있음                    |
| Data Representation  | Normalizer             | 데이터를 분리 저장해서 일관성 확보                   | 여러 테이블 join 필요 시 쿼리 비용 증가                  |
| Data Representation  | Denormalizer           | 테이블 간 join 횟수를 줄임 (One Big Table 등)   | 업데이트 시 row 일관성 깨질 위험, 저장 용량 증가             |

**챕터 앞부분과의 연결 (참고)**

- Partitioning은 Chapter 4에서 본 idempotency 패턴(예: Fast Metadata Cleaner)을 구현하는 데도 활용됨 — 파티션 단위로 삭제/덮어쓰기 하면 idempotent 연산이 쉬워짐
- Vertical Partitioner는 Chapter 7(보안)에도 동명의 패턴이 있었는데, 그건 "PII 삭제 최적화"가 목적이고, Chapter 8 버전은 "저장소 최적화"가 목적. 이름은 같지만 적용 맥락이 다름

> (엔지니어 독백)
> 
> 이 챕터 순서가 논리적으로 잘 짜여있다고 느낀 이유가, Partitioning → Records Organization → Read Optimization → Representation 순으로 갈수록 "더 세밀한 최적화"로 내려간다. (1)Partitioning은 큰 덩어리를 나누는 거고, 
> (2)Bucket/Sorter는 그 안에서 row들을 더 똑똑하게 배치하는 거고, 
> (3)Materializer/Manifest는 아예 읽기 자체를 캐싱하거나 스캔을 생략하는 거고, 
> 마지막 (4)Normalizer/Denormalizer는 테이블 구조 자체의 트레이드오프다. 실무에서 성능 튜닝할 때도 이 순서대로 검토하면 놓치는 게 적다 — 일단 파티셔닝이 제대로 됐는지부터 보고, 그다음 세부 최적화로 내려가는 식이다.



# **패턴 #50: Horizontal Partitioner (수평 파티셔너)**

한 줄 정의: 데이터셋을 특정 속성(partitioning attribute/distribution key) 값 기준으로 물리적으로 분리된 저장 공간에 나눠 저장하는 패턴.

## (1) 문제상황

핵심 고통: 데이터가 계속 쌓이는데, 매번 전체를 훑어야 해서 배치 잡이 느려지고 비싸진다.

> (엔지니어 독백)
> 
> 최근 4일치 데이터로 rolling aggregate를 계산하는 배치 잡을 만들었다. 처음 몇 달은 잘 돌았는데, 저장 계층에 데이터가 계속 쌓이면서 잡 성능이 눈에 띄게 떨어졌다. 
> 원인을 보니, "4일보다 오래된 레코드는 무시한다"는 
> 필터링 연산의 실행 시간이 계속 늘어나고 있었다 
> — 4일치만 쓰는데도 전체 데이터를 다 스캔한 뒤에 걸러내고 있었기 때문이다.
> 
> 급한 대로 클러스터에 컴퓨팅 파워를 더 붙여서 버텼는데, 그만큼 비용이 올라갔다. 
> 결국 "필요한 4일치만 애초에 물리적으로 분리해서, 
> 그 부분만 딱 골라 읽을 수 있는" 구조가 필요했다. 
> 이게 Horizontal Partitioner가 푸는 지점이다.





## (2)솔루션

주요컨셉: partitioning attribute(distribution key)를 정해서, 데이터 ingestion 과정 또는 데이터 저장소가 이 속성값별로 물리적으로 분리된 저장 공간에 데이터를 나눠 쓴다.

- **시간 기반 파티션 (가장 흔한 형태)**
    - 시간 경계로 데이터를 나눠서, 필요한 시간대만 빠르고 싸게 조회 가능하게 함
    - 시간 속성의 출처는 두 가지로 나뉨:

1. **잡 실행 시점(job execution context) 기준**
    - 잡이 실행된 시각을 기준으로 파티셔닝. 같은 실행 회차에 처리된 레코드는 전부 같은 파티션 값을 가짐
    - 예: 2024-12-31에 실행된 잡이면, 그 실행에서 처리한 모든 레코드가 `2024-12-31` 파티션에 들어감
    - 적합한 상황: 레코드 자체의 발생 시각보다 "언제 처리됐는지"가 중요한 배치 잡
    - **실무 예시**: 
	    - 매일 새벽 1시에 도는 배치 잡이 있고, 전날 발생한 서버 로그 파일을 하루 한 번 통째로 읽어서 처리한다고 하자. 이 로그 파일 안 레코드들의 실제 발생 시각이 23:58, 23:59, 00:01 등으로 자정 앞뒤에 걸쳐 있어도, 어차피 이 잡은 "어제자 로그를 오늘 처리한다"는 개념으로 동작하니까 굳이 레코드 하나하나의 정확한 발생 시각을 볼 필요가 없다. 그냥 "이 잡의 실행 날짜 = 2024-12-31"로 전체를 한 파티션에 몰아넣으면 충분하고, 파티셔닝 로직도 단순해진다.
	    >  (엔지니어 독백) 이 방식이 맞는지는 "레코드 하나하나의 정확한 발생 시각이 다운스트림 분석에서 중요한가"로 판단한다. 하루 단위 배치 요약(일간 리포트, 일별 매출 집계 등)처럼 애초에 "그날 통째로"가 분석 단위인 경우엔, 굳이 이벤트 시간을 따로 볼 필요 없이 실행 시점 기준이 구현도 간단하고 충분하다.



2. **레코드의 이벤트 시간(event time) 기준**
    - 레코드가 실제로 발생한 시각을 기준으로 파티셔닝. 잡이 언제 돌았는지와 무관하게, 레코드 안의 timestamp 값으로 분류
    - 적합한 상황: 지연 도착(late arrival) 데이터가 있어서 "언제 처리됐는지"보다 "언제 실제 발생했는지"가 중요한 경우
    - rolling aggregate, 시계열 분석 같은 케이스
    - 실무 예시
	    - 모바일 앱에서 발생하는 클릭 이벤트를 Kafka로 수집해서 Spark Structured Streaming으로 처리하는 상황
	    - 유저가 지하철처럼 네트워크가 불안정한 곳에서 앱을 쓰면, 실제로는 09:00에 발생한 클릭 이벤트가 네트워크 재연결 후 09:40에야 서버에 도착하는 경우가 흔하다. 이때 "잡이 처리한 시각(09:40)" 기준으로 파티셔닝하면, 이 이벤트가 실제로는 09:00대 트래픽 분석에 포함돼야 하는데 09:40 파티션에 섞여버려서 "09:00~09:10 사이 클릭 수" 같은 시계열 집계가 부정확해진다. 레코드 안에 있는 실제 발생 timestamp를 기준으로 파티셔닝해야, 늦게 도착해도 올바른 시간대 파티션에 들어감
	    > (엔지니어 독백) late arrival이 실제로 자주 발생하는 스트리밍 파이프라인에서는 이벤트 시간 기준이 사실상 필수다. 다만 이 경우 이미 write 완료된 과거 파티션에 뒤늦게 데이터를 추가로 써야 하는 상황이 생기므로, watermark 같은 late data 처리 메커니즘과 같이 설계해야 한다. "언제까지 늦게 온 데이터를 받아줄지" 기준을 안 정해두면, 파티션이 계속 열려있는 상태로 관리 부담이 커진다.

|기준|적합한 상황|실무 예시|
|---|---|---|
|잡 실행 시점|처리 단위 자체가 "그날 통째로"인 배치 잡|매일 새벽 도는 일간 로그 배치 처리|
|이벤트 시간|late arrival이 있고, 실제 발생 시각이 분석에 중요|모바일 클릭 스트림, 네트워크 지연으로 늦게 도착하는 이벤트|


실무 선호: 대부분의 rolling aggregate, 시계열 분석 같은 케이스는 **이벤트 시간 기준**이 정확도 면에서 낫다. 다만 이벤트 시간 기준은 late data가 오면 이미 write된 과거 파티션에 다시 데이터를 추가해야 하는 부담이 있어서, late data가 거의 없는 파이프라인이면 잡 실행 시점 기준이 구현이 더 단순하고 충분함.


## (3)결과

결론: 역설적으로 이 패턴의 가장 큰 단점은 파티션 그 자체, 특히 정적(static)이라는 특성에서 나온다.

1. **Granularity*와 메타데이터 오버헤드**
	- *Granularity = 데이터를 얼마나 잘게 나누는지의 단위
		- coarse-grained (거친/큰 단위)**: 하나의 파티션에 데이터가 많이 몰림
		    - 예: 이벤트 시간을 "일(day)" 단위로 파티셔닝 → 하루치 수백만 건이 파티션 하나에
		- fine-grained (세밀한/작은 단위): 하나의 파티션에 데이터가 적게 들어감, 파티션 개수는 많아짐
		    - 예: username처럼 사람마다 다른 값으로 파티셔닝 → 유저 한 명당 파티션 하나씩, 파티션이 수백만 개
    - 배경: 
	    - 파티션은 같은 속성값을 가진 레코드들을 모아두는 물리적 위치. 파티션이 너무 많아지면 데이터 저장소에 부담을 줌(I/O부하)
    - 문제: 
	    - 예를 들어 하루 100만 명의 고유 유저가 방문하는 사이트에서 username으로 파티셔닝하면, 100만 개 파티션이 생김. 파티션 목록 조회(listing)가 느려지고, 작은 파일이 잔뜩 생기는 small files 문제까지 겹침
    - 대응: 
	    - 저카디널리티(distinct 값이 적은) 속성을 파티션 키로 쓰는 게 원칙. 
	    - 이벤트 시간을 시간/일 단위로 반올림하는 게 대표적인 좋은 예. 
	    - IoT 디바이스 ID처럼 고카디널리티 속성은 파티셔닝 대신 뒤에 나올, 해시로 몇십~몇백 개 그룹으로 묶어서 저장하는 Bucket 기법을 씀 — `hash(device_id) % 100`처럼 계산해서 같은 그룹끼리 모아 저장
2. **Skew (데이터 쏠림)**
    - 배경: 
	    - 수평 파티셔닝이 균등한 데이터 분산을 보장할 거라 생각하기 쉽지만 실제론 아님
    - 문제: 
	    - 특히 micro-batch 스트림 처리 모델에서 치명적. micro-batch는 이전 배치가 끝나야 다음 배치가 도는 blocking 구조라서, 한 파티션에 데이터가 쏠리면 그 파티션이 전체 micro-batch의 완료 시간을 결정해버림 — 짧은 파티션들도 쏠린 파티션 끝날 때까지 대기
    - 대응: 
	    - backpressure 메커니즘 적용. 
	    - 쏠린 파티션의 초과분을 별도 버퍼에 저장해두고 다음 micro-batch에서 처리. 대신 그 쏠린 파티션의 데이터 지연(latency)은 늘어나는 대신, 나머지 파티션은 거의 실시간으로 계속 처리 가능
	    - *backpressure: 처리 속도가 유입 속도를 못 따라갈 때, 넘치는 만큼을 뒤로 미뤄서(버퍼에 쌓아두고) 시스템이 감당 가능한 속도로만 처리하게 만드는 흐름 제어(flow control) 메커니즘


		**"넘친다"의 기준: 이번 micro-batch에서 "처리하기로 정해둔 최대량"**
		
		- micro-batch 시스템은 보통 "이번 배치에서 파티션당 최대 몇 개의 레코드를 처리할지" 상한선을 설정해둠 (예: Spark Structured Streaming의 `maxOffsetsPerTrigger`, `maxBytesPerTrigger` 같은 설정)
		- 예: "한 파티션당 이번 batch에서 최대 1,000개 레코드까지만 처리한다"고 설정했다고 하자
		
		**구체적 숫자로**
		
		- 평소: 한 파티션에 5초마다 500개 레코드 쌓임 → 상한선(1,000개) 안 넘음 → 다 처리
		- 쏠린 시점: 한 파티션에 5초 사이 5,000개 레코드 쌓임 → 상한선(1,000개) 넘음
		- 이때 "초과분"이란: 5,000개 중 처리 상한선을 넘는 **4,000개** — 이 4,000개는 이번 batch에서 처리 안 하고 버퍼(Kafka라면 그냥 아직 안 읽은 offset으로) 남겨둠
		- 이번 batch는 1,000개만 처리하고 끝 → 짧게 끝남 → 다음 batch 바로 시작 가능
		- 남은 4,000개는 다음 batch, 그다음 batch에서 다시 1,000개씩 나눠서 처리
		
		**"데이터 양을 줄여서 처리한다"는 게 아니라, "처리를 여러 batch에 나눠서 분할한다"는 뜻**
		
		- 데이터를 버리는 게 아님. 5,000개 전부 언젠가는 다 처리됨
		- 다만 "한 번에 다 처리" 대신 "여러 번에 걸쳐서 나눠 처리"로 바뀌는 것
		- 그래서 이 파티션의 데이터는 원래보다 늦게(여러 batch에 걸쳐서) 처리 완료됨 → latency 증가


3. **Mutability (파티션 키 변경의 어려움)**
    - 배경: 
	    - 파티션 키를 바꾸려면 이미 써놓은 모든 데이터를 새 위치로 옮겨야 함
    - 문제: 
	    - 비용도 크고 시간도 오래 걸림
    - 대응: 
	    - Apache Iceberg 같은 일부 데이터 저장소는 
	    - 메타데이터 레이어에서만 파티션 스키마를 즉시 바꿀 수 있음(파일 이동 없이). 
	    - 다만 이건 새로 들어오는 레코드부터만 새 구조 적용, 기존 레코드는 옛날 파티션 구조 그대로 남음
	    > Q.이거 잡코드 바꿔야하나?
		    - "어? 그럼 파티션 구조가 day였다가 hour로 바뀌었는데, 이 두 구조가 섞여있는 테이블을 읽는 잡 코드는 예전 구조 읽는 부분이랑 새 구조 읽는 부분을 각각 다르게 처리해야 하는 거 아닌가?"라는 걱정이 자연스럽게 생김
		    -**잡 코드는 안 바꿔도 된다.** Iceberg의 메타데이터 계층이 이 차이를 알아서 흡수해준다.
		    - 파티션 스키마가 바뀌면(예: `day` 단위 → `hour` 단위), Iceberg는 "이 시점 이후 새 파일들은 hour 파티션 구조다"라는 정보만 메타데이터에 추가로 기록함
			-잡 코드에서 `spark.read.table('visits').filter(...)`처럼 테이블을 조회하면, Spark는 Iceberg 메타데이터를 보고 **"이 쿼리 범위에서 옛날 파일은 day 구조로, 새 파일은 hour 구조로 읽어야 한다"는 걸 자동으로 판단**해서 각각 맞는 방식으로 읽어옴


> (엔지니어 독백)
> 
> Skew 문제는 실제로 스트리밍 파이프라인 운영하면서 제일 골치 아팠던 부분이다. 처음엔 "파티션 몇 개로 나눴으니 균등하게 처리되겠지"라고 안일하게 생각했는데, 특정 시간대(예: 이벤트 발생 직후)에 트래픽이 몰리면 그 파티션 하나 때문에 전체 micro-batch가 다 느려지는 걸 직접 봤다. backpressure 버퍼를 붙이고 나서야 "쏠린 파티션은 좀 늦게 처리되더라도 나머지는 제때 처리된다"는 구조로 안정화됐다.


##  (4)예시

### **기술1: Apache Spark — partitionBy**

- 이벤트 시간을 year/month/day/hour로 쪼개서 컬럼으로 만든 뒤, 이 컬럼들 기준으로 물리적 파티셔닝


```python
partitioned_users = (input_users
    .withColumn('year', functions.year('change_date'))
    .withColumn('month', functions.month('change_date'))
    .withColumn('day', functions.day('change_date'))
    .withColumn('hour', functions.hour('change_date')))

(partitioned_users.write.mode('overwrite').format('delta')
    .partitionBy('year', 'month', 'day', 'hour').save(output_dir))
```

- 한 줄 요약: `change_date` 컬럼에서 연/월/일/시를 각각 별도 컬럼으로 뽑아내고, 그 네 컬럼 조합으로 물리적 디렉토리를 나눠서 저장
- 실행 순서:
    1. `withColumn`으로 year/month/day/hour 4개 컬럼을 순서대로 추가 (원본 `change_date`는 그대로 유지)
    2. `.write.mode('overwrite').format('delta')`로 출력 형식을 Delta Lake로 지정
    3. `.partitionBy('year', 'month', 'day', 'hour')` 호출 — 이 네 컬럼 조합값별로 물리적 디렉토리 생성
    4. `.save(output_dir)` 호출 시점에 실제 쓰기 실행 (write는 action이므로 여기서 비로소 계산 트리거)
- ★ 핵심: 
	- `.partitionBy('year', 'month', 'day', 'hour')` 적용→  `output_dir/year=2024/month=12/day=31/hour=09/`
> (엔지니어 독백)
>  year/month/day/hour를 따로 쪼갠 이유는 조회 유연성 때문이다. 
>  day까지만 파티셔닝했으면 "이번 달 전체" 같은 넓은 범위 조회는 되지만 
>  "이 시간대만" 같은 좁은 조회는 못 한다. 
>  반대로 hour까지 쪼개놓으면 넓은 범위든 좁은 범위든 다 필요한 디렉토리만 골라 읽을 수 있다. 
>  다만 너무 잘게 쪼개면 (3)에서 본 granularity 문제가 생기니까, 데이터 volume 대비 hour 단위가 여전히 파티션 하나에 유의미한 양(수백~수천 row 이상)이 들어갈 정도인지 먼저 확인해야 한다.



### 기술2 **Apache Kafka Custom Partitioner**

- 배경: 
	- producer가 메시지 보낼 때마다 "몇 번 파티션에 넣을지" 결정 필요. 
	- 기본은 key 해시값으로 자동 결정.
	- 특정 key를 반드시 특정 파티션에 몰아야 하는 경우, 기본 로직 대신 직접 규칙을 짜는 게 custom partitioner

**1단계: 규칙표 정의**
```java
private final static Map<String, Integer> RANGES_PER_PARTITIONS = new HashMap<>();
static {
    RANGES_PER_PARTITIONS.put("A", 0);
    RANGES_PER_PARTITIONS.put("B", 0);
}
```

- key "A", "B" → 파티션 0으로 보낸다는 규칙표 생성
- `static { }` 블록이라 클래스 로드 시 딱 한 번만 실행, 이후 계속 재사용


**2단계: 규칙표 조회 함수**
```java
@Override
public int partition(String topic, Object key, byte[] keyBytes,
                      Object value, byte[] valueBytes, Cluster cluster) {
    String keyAsString = key.toString();
    return RANGES_PER_PARTITIONS.getOrDefault(keyAsString, DEFAULT_PARTITION);
}
```

- Kafka가 메시지 보낼 때마다 이 `partition()` 함수를 자동 호출
- key를 문자열로 변환 → 규칙표에서 조회 → 있으면 그 값, 없으면 `DEFAULT_PARTITION`(1) 리턴
- 이 리턴값이 곧 메시지가 갈 파티션 번호


**3단계: producer에 등록**
```java
Properties props = new Properties();
props.put("partitioner.class", "com.waitingforcode.RangePartitioner");
```

- 1~2단계로 클래스만 만들어놓으면 아직 아무 효과 없음
- `partitioner.class` 설정으로 producer한테 "이 클래스 써라"고 명시해야 실제 동작

**실행 흐름 (key="A" 예시)**

1. producer가 key="A"인 메시지 전송 시도
2. Kafka가 `partition()` 자동 호출
3. key "A"를 규칙표에서 조회 → 0 발견
4. 0 리턴 → 이 메시지는 파티션 0으로 전송
5. (key="C"처럼 규칙표에 없는 값이면 → `DEFAULT_PARTITION`(1) 리턴 → 파티션 1로 전송)

- ★ 핵심: `RANGES_PER_PARTITIONS.getOrDefault(keyAsString, DEFAULT_PARTITION)` — 규칙표 정의(1단계) → 조회(2단계) → producer 등록(3단계), 이 세 단계가 다 이어져야 커스텀 라우팅이 실제로 동작함

> (엔지니어 독백)
> 
> 커스텀 파티셔너는 코드 복잡도를 늘리는 선택이다. 대부분은 Kafka 기본 파티셔너(key 해시 기반 자동 분배)로 충분하고, 커스텀은 "특정 key를 반드시 특정 파티션에 몰아야 하는" 명확한 이유(예: 특정 고객사 데이터를 전용 파티션/consumer로 격리해야 하는 계약 조건)가 있을 때만 쓴다. 이유 없이 짜면 오버엔지니어링이다.



### **기술3: PostgreSQL — RANGE 파티셔닝**

먼저 왜 이게 필요한지: 방문 로그처럼 계속 쌓이는 테이블을 하나로 두면, 날짜 하나 조회할 때도 테이블 전체를 다 훑어야 한다. 날짜별로 물리적으로 나눠서 저장해두면, 필요한 날짜의 조각만 읽으면 된다. PostgreSQL은 이걸 `PARTITION BY`라는 문법으로 지원한다.

**1단계: 부모 테이블 생성**
```sql
CREATE TABLE visits_all (
    visit_id CHAR(36) NOT NULL,
    event_time TIMESTAMP NOT NULL,
    user_id TEXT NOT NULL,
    page VARCHAR(20) NULL,
    PRIMARY KEY(visit_id, event_time)
) PARTITION BY RANGE(event_time);
```

- 이 코드가 뭘 하는 코드인지: `visits_all`이라는 테이블을 만드는데, 맨 끝에 `PARTITION BY RANGE(event_time)`가 붙어있어서 일반 테이블이 아니라 "파티션을 가진 테이블"로 선언됨
- `PARTITION BY RANGE(event_time)`의 의미: "이 테이블은 `event_time` 컬럼 값의 범위(RANGE)에 따라 여러 조각으로 나눠서 저장하겠다"는 선언. 근데 이 CREATE TABLE 문 자체는 "나눌 거다"라는 규칙만 정의한 것이고, 실제로 나뉘어질 조각(자식 테이블)은 아직 하나도 안 만들어짐
- 이 상태에서 바로 데이터를 넣으려고 하면 에러가 남 — 규칙만 있고 실제로 데이터를 담을 조각이 없기 때문

**2단계: 자식 테이블(실제 데이터가 들어갈 조각) 생성**
```sql
CREATE TABLE visits_all_20231124 PARTITION OF visits_all
FOR VALUES FROM('2023-11-24 00:00:00') TO ('2023-11-24 23:59:59');
```

- 이 코드가 뭘 하는 코드인지: `visits_all_20231124`라는 새 테이블을 만드는데, `PARTITION OF visits_all`이라고 써서 "이건 독립된 테이블이 아니라 `visits_all`의 조각 중 하나다"라고 선언
- `FOR VALUES FROM('2023-11-24 00:00:00') TO ('2023-11-24 23:59:59')`의 의미: "이 조각은 `event_time`이 2023-11-24 00시부터 23시59분 사이인 row만 담당한다"는 범위 지정
- 즉 이 한 줄로 "11월 24일치 데이터를 담을 물리적 공간"이 하나 만들어짐

```sql
CREATE TABLE visits_all_20231125 PARTITION OF visits_all
FOR VALUES FROM('2023-11-25 00:00:00') TO ('2023-11-25 23:59:59');
```

- 같은 방식으로 11월 25일치를 담당하는 조각을 하나 더 생성. 필요한 날짜 수만큼 이 문장을 반복해서 조각을 계속 추가하는 구조

**3단계: 실제 사용 — INSERT할 때 무슨 일이 일어나는지**
```sql
INSERT INTO visits_all (visit_id, event_time, user_id, page)
VALUES ('abc-123', '2023-11-24 09:30:00', 'user1', 'home.html');
```

- 애플리케이션(또는 Spark 잡)은 그냥 `visits_all`이라는 부모 테이블에 INSERT하면 됨. `visits_all_20231124`를 직접 지정할 필요 없음
- PostgreSQL이 알아서 이 row의 `event_time`(2023-11-24 09:30:00)을 보고, 이 값이 어느 자식 테이블의 범위에 속하는지 확인한 뒤 `visits_all_20231124`에 실제로 저장

**4단계: 조회할 때 무슨 일이 일어나는지**

```sql
SELECT * FROM visits_all WHERE event_time >= '2023-11-24 00:00:00' AND event_time < '2023-11-25 00:00:00';
```

- 이 쿼리도 부모 테이블 `visits_all`에 그대로 날림
- PostgreSQL이 WHERE 조건의 날짜 범위를 보고, 이 조건에 해당하는 자식 테이블이 `visits_all_20231124` 하나뿐이라는 걸 판단해서 그 테이블만 스캔. 나머지 자식 테이블(`visits_all_20231125` 등)은 아예 안 건드림 — 이걸 partition pruning이라고 부름

**정리**

- 부모 테이블(`visits_all`)은 실제 데이터를 안 담고, "어떤 값이 어느 자식 테이블로 가야 하는지"에 대한 규칙만 가짐
- 자식 테이블(`visits_all_20231124` 등)이 실제 물리적으로 데이터를 저장하는 공간
- 애플리케이션은 항상 부모 테이블만 보고 INSERT/SELECT하면 되고, 라우팅과 pruning은 PostgreSQL이 알아서 처리

> (엔지니어 독백)
> 
> 매일 새 파티션 테이블을 수동으로 CREATE 하는 건 실무에서 안 한다. 보통 `pg_partman` 같은 확장이나 스케줄된 잡으로 다음 날짜 파티션을 미리 자동 생성해둔다. 파티션이 없는 날짜의 데이터가 들어오면 INSERT 자체가 실패하니까, 자동 생성 장치 없이 운영하면 언젠가 자정 넘어 장애가 난다.

**분류 요약**

|기술|파티셔닝 기준|구현 방식|
|---|---|---|
|Apache Spark|컬럼 값 (year/month/day/hour)|`partitionBy` + 물리적 디렉토리|
|Apache Kafka|메시지 key|커스텀 `Partitioner` 클래스|
|PostgreSQL|RANGE(컬럼 범위)|`PARTITION BY RANGE` + 자식 테이블|


## (5)최신트렌드


### **1. Apache Iceberg — Hidden Partitioning + Partition Evolution**

- 정체: 파티션 값을 사람이 직접 컬럼으로 안 만들어도, 테이블 정의 시점에 "이 컬럼을 이런 함수로 변환해서 파티셔닝하겠다"고 선언하면 Iceberg가 알아서 처리해주는 기능
- 이전 한계: Spark `partitionBy` 예시에서 봤듯, year/month/day/hour 컬럼을 사람이 직접 `withColumn`으로 만들어야 했음. 쿼리할 때도 이 파생 컬럼들을 정확히 알고 필터링해야 partition pruning이 제대로 작동함
- - 전: `event_time`에서 year/month/day/hour 컬럼을 사람이 직접 만들고, 쿼리할 때도 그 컬럼들을 알고 필터링해야 함
- 후: 원본 `event_time`만 필터링해도 Iceberg가 알아서 파티션 매핑. 파생 컬럼 관리 자체가 사라짐
- 체감: 파티션 구조를 나중에 바꿔도 코드 수정 없음 (앞서 본 partition evolution)

	**Iceberg Hidden Partitioning은 이 파생 컬럼 자체를 사람이 안 만든다**
	테이블 만들 때 이렇게 선언:
	```sql
	CREATE TABLE visits (
	    visit_id STRING,
	    change_date TIMESTAMP,
	    page STRING
	)
	PARTITIONED BY (day(change_date))
	```
	- (기존) `change_date`라는 원본 컬럼에서 사람이 직접 year/month/day/hour 파생 컬럼을 4개 만들어야 함
	- `PARTITIONED BY (day(change_date))` — "원본 컬럼 `change_date`를 `day()`라는 함수로 변환한 값 기준으로 파티셔닝하겠다"는 선언
	- 여기서 중요한 점: **테이블 스키마에 `year`, `month`, `day` 같은 컬럼이 새로 생기지 않음.** 테이블은 여전히 `visit_id`, `change_date`, `page` 세 컬럼만 갖고 있음. 파티션 값은 Iceberg 내부 메타데이터에만 존재하고 사용자 눈에는 "숨겨져(hidden)" 있음 — 그래서 이름이 Hidden Partitioning
### **2. Databricks Delta Lake — Liquid Clustering**

- 정체: 전통적인 파티션 컬럼 지정 없이, 자주 조회되는 컬럼들을 클러스터링 키로만 지정하면 Delta Lake가 데이터 배치를 자동으로 최적화해주는 기능
- - 전: 
	- 파티션 키 한번 정하면 바꾸기 어렵고, 고카디널리티 컬럼은 못 씀
- 후: 
	- 클러스터링 키만 지정하면 데이터 배치를 자동 최적화. 파티션 개수를 미리 안 정해도 됨
- 체감: 
	- 쿼리 패턴 바뀌어도 재설계 부담 적음

**Liquid Clustering — 예시로**
```sql
CREATE TABLE visits (
    visit_id STRING,
    change_date TIMESTAMP,
    user_id STRING,
    page STRING
) CLUSTER BY (change_date, user_id)
```
- `CLUSTER BY (change_date, user_id)` — 이건 "파티션을 이렇게 나눠라"가 아니라 "이 컬럼들이 자주 조회되니까, 물리적으로 서로 가까운 값끼리 모아둬라"는 힌트
- 데이터가 명확한 그룹(파티션)으로 안 나뉨. 대신 Delta Lake가 알아서 파일들을 재구성하면서, `change_date`와 `user_id` 값이 비슷한 row들을 같은 파일에 계속 몰아넣음
- 데이터가 쓰일 때마다(INSERT, UPDATE 등) Delta Lake가 자동으로 "지금 이 파일 배치가 최적인가?"를 판단해서 필요하면 재정렬/재배치

### **3. AWS Athena / Glue — Partition Projection**

- 정체: 파티션 목록을 메타스토어에 실제로 등록해두지 않고, 파티션 값의 규칙(예: 날짜 범위, 증가 패턴)만 설정해두면 쿼리 시점에 즉석으로 계산해서 어떤 파티션을 읽어야 할지 판단하는 기능
- 이전 한계: PostgreSQL 예시처럼 파티션이 늘어날 때마다 메타스토어에 파티션 정보를 등록(`MSCK REPAIR TABLE`, `ADD PARTITION` 등)하는 별도 작업이 필요했음. 파티션 수가 많아지면 이 메타데이터 관리 자체가 병목이 됨(앞서 본 granularity 문제)
- 전: 
	- 파티션 늘 때마다 메타스토어에 일일이 등록(`ADD PARTITION` 등) 필요. 등록 안 하면 조회 결과가 비어버림
- 후: 
	- 파티션 값의 규칙만 설정해두면 쿼리 시점에 즉석 계산. 등록 자체가 필요 없음
- 체감: 
	- 매시간 자동 생성되는 파티션에서 운영 부담이 확 줄어듦




	**AWS Athena Partition Projection — 예시로**
	
	**먼저, 이게 왜 필요한지 — 메타스토어 등록 방식(기존)부터**
	
	- Athena/Glue는 S3에 저장된 파일들을 테이블처럼 조회하는 서비스
	- S3 자체는 그냥 파일 저장소일 뿐이라, "어떤 경로가 어떤 파티션에 해당하는지"를 Athena가 알려면 별도로 등록을 해줘야 함 — 이 등록 정보를 담아두는 곳이 Glue Data Catalog(메타스토어)
	
	기존 방식 예시:
	```sql
	CREATE EXTERNAL TABLE visits (visit_id STRING, page STRING)
	PARTITIONED BY (event_date STRING)
	LOCATION 's3://my-bucket/visits/'
	```

	```sql
	ALTER TABLE visits ADD PARTITION (event_date='2024-12-31')
	LOCATION 's3://my-bucket/visits/event_date=2024-12-31/'
	```
	- 새 날짜의 데이터가 S3에 새로 쌓일 때마다, **이 `ADD PARTITION` 문을 매번 실행해서 메타스토어에 "이 경로도 파티션이다"라고 등록**해줘야 함
	- 매일 자동으로 새 파티션 폴더가 생기는 파이프라인이면, 이 등록 작업도 매일 자동화된 스크립트로 돌려야 함 (`MSCK REPAIR TABLE`로 한 번에 스캔해서 등록하는 방법도 있지만, 파티션 수가 많아지면 이 스캔 자체가 느려짐)
	- 등록 안 하면? S3에 파일은 있는데 Athena 쿼리에는 그 데이터가 아예 안 잡힘(빈 결과)
	
	**Partition Projection — 규칙만 정의**
	```sql
	CREATE EXTERNAL TABLE visits (visit_id STRING, page STRING)
	PARTITIONED BY (event_date STRING)
	LOCATION 's3://my-bucket/visits/'
	TBLPROPERTIES (
	    'projection.enabled' = 'true',
	    'projection.event_date.type' = 'date',
	    'projection.event_date.range' = '2024-01-01,NOW',
	    'projection.event_date.format' = 'yyyy-MM-dd',
	    'storage.location.template' = 's3://my-bucket/visits/event_date=${event_date}/'
	)
	```
	- 이 코드가 뭘 하는 코드인지: 파티션을 하나하나 등록하는 대신, "event_date는 2024-01-01부터 지금까지 매일 하나씩 생기고, 경로 패턴은 이렇게 생겼다"는 **규칙**만 선언
	- `projection.event_date.range = '2024-01-01,NOW'`: 파티션 값의 범위(2024-01-01부터 현재까지)
	- `storage.location.template`: 특정 event_date 값이 주어지면 S3 경로가 어떻게 생성되는지 패턴
	
	**쿼리할 때 실제로 뭐가 일어나는지**
	```sql
	SELECT * FROM visits WHERE event_date = '2024-12-31';
	```
	1. Athena가 메타스토어에서 "2024-12-31 파티션이 등록돼있나" 찾지 않음
	2. 대신 `TBLPROPERTIES`에 정의된 규칙으로 즉석 계산: "event_date=2024-12-31이면 경로는 `s3://my-bucket/visits/event_date=2024-12-31/`이겠구나"
	3. 이 경로가 실제로 S3에 존재하는지 확인하고 바로 읽음
	4. **메타스토어에 이 파티션이 등록돼있는지 여부와 무관하게 동작** — 등록 절차 자체가 필요 없음

**실무 선호 정리**
- Iceberg/Delta Lake 같은 최신 테이블 포맷을 이미 쓰는 환경 → 1번, 2번이 자연스러운 선택. 파티션 관리 자체를 플랫폼에 위임
- S3 + Athena/Glue 같은 서버리스 쿼리 환경 → 3번(Partition Projection)으로 메타스토어 등록 부담을 없앰
- 셋 다 공통적으로 "사람이 파티션을 직접 설계·관리하던 부분을 플랫폼이 대신 처리"하는 방향으로 가고 있음

> (엔지니어 독백)
> 
> 옛날 Hive 스타일 파티셔닝 관리해본 사람이면 "파티션 하나 새로 생겼는데 메타스토어엔 안 잡혀서 쿼리가 빈 결과 준다"는 경험이 다들 있을 거다. 이 세 가지 트렌드가 결국 다 같은 문제를 겨냥한다 — "물리적으로 데이터를 어떻게 나눌지"와 "그 나눔을 어떻게 추적하고 관리할지"를 분리해서, 후자를 사람이 아니라 플랫폼이 맡게 만드는 방향이다. 신규 파이프라인을 설계한다면, Hive 스타일 정적 파티셔닝보다는 이 셋 중 하나를 기본으로 검토하는 게 실무에서 시행착오를 줄여준다.



<br><br><br>
<br>




# 패턴 #51: Vertical Partitioner (저장소용)

한 줄 정의: row를 가변(mutable)/불변(immutable) 속성 그룹으로 쪼개서 서로 다른 위치에 저장하는 패턴.

_(참고: 이름이 Chapter 7의 Vertical Partitioner와 같지만, 그건 보안/PII 삭제 목적이고 이건 저장 최적화 목적)_

## **(1) 문제상황**

핵심 고통: 매 방문마다 안 바뀌는 정보를(불변) 계속 중복 저장한다.

**지금 저장 구조 (분리 전) — visits 테이블 하나에 다 때려박음**
```
+----------+---------------------+-------------+
| visit_id |          event_time |        page |
+----------+---------------------+-------------+
| visit-1  | 2024-12-01 09:00:00 |   home.html |
| visit-1  | 2024-12-01 09:00:15 | product.html|
| visit-1  | 2024-12-01 09:01:02 |   cart.html |
| visit-1  | 2024-12-01 09:02:30 | payment.html|
+----------+---------------------+-------------+
```
- `event_time`, `page` → 매 row마다 다름 (가변 속성)
- `ip_address`, `user` → 같은 visit_id 안에서는 4개 row 다 동일한 값 (불변 속성인데 4번 중복 저장됨)

> (엔지니어 독백)
> 하루 방문이 수백만 건이고 방문당 평균 페이지 이동이 5~6번이면, ip_address 같은 불변 컬럼만 놓고 봐도 실제 필요한 저장량의 5~6배를 쓰고 있는 셈이다. 이게 하루이틀이 아니라 매일 누적되니까, 몇 달 지나면 이 중복분만 수십 GB씩 불필요하게 쌓인다.


**Vertical Partitioner 적용 후 — 가변/불변 그룹을 별도 테이블로 분리**

가변 속성 테이블 (visit_events) — 계속 append됨:

```
+----------+---------------------+-------------+
| visit_id |          event_time |        page |
+----------+---------------------+-------------+
| visit-1  | 2024-12-01 09:00:00 |   home.html |
| visit-1  | 2024-12-01 09:00:15 | product.html|
| visit-1  | 2024-12-01 09:01:02 |   cart.html |
| visit-1  | 2024-12-01 09:02:30 | payment.html|
+----------+---------------------+-------------+
```

불변 속성 테이블 (visit_context) — visit_id당 딱 한 row만:

```
+----------+-----------------+---------+
| visit_id |      ip_address |    user |
+----------+-----------------+---------+
| visit-1  | 203.0.113.45    | user_a  |
+----------+-----------------+---------+
```

- 두 테이블은 `visit_id`로 다시 join해서 합칠 수 있음
- ip_address/user는 이제 방문당 딱 1번만 저장 → 중복 4배 → 1배로 줄어듦

> (엔지니어 독백)
> 
> 유저 방문 로그를 저장하는데, 
> visit_time·visited_page처럼 방문마다 바뀌는 값과, 
> IP address처럼 그 방문 내내 안 바뀌는 값이 섞여있다. 
> 방문 하나에 페이지 이동이 여러 번 있으면, 
> 그때마다 같은 IP를 매번 다시 써서 저장하고 있었다 — 불필요한 중복이 계속 쌓이는 거다.

## **(2) 솔루션**

주요컨셉: 
	row를 가변/불변 속성 그룹으로 나누고, 나중에 다시 합칠 공통 키(visit_id)를 정한 뒤, 
	각 그룹을 별도 위치(테이블/디렉토리)에 저장한다.

**1단계: 속성 분류**
- 관련 있는 속성끼리 그룹으로 묶음
- 이번 케이스: 가변 그룹(event_time, page) / 불변 그룹(ip_address, user)
- 이 두 그룹을 나중에 다시 이어붙일 때 쓸 공통 키를 정함 → visit_id

**2단계: 분리 저장**
- 데이터 처리 잡이 이 그룹별로 각각 다른 위치(데이터 저장소의 테이블, 또는 파일시스템의 디렉토리)에 씀
- 앞서 도식으로 본 `visit_events` 테이블(가변)과 `visit_context` 테이블(불변)이 이 결과물

가변 속성 테이블 (visit_events) :
```
+----------+---------------------+-------------+
| visit_id |          event_time |        page |
+----------+---------------------+-------------+
| visit-1  | 2024-12-01 09:00:00 |   home.html |
| visit-1  | 2024-12-01 09:00:15 | product.html|
| visit-1  | 2024-12-01 09:01:02 |   cart.html |
| visit-1  | 2024-12-01 09:02:30 | payment.html|
+----------+---------------------+-------------+
```

불변 속성 테이블 (visit_context) — visit_id당 딱 한 row만:
```
+----------+-----------------+---------+
| visit_id |      ip_address |    user |
+----------+-----------------+---------+
| visit-1  | 203.0.113.45    | user_a  |
+----------+-----------------+---------+
```


**부가 이점 — 단순히 용량만 줄이는 게 아니다**
- row가 이미 나뉘어 있으니, 그룹별로 서로 다른 data retention 정책이나 접근 정책을 적용하기 쉬워짐
- 예: ip_address 같은 개인정보성 데이터는 90일 후 삭제, page 방문 로그는 1년 유지 — 이런 정책 차등 적용이 한 테이블에 다 있을 때보다 훨씬 간단해짐

**Horizontal Partitioner와의 차이 (파티셔닝 기준)**
- Horizontal: row를 통째로 다른 위치로 옮김 (예: 날짜별 테이블)
- Vertical: row 하나를 쪼개서 각 부분을 다른 위치에 씀
- 즉 "무엇을 기준으로 나누느냐"가 아니라 "row를 통째로 옮기느냐, 쪼개서 옮기느냐"가 핵심 차이


## **(3) 결과**

결론: 저장 용량은 줄지만, 그 대가로 조회할 때 여러 곳을 다시 이어붙여야 하는 부담이 생긴다.

1. **쿼리 성능 저하**
    - 배경: 
	    - row가 나뉘기 전엔 한 row 안에 모든 컬럼이 같이 있어서, 조회할 때 로컬에서 한 번에 읽으면 끝
    - 문제: 
	    - 나뉜 뒤에는 `visit_events`와 `visit_context`를 다시 합치려면 join이 필요. join은 다른 노드에 있는 데이터를 네트워크로 가져와야 하는 경우가 많아서, 원래 로컬 읽기였던 게 네트워크 트래픽을 타는 연산으로 바뀜
    - 실무 예시: 
	    - 분석가가 "이 방문에서 어떤 페이지를 봤고, 어느 IP에서 왔는지" 한 번에 보고 싶을 때마다 매번 두 테이블을 join해야 함. 예전엔 `SELECT * FROM visits` 한 번이면 끝났던 걸, 이제 매번 `JOIN` 절을 붙여야 함

2. **쿼리 복잡도 증가**
    - 배경: 
	    - 데이터가 물리적으로 나뉘어 있다는 사실을, 이 데이터를 쓰는 사람이 알아야 함
    - 문제: 
	    - "이 컬럼이 어느 테이블에 있는지" 매번 확인해야 하는 인지 부담
    - 대응: 
	    - 단일 진입점(view)으로 두 테이블을 미리 join해서 노출하거나, 데이터 카탈로그로 어느 컬럼이 어디 있는지 문서화

3. **폴리글랏 저장 환경에서 복잡도 가중**
	- 정의: 
		- 같은 데이터라도, 용도에 따라 서로 다른 종류의 데이터 저장소에 저장하는 방식
		- 예시:
			- 검색 기능에는 Elasticsearch(검색 특화 DB)에 저장
			- 실시간 조회에는 Redis(key-value store, 빠름)에 저장
			- 정형 분석에는 PostgreSQL(관계형 DB)에 저장
    - 배경: 
	    - 같은 데이터를 여러 종류의 저장소에 동시에 두는 경우가 있음 — 예를 들어 검색 기능엔 검색 특화 DB, 저지연 조회엔 key-value store, 이렇게 컨슈머 성격에 맞는 저장소를 따로 씀(polyglot persistence)
    - 문제: 
	    - 근데 같은 데이터가 Elasticsearch에도, Redis에도, PostgreSQL에도 각각 저장돼 있다면, 가변/불변 분리를 **저장소마다 따로** 해줘야 함. Elasticsearch용 분리 로직, Redis용 분리 로직, PostgreSQL용 분리 로직 이렇게 세 벌이 필요해지는 것
	    - 저장소가 하나 더 늘어날 때마다 관리해야 할 분리 파이프라인도 하나씩 더 늘어남
	    - 한마디로 귀찮다
    - 대응: 
	    - 한 잡이 row를 먼저 쪼개고, 그 이후 각 저장소별 consumer가 자기 저장소에 맞는 형태로 각각 write하는 구조로 설계

> 	(엔지니어 독백)
> 	처음 이 개념 들으면 "왜 굳이 여러 저장소에 같은 걸 중복해서 두나" 싶은데, 실무에서는 저장소마다 잘하는 게 다르기 때문이다. Redis는 빠른 단건 조회는 잘하지만 복잡한 검색은 못 하고, Elasticsearch는 검색은 잘하지만 트랜잭션 처리는 안 맞는다. 그래서 큰 서비스일수록 자연스럽게 polyglot 구조로 가게 되고, 이 상태에서 데이터 정리/분리 작업을 하려면 저장소 개수만큼 손이 더 간다는 게 여기서 말하는 문제다.


> (엔지니어 독백)
> 
> 이 패턴 도입 여부는 결국 "얼마나 자주 두 그룹을 다시 합쳐서 봐야 하는가"에 달려있다. 저장 비용은 확실히 줄지만, 매번 join 비용을 치러야 한다면 배보다 배꼽이 커질 수 있다. 실무에서는 불변 속성 그룹을 조회하는 빈도가 낮고(예: 감사 목적으로 가끔만 봄), 가변 속성만 자주 조회한다면 이 트레이드오프가 확실히 남는다. 반대로 매 쿼리마다 두 그룹을 다 봐야 한다면, 차라리 컬럼형 저장 포맷(Parquet 등)으로 컬럼 단위 프루닝만 활용하는 게 더 나을 수 있다.





## **(4) 예시**

**원본 데이터 (visits)**
visit-1의 두 row는 페이지만 다르고 user/context 정보는 동일하게 중복돼있다는 걸 보여주는 표:

|visit_id|user_id|event_time|page|context.user.login|context.user.email|context.technical.browser|context.technical.browser_version|
|---|---|---|---|---|---|---|---|
|visit-1|user_a|2024-12-01 09:00:00|home.html|user_a|[a@x.com](mailto:a@x.com)|Chrome|120|
|visit-1|user_a|2024-12-01 09:00:15|product.html|user_a|[a@x.com](mailto:a@x.com)|Chrome|120|
|visit-2|user_b|2024-12-01 10:00:00|home.html|user_b|[b@x.com](mailto:b@x.com)|Safari|17|

- visit-1 두 row 다 login/email/browser/browser_version이 완전히 동일 → 이 부분이 (1)문제상황에서 말한 중복


### **기술1: Apache Spark — user/technical context 분리**

- visits 데이터를 읽어서 user 정보, technical 정보, 나머지를 각각 다른 테이블에 씀

```python
visits = spark_session.read.schema(visit_schema).json(input_location)
visits.persist()
```

- 이 코드가 뭘 하는 코드인지: 
	- visits 데이터를 읽고, `persist()`로 메모리에 캐싱
- `persist()`를 왜 부르는지: 
	- 아래에서 이 `visits` 데이터를 3번(technical 제외본, user 테이블, technical 테이블) 반복해서 씀. `persist()` 없으면 Spark가 매번 원본 파일을 다시 읽어들임 — lazy evaluation 특성상 동일 DataFrame을 여러 action에서 재사용하면 그 앞의 transformation이 매번 재실행되기 때문. 한 번 캐싱해두면 재읽기 없이 재사용됨

```python
visits_without_user_technical_context = (visits.drop('user_id')
    .withColumn('context', F.col('context').dropFields('user'))
    .withColumn('context', F.col('context').dropFields('technical')))
visits_without_user_technical_context.write.format('delta').save(output_dir)
```
- 이 코드가 뭘 하는 코드인지: 
	- user_id 컬럼과, context 안의 user/technical 필드를 제거한 "나머지" 데이터를 만들어서 저장
- `dropFields('user')`: 
	- `context`가 struct(중첩 구조) 타입인데, 그 안의 `user`라는 하위 필드만 제거. 컬럼 전체가 아니라 struct 내부 필드 단위로 제거하는 함수

→ **결과 테이블 1: visits_without_user_technical_context**
```
+----------+---------------------+-------------+---------+
| visit_id |          event_time |        page |  context|
+----------+---------------------+-------------+---------+
| visit-1  | 2024-12-01 09:00:00 |   home.html |       {}|
| visit-1  | 2024-12-01 09:00:15 | product.html|       {}|
| visit-2  | 2024-12-01 10:00:00 |   home.html |       {}|
+----------+---------------------+-------------+---------+
```
- user_id 컬럼 자체가 사라졌고, context는 user/technical 필드가 다 빠져서 빈 구조체만 남음. 페이지 이동마다 중복되던 무거운 정보가 이제 없음





```python
(visits.selectExpr('visit_id', 'context.user.*', 'user_id').dropDuplicates()
    .write.format('delta').save(get_delta_users_table_dir()))
```
- 이 코드가 뭘 하는 코드인지: 
	- visit_id + user 관련 필드만 뽑아서 별도 user 테이블에 저장
- `context.user.*`: 
	- struct 안의 user 하위 필드들을 전부 펼쳐서(flatten) 가져옴
- `dropDuplicates()`: 
	- 같은 visit_id의 user 정보가 여러 row에 중복돼 있을 수 있으니 중복 제거 — 이게 바로 (1)문제상황에서 본 "IP address가 여러 번 중복 저장되는 문제"를 여기서 해결하는 부분

→ **결과 테이블 2: users**
```
+----------+---------+------------+
| visit_id |   login |       email|
+----------+---------+------------+
| visit-1  | user_a  |  a@x.com   |
| visit-2  | user_b  |  b@x.com   |
+----------+---------+------------+
```
- 원본에서 visit-1의 user 정보가 2번 나왔는데, 여기선 1번만 남음 (dropDuplicates 효과)



```python
(visits.selectExpr('visit_id', 'context.technical.*').dropDuplicates()
    .write.format('delta').save(get_delta_technical_table_dir()))
visits.unpersist()
```
- technical 정보도 동일한 방식으로 별도 테이블에 저장
- `unpersist()`: 
	- 다 쓴 캐시를 메모리에서 해제. 안 하면 이후 잡에서도 이 캐시가 메모리를 계속 차지함
- ★ 핵심: 
	- `dropDuplicates()` — 이게 없으면 user/technical 테이블도 원본만큼 중복이 그대로 남아서, 애초에 이 패턴을 쓴 의미가 없어짐

→ **결과 테이블 3: technical**
```
+----------+---------+----------------+
| visit_id | browser | browser_version|
+----------+---------+----------------+
| visit-1  | Chrome  |             120|
| visit-2  | Safari  |              17|
+----------+---------+----------------+
```
- ★ 핵심: `dropDuplicates()` — 이게 없으면 user/technical 테이블도 원본만큼 중복이 그대로 남아서, 이 패턴을 쓴 의미가 없어짐

### **기술2: PostgreSQL — INSERT INTO...SELECT FROM**

```sql
INSERT INTO dedp.technical (visit_id, browser, browser_version)
(SELECT DISTINCT visit_id, context->'technical'->>'browser',
    context->'technical'->>'browser_version'
    FROM dedp.visits_all);
```
- 이 코드가 뭘 하는 코드인지: 
	- 원본 `visits_all` 테이블에서 기술 정보만 뽑아서 `technical` 테이블에 삽입
	- `INSERT INTO ... (SELECT ...)`: 
		- 넣을 값을 하나씩 나열하는 대신, SELECT 쿼리 결과를 통째로 삽입하는 문법
	- `context->'technical'->>'browser'`:
		- JSON 컬럼에서 `technical.browser` 값을 꺼내는 PostgreSQL 연산자
	- `DISTINCT`: 
		- Spark의 `dropDuplicates()`와 같은 역할 — 중복 제거

### **기술3: PostgreSQL — CTAS(CREATE TABLE AS SELECT)**
```sql
CREATE TABLE dedp.technical_select AS (
    SELECT DISTINCT visit_id, context->'technical'->>'browser' AS browser
    FROM dedp.visits_all
);
```
- 이 코드가 뭘 하는 코드인지: 
	- 기존 테이블에 INSERT하는 대신, SELECT 결과로 새 테이블을 아예 새로 생성
- 기술2와 차이: 
	- 기술2는 이미 존재하는 테이블에 데이터를 추가하는 것이고, 이건 테이블 자체를 처음부터 새로 만드는 것

**분류 요약**

| 기술                              | 방식                    | 적합한 상황                         |
| ------------------------------- | --------------------- | ------------------------------ |
| Apache Spark                    | drop/select + persist | 대용량 배치 처리, 여러 출력 테이블 동시 생성     |
| PostgreSQL INSERT INTO...SELECT | 기존 테이블에 추가            | 이미 만들어진 테이블에 지속적으로 데이터 채워야 할 때 |
| PostgreSQL CTAS                 | 새 테이블 생성              | 처음 한 번 테이블 구조를 정의하며 만들 때       |


## **(5)최신트렌드**

**1. Delta Lake / Iceberg — MERGE 기반 중복 제거 자동화**
- 전: 
	- `dropDuplicates()`처럼 매 배치마다 직접 중복 제거 로직을 짜야 했음. 이미 저장된 테이블에 새 배치를 반영할 때도 수동으로 병합 로직 필요
- 후: 
	- `MERGE INTO` 구문으로 "이 키가 이미 있으면 업데이트, 없으면 삽입"을 한 줄로 처리. 중복 제거와 최신 상태 유지를 테이블 엔진이 담당
- 체감: 
	- user_context처럼 계속 갱신되는 불변 속성 테이블을 유지보수할 때, 매번 커스텀 병합 코드 안 짜도 됨


**2. Databricks Unity Catalog — 뷰(View)로 조인 감춤**
- 전: 
	- 나뉜 테이블을 조회하려면 컨슈머가 매번 join을 직접 작성해야 함 → (3)결과에서 본 쿼리 복잡도 문제
- 후: 
	- 카탈로그 레벨에서 미리 join된 view를 만들어두고, 컨슈머는 이 view 하나만 조회
- 체감: 
	- 데이터가 물리적으로 나뉘어 있다는 사실 자체를 컨슈머가 몰라도 됨


**3. Feature Store (Feast, Databricks Feature Store 등)**
- 정체: 
	- 가변/불변 속성을 아예 처음부터 분리해서 관리하도록 설계된 전용 저장 계층. ML 피처처럼 "자주 안 바뀌는 속성"과 "실시간으로 바뀌는 속성"을 구조적으로 나눠서 저장
- 전: 
	- 이런 분리를 직접 파이프라인으로 구현해야 했음(오늘 본 예시처럼)
- 후: 
	- 저장소 자체가 이 분리 개념을 내장하고 있어서, 속성을 등록할 때부터 "이건 실시간(online) 속성, 이건 배치(offline) 속성"으로 구분해서 관리
- 체감: 
	- ML 관련 파이프라인이면, Vertical Partitioner를 직접 구현하는 대신 이런 전용 저장소로 아예 옮기는 경우가 많음

> (엔지니어 독백)
> 
> 세 가지 트렌드 다 결국 "나눠 저장하는 부담을 인프라가 대신 지게 한다"는 방향이다. MERGE는 쓰기 쪽 부담을, 뷰는 읽기 쪽 부담을 줄여준다. Feature Store는 아예 이 분리 자체를 저장소 설계 철학으로 내장한 케이스다. 이 패턴을 직접 구현하기 전에, 혹시 이미 이런 기능을 가진 저장소를 쓰고 있는 건 아닌지부터 확인하는 게 실무에서 시간을 아낀다.





# **패턴 #52: Bucket (버킷)**

한 줄 정의: 고카디널리티 컬럼을 파티션처럼 각자 다른 위치에 두는 대신, 해시로 몇 개 그룹(bucket)으로 묶어서 같은 공간에 colocate하는 패턴.
- Colocate(콜로케이트) = 같은 물리적 위치에 함께 둔다

## **(1) 문제상황**

핵심 고통: 쿼리에 자주 쓰이는 컬럼인데, 카디널리티가 너무 높아서 파티션 키로 못 쓴다.

> (엔지니어 독백)
> 
> 쿼리의 80%가 특정 비즈니스 속성(예: user_id)을 조건절에 쓰고 있어서, 처음엔 이걸 파티션 키로 잡으려고 했다. 근데 이 속성은 유저마다 값이 다 다른 고카디널리티 컬럼이다. 파티션 키로 쓰면 유저 수만큼 파티션이 생겨서, 결국 데이터 저장소의 메타데이터 한계에 부딪힌다 (Horizontal Partitioner의 granularity 문제와 똑같은 이유).
> 
> 그렇다고 이 컬럼을 최적화 안 하고 두면, 쿼리 80%가 여전히 느리다. 파티셔닝처럼 "값마다 다른 위치"가 아니라, "값들을 몇 개 그룹으로 묶어서 그 그룹 단위로 위치를 정하는" 방법이 필요하다 — 이게 Bucket이 푸는 지점이다.

**Horizontal Partitioner (파티셔닝) — 값마다 다른 위치**
user_id로 파티셔닝하면 이렇게 됨:

|물리적 위치|저장된 user_id|
|---|---|
|/visits/user_id=user_1/|user_1의 row만|
|/visits/user_id=user_2/|user_2의 row만|
|/visits/user_id=user_3/|user_3의 row만|
|...|... (유저 100만 명 = 디렉토리 100만 개)|

- 유저 한 명 = 디렉토리 하나. 유저가 많을수록 디렉토리도 그만큼 늘어남 → 100만 개 디렉토리 문제


**Bucket — 여러 값을 그룹으로 묶어서 같은 위치**
같은 user_id를 bucket 4개로 묶으면 (`hash(user_id) % 4`):

|bucket 번호|계산|저장된 user_id|
|---|---|---|
|bucket_0|hash(user_1)%4=0, hash(user_5)%4=0|user_1, user_5, user_9...|
|bucket_1|hash(user_2)%4=1|user_2, user_6, user_10...|
|bucket_2|hash(user_3)%4=2|user_3, user_7, user_11...|
|bucket_3|hash(user_4)%4=3|user_4, user_8, user_12...|

- 물리적 위치는 딱 4개뿐. 유저가 100만 명이든 1000만 명이든 bucket 개수(4개)는 안 늘어남
- 대신 하나의 bucket 안에는 서로 다른 여러 user_id가 섞여 들어감

**조회할 때 무슨 일이 일어나는지**
- `WHERE user_id = 'user_3'`로 조회하면
- 쿼리 엔진이 `hash('user_3') % 4 = 2`를 계산 → bucket_2 위치 하나만 읽음 (나머지 bucket_0, 1, 3은 아예 안 건드림)
- 이게 **bucket pruning** — 파티셔닝의 partition pruning과 같은 원리, 대상이 파티션이 아니라 bucket일 뿐

> (엔지니어 독백)
> 결국 핵심은 "값 하나당 위치 하나"(파티셔닝) 대 "여러 값을 위치 하나에 몰아넣기"(버킷)의 차이다. 파티셔닝은 위치 개수가 카디널리티에 비례해서 늘어나는 게 문제였는데, 버킷은 위치 개수(bucket 수)를 내가 미리 고정해두기 때문에 카디널리티가 아무리 높아도 위치 개수는 안 변한다. 대신 bucket 하나 안에 여러 값이 섞이니까, 파티셔닝만큼 완벽하게 좁혀지진 않고 "그 bucket 안에서 한 번 더 걸러야" 한다는 차이는 있다.


## **(2)솔루션**

주요컨셉: bucket 컬럼(그룹핑 기준 컬럼)과 bucket 개수를 정해서, `hash(값) % bucket 개수` 계산 결과가 같은 row들을 같은 물리적 위치에 모아 저장한다.

**1단계: bucket 컬럼 정하기**
- 쿼리에서 자주 조건절로 쓰이는 고카디널리티 컬럼을 선정 (예: user_id)
- 이미 파티셔닝/버티컬 파티셔닝이 적용된 데이터셋이면, bucket 컬럼은 파티션 키(1차 키) 아래의 2차 그룹핑 키 역할을 함

**2단계: bucket 개수 정하기**
- bucket 키의 카디널리티에 따라 결정. 카디널리티가 높을수록(값 종류가 많을수록) bucket 개수를 늘리는 게 일반적
- bucket 개수가 많으면 → bucket 하나당 크기는 작아짐(더 세밀하게 나뉨)
- bucket 개수가 적으면 → bucket 하나당 크기는 커짐(덜 세밀하게 나뉨)
- 계산 방식: `hash(key) % bucket_수`

**이 구조로 얻는 최적화 두 가지**
1. **Bucket Pruning**
    - bucket 컬럼이 쿼리 조건절에 쓰이면, 쿼리 엔진이 이 컬럼 값의 hash를 계산해서 해당 bucket만 읽고 나머지는 건너뜀
    - 앞서 표에서 본 `WHERE user_id = 'user_3'` 예시가 이거
2. **Shuffle(네트워크 교환) 제거**
    - JOIN 연산에서, 양쪽 테이블이 **동일한 bucket 컬럼 + 동일한 bucket 개수**로 버킷팅돼있으면, 같은 bucket 번호끼리는 이미 같은 값들이 모여있다는 게 보장됨
    - 그래서 join 시 데이터를 네트워크로 재분배(shuffle)할 필요 없이, 같은 bucket 번호 파일끼리 바로 로컬에서 매칭 가능
    - Distributed Aggregator 패턴에서 본 "다른 노드 데이터를 끌어와야 하는" 네트워크 비용을, 버킷 구조가 사전에 없애주는 것

**역사적 배경**
- Bucket 기능은 원래 Apache Hive에서 대중화됐고, 지금은 Apache Spark, AWS Athena 등 현대 데이터 솔루션에 통합되어 있음

## (3)결과

결론: Horizontal Partitioner와 마찬가지로, 데이터가 정적(static)이라는 게 여기서도 제일 큰 문제다.

1. **Mutability (버킷 스키마 변경의 어려움)**
    - 배경: 버킷팅 컬럼이나 버킷 개수를 한번 정하면, 그 기준으로 이미 데이터가 물리적으로 다 배치돼있음
    - 문제: 기술적으로 버킷 컬럼이나 버킷 개수를 바꾸는 것 자체는 가능하지만, 기존 데이터를 전부 다시 읽어서 새 기준으로 재배치(backfill)해야 함 — 비용이 큼
    - 대응: 처음 설계 시점에 데이터 증가 추이를 고려해서 신중하게 정하는 수밖에 없음 (뒤에서 이어지는 bucket size 문제와 연결됨)
2. **Bucket 크기(개수) 설정의 딜레마**
    - 배경: 버킷 개수를 지금 당장 정해야 하는데, 미래에 데이터가 얼마나 늘어날지 모름
    - 문제: 두 가지 선택 다 리스크가 있음
        - 지금 데이터 volume 기준으로 개수를 정하면 → 나중에 데이터가 늘어났을 때 bucket 하나당 크기가 너무 커짐 (bucket이 커지면 그 안에서 걸러내야 할 양도 늘어나서 pruning 효과가 약해짐)
        - 미래 증가량을 미리 예측해서 넉넉하게 개수를 잡으면 → 예측이 틀릴 수 있고, 그동안은 필요 이상으로 많은 bucket이 만들어져서 오히려 작은 파일 문제가 생길 수 있음
    - 대응: 책에서도 "두 방법 다 나름의 함정이 있다"고 인정 — 완벽한 해법은 없고, 데이터 증가 패턴을 주기적으로 재검토하면서 조정하는 수밖에 없음

> (엔지니어 독백)
> 
> 버킷 개수 정하는 게 파티션 키 정하는 것보다 더 애매하다고 느꼈다. 파티션은 "이 컬럼의 distinct 값이 몇 개나 되나"만 보면 되는데, 버킷은 거기에 "몇 개로 쪼갤지"라는 숫자까지 따로 정해야 한다. 처음 설계할 때는 현재 데이터 volume에 여유분(예: 2~3배)을 곱해서 버킷 수를 잡고, 이후 실제 트래픽 늘어나는 걸 보면서 재배치 시점을 미리 계획해두는 방식을 썼다. "한 번 정하면 안 바뀐다"는 전제를 갖고 처음부터 신중하게 접근하는 게 나중에 backfill 비용을 아끼는 길이었다.


## **(4)예시**

### **기술1: AWS Athena — CLUSTERED BY**

S3에 저장된 테이블에 버킷팅 규칙을 논리적으로 적용. Athena는 서버리스 쿼리 서비스라 데이터를 직접 쓰지 않고, 이미 있는 테이블에 버킷팅 로직만 적용한다.

```sql
CREATE EXTERNAL TABLE visits (...) 
CLUSTERED BY (`user_id`) INTO 50 BUCKETS
TBLPROPERTIES ('bucketing_format' = 'spark')
```

이 코드가 뭘 하는 코드인지:
`visits`라는 외부 테이블을 정의하면서, `user_id` 컬럼을 기준으로 50개 bucket으로 나눈다고 선언한다.

각 부분 설명:
- `CLUSTERED BY (user_id)` — 이 컬럼이 bucket 컬럼이라는 선언
- `INTO 50 BUCKETS` — bucket 개수를 50개로 고정
- `TBLPROPERTIES ('bucketing_format' = 'spark')` — 이 버킷팅이 Spark가 만든 형식과 호환되도록 지정. Athena와 Spark가 같은 해시 방식을 쓰도록 맞추는 설정

★ 핵심: Athena는 데이터를 쓰지 않으므로, 이 테이블에 `INSERT INTO`를 시도하면 에러가 난다. 이미 S3에 존재하는, 다른 도구(Spark 등)로 버킷팅되어 저장된 데이터에 대해 "이 데이터는 이런 규칙으로 버킷팅돼있다"는 걸 Athena한테 알려주는 선언일 뿐이다.

> (엔지니어 독백)
> 
> 처음 혼란스러웠던 게, Athena가 버킷팅을 "적용"한다고 해서 데이터를 직접 재배치해줄 거라 생각했다는 점이다. 실제론 아니다. Athena는 그냥 "이 테이블은 이런 규칙으로 이미 나뉘어 있다"는 메타데이터만 갖고, 조회할 때 그 규칙을 활용해서 bucket pruning만 해준다. 실제 데이터 배치는 Spark 같은 쓰기 엔진이 담당한다.

---

### **기술2: Apache Spark — bucketBy**

실제로 데이터를 버킷팅해서 저장하는 쓰기 작업이다.
```python
input_dataset.write.bucketBy(50, 'user_id').saveAsTable(table_name)
```

이 코드가 뭘 하는 코드인지:
`input_dataset`을 `user_id` 기준으로 50개 bucket에 나눠서 테이블로 저장한다.

핵심 동작:
`bucketBy(50, 'user_id')`는 (2)솔루션에서 본 `hash(user_id) % 50` 계산을 내부적으로 적용해서, 같은 결과값을 가진 row들을 같은 물리 파일에 쓴다.

★ 핵심: 숫자(50)와 컬럼(user_id) 두 인자가 전체 버킷팅 규칙을 결정한다. 이 두 값이 나중에 Athena `CLUSTERED BY` 선언과 정확히 일치해야, Athena가 이 데이터를 올바르게 pruning할 수 있다.

> (엔지니어 독백)
> `bucketBy`와 Athena `CLUSTERED BY`가 세트로 움직인다는 걸 이해하는 게 중요했다. Spark가 쓰기를 담당하고, Athena가 읽기(선언 기반 pruning)를 담당하는 분업 구조다. 숫자나 컬럼이 둘 사이에 안 맞으면, Athena는 잘못된 가정으로 pruning을 시도하다가 엉뚱한 결과를 내거나 최적화가 아예 안 먹힐 수 있다.

---

**분류 요약**

|기술|역할|비고|
|---|---|---|
|Apache Spark (`bucketBy`)|실제 데이터를 버킷팅해서 물리적으로 저장|쓰기 담당|
|AWS Athena (`CLUSTERED BY`)|이미 버킷팅된 데이터에 규칙을 선언해서 조회 최적화|읽기 담당, 직접 쓰기 불가|


##  (5)최신트렌드

### **1. Databricks Liquid Clustering**
- 전:
    - 버킷 개수를 미리 고정해야 함
    - 나중에 바꾸려면 전체 데이터를 재배치(backfill)해야 함
    - (3)결과에서 본 mutability/bucket size 딜레마가 그대로 발생
- 후:
    - 클러스터링 키만 지정하면, 데이터가 쓰일 때마다 최적의 배치를 자동으로 재조정
    - 고정된 bucket 개수 개념 자체가 없어짐
- 체감:
    - bucket 개수를 미리 못 정해서 겪던 딜레마 자체가 사라짐
    - 데이터 늘어나는 추이를 지켜보며 재배치 시점을 계획할 필요가 없어짐


### **2. Apache Iceberg — Bucket Transform**
- 전:
    - Hive/Spark 스타일 버킷팅은 버킷 컬럼과 개수가 테이블 스키마에 고정됨
    - 바꾸려면 costly backfill 필요
- 후:
    - 버킷팅을 partition evolution과 결합
    - 버킷 개수를 나중에 바꿔도 기존 데이터는 그대로 두고, 새 데이터부터만 새 규칙 적용
- 체감:
    - 버킷 개수를 잘못 예측해도 전체 재작업 없이 점진적으로 조정 가능

	**1단계: 클러스터링 키 지정 (사람이 하는 유일한 일)**
	```sql
	CREATE TABLE visits (
	    visit_id STRING, event_time TIMESTAMP, user_id STRING, page STRING
	) CLUSTER BY (user_id)
	```
	- `CLUSTER BY (user_id)` — "이 컬럼으로 자주 조회되니, 물리적으로 잘 묶어달라"는 힌트만 줌. bucket 개수 같은 숫자는 아예 안 정함
	
	**2단계: 데이터를 쓸 때 — Hilbert Curve / 클러스터링 알고리즘으로 정렬**
	- 전통적 버킷팅: `hash(user_id) % N` 이라는 고정 공식으로 값을 N개 그룹 중 하나에 딱 배정
	- Liquid Clustering: user_id 값들을 공간 채움 곡선(space-filling curve, Z-order와 비슷한 계열의 Hilbert curve 등) 알고리즘으로 정렬해서, **값이 비슷한 row들이 물리적으로 가까운 파일에 놓이도록 배치**
	- 이 방식은 "몇 개 그룹으로 나눌지"를 미리 정할 필요가 없음 — 그냥 "가까운 값끼리 뭉치게" 하는 정렬 방식이기 때문에, 데이터가 늘어나도 그 정렬 로직 자체는 그대로 적용됨
	
	**3단계: 데이터가 계속 들어올 때 — 점진적 재구성(incremental repartitioning)**
	- 매번 전체 테이블을 다시 재배치하는 게 아니라, **새로 들어온 데이터가 있는 부분만** 기존 배치와 맞춰서 국소적으로 재정렬
	- 이게 "backfill 없이 자동 최적화"가 되는 핵심 이유: 예전 버킷팅처럼 "개수를 바꾸면 전체를 다시 계산"하는 게 아니라, 새 데이터가 들어올 때마다 그 근처 파일들만 조금씩 다시 정리
### **3. Z-Ordering (Delta Lake `OPTIMIZE ZORDER BY`)**
- 정체:
    - 여러 컬럼을 동시에 고려해서 물리적으로 가까운 값끼리 묶어 재배치하는 다차원 정렬 기법
- 전:
    - 버킷은 컬럼 하나(또는 소수) 기준으로만 그룹핑 가능
    - 쿼리가 여러 컬럼을 동시에 조건으로 걸면 버킷팅 하나로는 커버 안 됨
- 후:
    - `OPTIMIZE table ZORDER BY (col1, col2)`처럼 여러 컬럼을 한 번에 최적화 대상으로 지정
    - 쿼리가 이 컬럼들을 조합해서 필터링해도 효율적으로 데이터 블록을 건너뜀
- 체감:
    - 버킷 컬럼 하나만으로는 부족한 다중 조건 쿼리 패턴에서 특히 유용


**실무 선호 정리**
- Databricks 환경이면 → 1번(Liquid Clustering)이 신규 테이블의 기본 선택
- Iceberg 기반 레이크하우스 → 2번으로 버킷 변경 부담을 줄임
- 쿼리 조건이 컬럼 하나가 아니라 여러 개를 조합하는 경우 → 3번(Z-Order) 고려

> (엔지니어 독백)
> 세 가지 다 결국 "버킷 개수를 미리 정하고 고정해야 한다"는 전통적 버킷팅의 제약을 깨는 방향이다. 1.Liquid Clustering은 개수 자체를 없앴고,
> 2.Iceberg Bucket Transform은 나중에 바꿔도 부담을 줄였고,
> 3.Z-Order는 애초에 컬럼 하나에 묶이는 한계를 넘어섰다. 
> 
> 신규 파이프라인이면 전통적 `bucketBy` 방식보다 이 셋 중 하나를 먼저 검토하는 게 실무에서 나중 삽질을 줄여준다.



# **패턴 #53: Sorter (소터)**

한 줄 정의: 저장 시점에 특정 컬럼 기준으로 row를 정렬해서, 조회 시 불필요한 데이터 블록을 건너뛸 수 있게 하는 패턴.

## **(1) 문제상황**

핵심 고통: 파티셔닝은 이미 했는데, 여전히 쿼리가 느리다.
- 유지보수 편하려고 주 단위 테이블로 데이터를 저장 (Fast Metadata Cleaner 패턴 활용)
- 일간 유지보수는 편해졌는데, 쿼리 실행 시간은 그대로
- 파티셔닝 전략 자체는 바꾸고 싶지 않은데, 데이터 접근 지연은 줄이고 싶음
- 다행히 유저 쿼리 패턴을 이미 알고 있음 — 대부분 event_time 컬럼으로 필터링하거나 정렬함
> 	(엔지니어 독백) 
> 	이건 Bucket 문제상황과 비슷한 결의 고민이다. 
> 	파티셔닝만으로는 "파티션 하나 안에서" 얼마나 빨리 원하는 row를 찾을지는 못 다룬다. 
> 	주 단위 파티션 하나에도 수백만 row가 있으면, 
> 	그 안에서 특정 시간대만 찾는 건 여전히 전체 스캔이다. 
> 	
> 	"쿼리가 이 컬럼을 자주 쓴다"는 걸 이미 알고 있으면, 
> 	그 컬럼 기준으로 데이터를 미리 정렬해두면 되지 않을까 — 이게 Sorter가 푸는 지점이다.


## (2)솔루션

주요컨셉: 자주 필터링/정렬에 쓰이는 컬럼을 식별해서, 테이블 생성 시점에 정렬 컬럼으로 선언한다. 이후 데이터 저장소가 이 순서대로 row를 정리해서 저장한다.

**1단계: 정렬 컬럼 식별**
- 쿼리 패턴을 분석해서, 어떤 컬럼이 필터링(WHERE)이나 정렬(ORDER BY)에 자주 쓰이는지 파악
- 이번 케이스: event_time

**2단계: 테이블 생성 시 정렬 컬럼 선언**
- 이후 데이터 저장소가 write되는 row들을 이 순서대로 정리해서 저장
- 저장 시 각 데이터 블록(파일)마다 담당하는 값의 범위가 메타데이터로 기록됨

**정렬 전 vs 정렬 후 — 실제로 뭐가 달라지는지**
정렬 안 된 상태 (주 단위 파티션 하나 안):

|파일|안에 담긴 event_time 범위|
|---|---|
|file_1.parquet|09:15, 14:20, 03:40, 22:10 (뒤섞임)|
|file_2.parquet|11:05, 01:30, 19:45, 07:20 (뒤섞임)|
|file_3.parquet|16:00, 08:10, 23:50, 04:15 (뒤섞임)|

- `WHERE event_time BETWEEN '08:00' AND '09:00'`로 조회 → 세 파일 다 그 범위에 해당하는 값이 섞여있을 가능성이 있어서 **파일 3개 다 열어봐야 함**

정렬된 상태 (event_time 기준으로 미리 정렬해서 저장):

|파일|안에 담긴 event_time 범위|
|---|---|
|file_1.parquet|00:00 ~ 07:59|
|file_2.parquet|08:00 ~ 15:59|
|file_3.parquet|16:00 ~ 23:59|

- 같은 조건으로 조회 → 이 범위가 file_2에만 들어있다는 걸 메타데이터(각 파일의 min/max 값)로 바로 알 수 있음 → **file_2만 열고, file_1과 file_3은 스킵**
- 이 메타데이터 기반 스킵이 Metadata Enhancer 패턴과 연결되는 지점

### **두 가지 정렬 방식**
1. **Lexicographical(사전식) 정렬**
    - 컬럼 하나 또는 여러 컬럼을 순서대로(1차 키, 2차 키...) 정렬
    - 여러 컬럼을 정렬 키로 쓰면 composite sort key라고 부름
    - 적합한 상황: 쿼리가 항상 앞쪽 정렬 키부터 순서대로 필터링하는 경우
2. **Curved Sort (곡선형 정렬) — 대표적으로 Z-order**
    - 사전식처럼 위→아래 한 방향이 아니라, 여러 차원(컬럼)을 동시에 고려해서 값이 서로 가까운 row끼리 묶는 방식
    - 적합한 상황: 여러 컬럼을 다양한 조합으로 필터링하는 쿼리가 많을 때

실무 선호: 쿼리가 거의 항상 같은 순서로 컬럼을 필터링하면(예: 항상 날짜 먼저, 그다음 유저) 사전식으로 충분. 쿼리가 컬럼 조합을 다양하게 섞어서 쓰면 Z-order가 더 안정적인 성능을 줌.

**1. Lexicographical(사전식) 정렬 — composite sort key 예시**

정렬 키를 `visit_time` → `page` 순서(1차 → 2차)로 지정했다고 하자:

|row|visit_time|page|
|---|---|---|
|1|09:00|cart|
|2|09:00|home|
|3|09:00|payment|
|4|10:00|cart|
|5|10:00|home|
|6|11:00|home|

- 정렬 순서: 먼저 visit_time으로 정렬하고, 같은 visit_time 안에서 page로 정렬
- **쿼리가 `visit_time`으로 필터링하면** (1차 키) → 관련 블록만 정확히 찾아서 스킵 가능
- **쿼리가 `visit_time` + `page`를 같이 필터링하면** (1차+2차 키 순서대로) → 여전히 정확하게 좁혀짐
- **쿼리가 `page`만 필터링하면** (2차 키만 단독으로) → 문제 발생. `page='home'`인 row가 09:00, 10:00, 11:00 블록 전체에 흩어져있어서, 결국 거의 모든 블록을 다 열어봐야 함

> (엔지니어 독백)
> 이게 사전식 정렬의 근본적 한계다. 전화번호부를 이름으로 정렬해두면 성(姓)으로 찾긴 쉬운데, 전화번호로 역방향 검색하려면 처음부터 끝까지 다 훑어야 하는 것과 같은 원리다. 1차 키를 안 쓰고 2차 키만 쓰는 쿼리가 많다면, 사전식 정렬은 그 쿼리엔 사실상 도움이 안 된다.

**2. Curved Sort (Z-order) — 같은 데이터, 다른 배치**

같은 두 컬럼(visit_time, page)을 Z-order로 정렬하면, 사전식처럼 한 방향으로 줄 세우지 않고 두 컬럼을 동시에 고려해서 값이 "가까운" row끼리 묶는다. 개념적으로 그려보면:

```
사전식 정렬 (visit_time 우선):
블록1: [09:00-cart, 09:00-home, 09:00-payment]
블록2: [10:00-cart, 10:00-home]
블록3: [11:00-home]
→ page='home'만 조회 시: 블록1, 2, 3 전부 열어야 함 (9개 row 관련 블록 다 스캔)

Z-order 정렬 (두 컬럼 동시 고려):
블록1: [09:00-cart, 09:00-home, 10:00-cart]
블록2: [09:00-payment, 10:00-home, 11:00-home]
→ page='home'만 조회 시: 블록1, 2 정도만 관련 (블록 수 자체가 줄어듦)
```

- 책에서 실측한 예시로는, 같은 조건에서 사전식은 데이터 블록 9개를 읽어야 했던 반면 Z-order는 7개만 읽음 — 컬럼 하나만 조회하는 쿼리에서 이 차이가 더 커짐

> (엔지니어 독백)
> Z-order는 "어느 컬럼이 1차, 2차인지"라는 우선순위 자체를 없애고, 여러 컬럼을 동등하게 고려해서 배치한다고 이해하면 된다. 그래서 쿼리가 컬럼 A만 쓰든, 컬럼 B만 쓰든, A+B를 같이 쓰든 사전식만큼 극단적으로 나빠지는 경우가 없다. 대신 완벽하게 한 컬럼만 최적화된 사전식만큼의 성능은 못 낸다 — 여러 컬럼을 동시에 만족시키려다 보니 어느 하나에 올인은 못 하는 트레이드오프다.


> Q."가까운" ? 뭐 어떻게? 

**Z-order가 "가까움"을 정의하는 방법 — Morton 코드(bit interleaving)**

**핵심 아이디어**: 여러 컬럼의 값을 각각 이진수(bit)로 바꾼 뒤, 그 비트들을 번갈아 섞어서(interleave) 하나의 숫자로 합친다. 이 합쳐진 숫자로 정렬하면, 원래 여러 컬럼에서 값이 비슷한 row들이 자연스럽게 옆자리에 모인다.

**아주 단순한 예시로 (2개 컬럼, 2비트씩)**

- visit_time을 0~3 사이 값으로, page를 0~3 사이 값으로 단순화했다고 하자 (실제로는 훨씬 큰 범위지만 원리는 동일)

|row|visit_time|page|visit_time 이진수|page 이진수|
|---|---|---|---|---|
|A|1|2|01|10|
|B|2|1|10|01|

**1단계: 각 값을 이진수로 표현**

- A: visit_time=1(01), page=2(10)
- B: visit_time=2(10), page=1(01)

**2단계: 두 이진수를 한 자리씩 번갈아 섞음(interleave)**

- 규칙: visit_time의 첫 비트, page의 첫 비트, visit_time의 둘째 비트, page의 둘째 비트... 순서로 섞음
- A: visit_time=`0`1, page=`1`0 → 섞으면 `0`(vt1) `1`(pg1) `1`(vt2) `0`(pg2) = `0110`
- B: visit_time=`1`0, page=`0`1 → 섞으면 `1`(vt1) `0`(pg1) `0`(vt2) `1`(pg2) = `1001`

**3단계: 이 섞인 숫자(Z-value)로 정렬**

- `0110`(=6, A) vs `1001`(=9, B) → A가 먼저, B가 나중

**왜 이게 "가까움"을 만드나**

- 이 Z-value로 데이터를 정렬해서 순서대로 나열하면, 2차원 평면(visit_time × page)에서 지그재그(Z자 모양)로 훑는 경로가 만들어짐
- 이 경로를 따라가면, **한 컬럼만 비슷하거나 두 컬럼 다 비슷한 row들이 이 순서상에서도 서로 가까운 위치에 놓이게 됨**
- 사전식 정렬은 vt를 완전히 다 정렬한 다음에야 page를 보므로, "page만 비슷한 row들"이 순서상 뿔뿔이 흩어짐. Z-order는 두 컬럼 비트를 같이 섞으니, 어느 한쪽만 비슷해도 어느 정도 순서상 가까워짐

> (엔지니어 독백)
> 
> 처음 이걸 이해할 때 "지그재그로 도장 찍듯 훑는 경로"를 상상하면 편했다. 사전식은 책장을 왼쪽부터 오른쪽으로, 한 줄 끝나면 다음 줄로 넘어가는 식이라 "같은 세로줄(page)"에 있는 값들은 책 전체에 흩어진다. Z-order는 지그재그로 작은 사각형 블록 단위로 훑고 넘어가서, 가로든 세로든 어느 정도는 뭉쳐있는 상태를 유지한다. 완벽하진 않지만, 어느 한 컬럼에 극단적으로 불리해지는 상황을 막아준다는 게 핵심이다.

**중요: 실무에서는 이 비트 연산을 직접 안 짠다**

- Delta Lake, Apache Iceberg 같은 테이블 포맷이 이 Z-order 계산을 내부적으로 구현해서 제공
- 개발자는 `OPTIMIZE table ZORDER BY (visit_time, page)`처럼 컬럼만 지정하면 됨


## (3)결과

### **1. Unsorted Segments (정렬 안 된 구간)**
배경: 정렬은 즉시 완료되는 작업이 아님. 새 레코드가 계속 들어올 때마다, 아직 정렬 안 된 블록들이 생김.
```
[정렬된 구간 - 이미 최적화됨]     [미정렬 구간 - 방금 들어온 데이터]
file_1: 00:00~07:59              file_new1: 뒤섞인 값들
file_2: 08:00~15:59       +      file_new2: 뒤섞인 값들
file_3: 16:00~23:59
```
- 쿼리가 08:00~09:00 구간을 찾으면, file_2는 바로 찾지만 file_new1, file_new2는 값이 섞여있어서 스킵 못 하고 다 열어봐야 함
- 즉 "정렬돼있다"고 선언했어도, 최근에 들어온 데이터는 그 혜택을 못 받는 구간이 항상 존재

대응: 정렬 작업을 쓰기 잡 안에 포함(즉시 정렬, 쓰기 느려짐) 또는 별도 스케줄로 분리(나중 정렬, 당분간 미정렬 구간 존재) — 둘 중 하나 감수

---

### **2. Composite Sort Key 사용 시 순서 제약**
정렬 키를 visit_time(1차) → page(2차) 순서로 지정한 상태:
```
쿼리가 "visit_time으로 필터링" (1차 키 사용)
→ 관련 블록만 정확히 찾음 (스킵 잘 됨)

쿼리가 "page만으로 필터링" (2차 키만 단독 사용)
→ page='home'인 row가 모든 시간대 블록에 흩어져있어서
   거의 모든 블록을 다 열어봐야 함 (스킵 거의 안 됨)
```
- 정렬을 선언했다고 해서 모든 쿼리가 혜택을 보는 게 아니라, **쿼리가 정렬 키 순서를 따라야만** 혜택을 봄

대응: 실제 쿼리 패턴 먼저 분석 → 컬럼 조합이 다양하면 Z-order 같은 curved sort로 전환 고려

---

### **3. Mutability (정렬 키 변경의 어려움)**
```
정렬 키를 event_time → user_id로 바꾸고 싶다
   ↓
기존에 event_time 기준으로 이미 정렬돼 저장된 파일들은
user_id 기준으로는 다시 뒤섞인 상태나 마찬가지
   ↓
테이블 전체를 user_id 기준으로 다시 정렬해서 재작성해야 함
(테이블 크기 클수록 이 작업 비용도 커짐)
```
대응: Bucket, Horizontal Partitioner와 마찬가지로 "한번 정하면 쉽게 안 바뀐다"는 전제로 초기 설계 시 신중하게 결정

> (엔지니어 독백)
> 
> 정렬 관련 패턴들이 공통적으로 갖는 문제가 "쓰기 시점의 비용을 읽기 성능과 맞바꾼다"는 거다. 매 쓰기마다 완벽하게 정렬하면 쓰기가 계속 느려지고, 아예 나중에 몰아서 정렬하면 그동안 읽기 성능 이득을 못 본다. 그래서 보통 "쓰기는 최대한 빠르게 하고, 별도 배치 잡(예: 야간에 도는 OPTIMIZE 작업)으로 주기적으로 정렬/압축을 몰아서 처리"하는 방식을 많이 쓴다.



## **(4)예시**

### **기술1: GCP BigQuery — Clustered Table (사전식 정렬)**
- 테이블 생성 시 CLUSTER BY로 정렬 컬럼을 선언

```sql
CREATE TABLE `dedp.visits.raw_visits`
PARTITION BY DATE(event_time)
CLUSTER BY visit_id, page
```

이 코드가 뭘 하는 코드인지:
`raw_visits` 테이블을 event_time의 날짜 단위로 파티셔닝하고, 그 안에서 visit_id → page 순서로 사전식 정렬(clustering)한다.

각 부분 설명:
- `PARTITION BY DATE(event_time)` — Horizontal Partitioner. 날짜별로 물리적 위치를 나눔
- `CLUSTER BY visit_id, page` — Sorter. 파티션 안에서 visit_id를 1차 키, page를 2차 키로 정렬

★ 핵심: `visit_id, page` 순서가 곧 정렬 우선순위. visit_id로 필터링하는 쿼리는 잘 최적화되지만, page만 단독으로 필터링하면 (3)결과에서 본 문제가 그대로 발생
- (엔지니어 독백) 파티셔닝과 클러스터링을 같이 쓰는 게 실무 표준이다. 파티셔닝으로 큰 덩어리를 먼저 나누고, 그 안에서 클러스터링으로 한 번 더 정렬하는 2단계 구조다. BigQuery 비용이 스캔한 데이터량 기준으로 청구되니, 이 조합이 곧바로 비용 절감으로 이어진다.

---

### **기술2: Delta Lake — Z-order Compaction (curved sort)**

BigQuery의 CLUSTER BY와 달리, page만 단독 필터링하는 쿼리도 있다면 Z-order를 씀
```python
DeltaTable.forPath(spark, output_dir) \
    .optimize().executeZOrderBy(['visit_id', 'page'])
```

이 코드가 뭘 하는 코드인지:

이미 저장된 Delta Lake 테이블을 대상으로, visit_id와 page 두 컬럼을 기준으로 Z-order 압축(재정렬)을 실행한다.

각 부분 설명:

- `DeltaTable.forPath(spark, output_dir)` — 정렬을 적용할 대상 테이블 경로 지정
- `.optimize()` — 파일 압축/재정리 작업을 시작하는 API 호출
- `.executeZOrderBy(['visit_id', 'page'])` — 이 두 컬럼을 기준으로 Z-order 알고리즘(앞서 본 bit interleaving)을 적용해서 파일들을 재배치

★ 핵심: `executeZOrderBy(['visit_id', 'page'])` — BigQuery의 `CLUSTER BY`와 인자는 똑같이 두 컬럼이지만, 정렬 방식 자체가 사전식이 아니라 curved(Z-order)라서 두 컬럼 중 어느 하나만 단독으로 필터링해도 어느 정도 성능 이점이 유지됨

- (엔지니어 독백) 이 작업이 (3)결과에서 본 "쓰기 비용"이 구체적으로 드러나는 지점이다. `optimize()`는 별도로 실행하는 배치성 작업이라, 매 write마다 자동으로 도는 게 아니다. 보통 야간 배치나 주기적 스케줄로 이 명령을 돌려서, 그동안 쌓인 unsorted segment들을 한꺼번에 정리하는 방식으로 운영한다.

---

**분류 요약**

|기술|정렬 방식|적합한 상황|
|---|---|---|
|BigQuery CLUSTER BY|사전식(lexicographical)|쿼리가 항상 정해진 컬럼 순서로 필터링|
|Delta Lake Z-order|curved sort|여러 컬럼을 다양한 조합으로 필터링|



## 5)최신트렌드

**1. Delta Lake / Iceberg — 자동 OPTIMIZE (Auto Compaction)**
- 전: 
	- `optimize().executeZOrderBy(...)`를 사람이 스케줄 잡아서 수동으로(또는 별도 배치 잡으로) 실행해야 했음. (3)결과에서 본 unsorted segments 문제를 관리하려면 이 스케줄 관리 자체가 운영 부담
- 후: 
	- 테이블 속성으로 auto-optimize를 켜두면, 쓰기가 일어날 때마다 백그라운드에서 자동으로 압축/정렬이 트리거됨
- 체감: 
	- 별도 배치 잡을 만들고 모니터링할 필요 없이, 정렬 상태가 지속적으로 유지됨




**2. Databricks Liquid Clustering — Sorter+Bucket 통합**
- 전: 
	- 사전식 정렬(Sorter)과 버킷팅(Bucket)이 각각 별도 개념으로, 상황에 따라 둘 중 하나를 선택하거나 따로 설정해야 했음
- 후: 
	- 클러스터링 키를 지정하면 정렬과 그룹핑 개념을 하나로 통합해서 자동 관리. Bucket 최신트렌드에서 본 것과 동일한 기능이 Sorter 문제도 같이 해결
- 체감: 
	- "정렬 키를 쓸지 버킷 키를 쓸지" 고민 자체가 줄어듦




**3. Apache Iceberg — Sort Order 메타데이터화**
- 전: 
	- 정렬 키를 바꾸려면 (3)결과에서 본 대로 테이블 전체를 재작성해야 했음
- 후: 
	- 정렬 순서(sort order)를 테이블 메타데이터에 버전으로 기록해서, 새로 쓰이는 데이터부터 새 정렬 규칙 적용 가능. Horizontal Partitioner에서 본 partition evolution과 같은 원리를 정렬에도 적용
- 체감: 
	- 정렬 키 변경 시 전체 재작성 비용을 어느 정도 회피 가능




**실무 선호 정리**
- Delta Lake/Iceberg 기반이면 → 1번(Auto Compaction)부터 켜두는 게 기본
- Databricks 환경 → 2번(Liquid Clustering)으로 Sorter/Bucket 고민을 한 번에 해결
- 정렬 키를 자주 바꿔야 하는 상황이면 → 3번(Sort Order 메타데이터화) 지원 여부 확인



> (엔지니어 독백)
> 
> Sorter 트렌드도 Bucket과 결이 같다. "언제 정렬할지"(자동화), "정렬 키를 어떻게 정할지"(Liquid Clustering으로 통합), "정렬 키를 나중에 바꿀 수 있을지"(메타데이터화) 세 가지 운영 부담을 플랫폼이 흡수하는 방향으로 가고 있다. 신규 테이블이면 수동 `optimize()` 스케줄링보다 auto-optimize부터 켜두고 시작하는 게 실무에서 유지보수 부담을 줄여준다.

1. 데이터 삭제 (Data Removal)
2. 접근 제어 (Access Control)
3. 데이터 보호 (Data Protection)
4. 연결 보안 (Connectivity)

**챕터 7 패턴 목록**

| 영역       | 패턴                                        | 한 줄 목적                |
| -------- | ----------------------------------------- | --------------------- |
| 1.데이터 삭제 | 패턴#39 Vertical Partitioner                | 데이터 레이아웃으로 삭제 비용 줄이기  |
|          | 패턴#40 In-Place Overwriter                 | 기존 데이터를 제자리에서 덮어써서 삭제 |
| 2.접근 제어  | 패턴#41 Fine-Grained Accessor for Tables    | 테이블 컬럼/행 단위 접근 제어     |
|          | 패턴#42 Fine-Grained Accessor for Resources | 클라우드 리소스 단위 접근 제어     |
| 3.데이터 보호 | 패턴#43 Encryptor                           | 저장/전송 데이터 암호화         |
|          | 패턴#44 Anonymizer                          | PII 완전 제거             |
|          | 패턴#45 Pseudo-Anonymizer                   | PII를 가짜 값으로 대체        |
| 4.연결 보안  | 패턴#46 Secrets Pointer                     | 코드 외부에 자격증명 보관        |
|          | 패턴#47 Secretless Connector                | 자격증명 없이 DB 연결         |



# #46 익명화기(Anonymizer)

한 줄 정의: 데이터셋에서 PII를 제거하거나 되돌릴 수 없게 변형해서, row 단위로 신원 식별이 불가능하게 만드는 패턴.

## **(1) 문제상황**

핵심 고통: 공유해야 할 데이터셋에 PII가 섞여있는데, 일부 유저는 제3자 공유에 동의 안 함.

- 마케팅팀이 외부 분석 업체와 계약. 유저 행동 데이터 분석해서 커뮤니케이션 전략 최적화 요청
- 데이터 엔지니어링팀이 데이터셋 열어보니 email, birthday, ssn 같은 PII 컬럼이 그대로 존재
- 일부 유저는 개인정보 제3자 공유에 동의 안 한 상태 → 규정(GDPR, CPPA 등) 위반 리스크
- 그냥 넘기면 안 되니, 공유 전에 데이터셋을 규정 준수 형태로 가공하는 파이프라인이 필요 → 이 지점을 Anonymizer가 푼다

## **(2) 솔루션**

주요컨셉: 민감 속성을 제거하거나 변형해서, 각 row를 신원 식별 불가능한 상태로 만든다.

1. **Data Removal (데이터 제거)**
    - 민감 컬럼 자체를 데이터셋에서 삭제
    - 구현 방법: SELECT에서 컬럼을 아예 안 읽거나(컬럼 수 적을 때), drop 함수로 제거(컬럼 수 많을 때)
    - 실무 선호: 셋 중 구현이 제일 쉬움. 컬럼이 분석에 안 쓰이면 이게 기본값
2. **Data Perturbation (데이터 교란)**
    - 원본 값에 노이즈를 섞어서 의미를 바꿈
    - 예: IP `123.456.789.012` → `1823.456.7809.012` (임의 위치에 숫자 추가)
    - 실무 선호: 컬럼은 유지하되 값의 신뢰성만 깨고 싶을 때. mapping 함수나 컬럼 변환으로 구현
3. **Synthetic Data Replacement (합성 데이터 대체)**
    - 원본 값을 합성 데이터 생성기가 만든 값으로 치환. 컬럼 타입은 유지되지만 값은 완전히 다른 것으로 바뀜
    - 예: country 컬럼 "Portugal" → "Croatia"
    - 이상적으로는 데이터 사이언스팀과 협업해서 ML 모델 기반 생성기를 만듦. 간소화 버전으로는 컬럼별 랜덤 값 생성 함수를 직접 짜는 것도 가능(예: Faker 라이브러리)

실무 선호: **Data Removal이 기본값**. 컬럼이 꼭 필요 없으면 제거가 제일 확실하고 구현도 쉬움. 컬럼 존재는 유지해야 하는데(스키마 검증 등) 값만 감추면 되면 Perturbation, 통계적 분포까지 자연스럽게 보존해야 하면 Synthetic Data 순으로 고려.

## **(3) 결과**

결론: 강한 보안 보장을 얻는 대신, 데이터 활용도(usability)를 크게 희생한다.

- **정보 손실(Information Loss)**: 배경 — 제거/변형된 컬럼은 원본 정보를 담고 있지 않음. 문제 — 데이터 분석가·데이터 사이언티스트가 이 컬럼에 의존하던 작업이 다 부정확해짐(잘못된 예측 모델, 잘못된 인사이트). 대응 — 이 트레이드오프가 너무 크면, 보안은 조금 양보하고 활용도를 살리는 Pseudo-Anonymizer 패턴(#47) 고려



## **(4)예시**

> (엔지니어 독백)
> 마케팅팀에 넘길 유저 데이터셋을 익명화한다. birthday는 완전히 필요 없다고 판단해서 제거하고, email은 형식 검증 로직(다른 서비스에서 `@` 포함 여부 체크하는 로직) 때문에 컬럼 자체는 남겨야 해서 가짜 값으로 치환한다.

원본 테이블(users):
```
+-------+-------------------------+----------+---------+
|user_id|                    email|  birthday|  country|
+-------+-------------------------+----------+---------+
|      1|  jane.doe@dedp.com       |1990-04-12|   Poland|
|      2|  john.smith@dedp.com     |1985-11-02|   France|
+-------+-------------------------+----------+---------+
```


**케이스 1. 컬럼 제거 — SELECT 방식**
- 읽어올 컬럼을 직접 나열해서 birthday를 목록에서 제외
```python
users.select('user_id', 'email', 'country')
```
- 한 줄 요약: birthday를 빼고(Select) Dataframe선언후 Write해서 결과 저장 
- 실행 순서: `select()`에 나열된 컬럼만 읽어서 새 DataFrame 생성 → birthday는 목록에 없으니 결과에 안 나타남
- ★ 핵심: 나열 방식이라 "지금 이 테이블에 어떤 컬럼이 남아있는지"가 코드만 봐도 바로 보임



**케이스 2. 컬럼 제거 — drop 방식**
- 제거할 컬럼만 지정
```python
users.drop('birthday')
```

- 한 줄 요약: birthday 컬럼 하나만 지정해서 제거하고 나머지는 자동으로 유지
- ★ 핵심: 제거 대상만 지정하면 되니, 컬럼이 새로 추가돼도 코드 수정 없이 자동으로 결과에 포함됨

- (엔지니어 독백) 실무에서 거의 항상 이 방식을 쓴다. 컬럼 많은 테이블에서 지울 컬럼 한두 개만 지정하는 게 코드도 짧고, 스키마가 나중에 바뀌어도 안전하다. 처음 익명화 코드 짤 때는 무조건 `drop` 먼저 고려하고, 컬럼이 정말 적은 특수 케이스에서만 `select`를 쓴다.


**케이스 3. 컬럼 치환 — Faker로 email 대체**

- email 값을 실제와 무관한 가짜 값으로 치환(Synthetic Data Replacement)
```python
@pandas_udf(StringType())
def replace_email(emails: pandas.Series) -> pandas.Series:
    faker_generator = Faker()
    return emails.apply(lambda email: faker_generator.email())

users.drop('birthday').withColumn('email', replace_email(users.email))
```

- 한 줄 요약: birthday는 drop으로 제거, email은 Faker가 생성한 가짜 이메일로 덮어씀

실행 순서:
1. `drop('birthday')` 호출 — birthday 컬럼이 빠진 실행 계획(logical plan)을 정의. 이 시점엔 아직 실제 데이터 처리 안 일어남(Spark의 lazy evaluation)
2. `withColumn('email', replace_email(users.email))` 호출 — email 컬럼을 `replace_email` 결과로 교체하는 단계를 실행 계획에 추가. 역시 아직 실행 안 됨
3. `result.write.parquet(...)` 호출 — 여기서 비로소 실행 계획 전체가 트리거됨 (action)
4. Spark가 파티션 단위로 데이터를 나눠서, 각 파티션의 email 컬럼을 pandas.Series로 묶어 `replace_email` 함수에 전달
5. 함수 내부에서 `.apply(lambda email: faker_generator.email())` 실행 — Series의 각 값을 Faker가 생성한 가짜 이메일로 교체
6. 교체된 결과가 email 컬럼에 반영되고, birthday 없는 최종 row들이 S3에 parquet 파일로 write됨
★ 핵심: 
- 1~2번은 "계획만 세우는" 단계이고, 실제 데이터가 움직이는 건, 
- 3번 `write`가 호출되는 순간부터라는 점. `drop`이나 `withColumn`만 써놓고 write를 안 하면 아무 파일도 안 만들어지고, 익명화도 실제로는 일어나지 않음


결과 테이블:
```
+-------+---------------------------+---------+
|user_id|                      email|  country|
+-------+---------------------------+---------+
|      1| kperez@example-fake.net   |   Poland|
|      2| tlopez@example-fake.org   |   France|
+-------+---------------------------+---------+
```

- (엔지니어 독백) `pandas_udf`를 쓰는 이유는 순전히 성능이다. 일반 Python UDF는 JVM과 Python 프로세스 사이에서 row 하나씩 데이터를 주고받다 보니 오버헤드가 크다. `pandas_udf`는 파티션 단위로 데이터를 배치로 한 번에 넘겨받아서 pandas 벡터 연산으로 처리하니까 훨씬 빠르다. 익명화는 보통 테이블 전체를 훑어야 하는 작업이라 이 성능 차이가 실무에서 눈에 띄게 느껴졌다. 그리고 Faker가 만든 이메일은 실제 도메인이 존재하지 않는 값이라, 나중에 분석팀이 이 값으로 실제 메일을 보내는 실수도 방지된다.




**(5) 최신트렌드**

1. **Databricks Unity Catalog column mask**:  
	1. 테이블 접근 권한 레이어에서 컬럼 마스킹/제거를 SQL 함수로 선언적으로 걸 수 있는 기능. 이전 한계 — 예시 코드처럼 파이프라인 안에 익명화 로직을 매번 하드코딩해야 했음. 
	2. 왜 요즘 쓰나 — 익명화 규칙을 데이터가 아니라 카탈로그 레벨 정책으로 관리해서, 쿼리하는 사람 권한에 따라 자동으로 마스킹/제거된 값이 노출됨
2. **AWS Glue DataBrew / Macie**:  — 
3. PII 자동 탐지 + 익명화 룰 적용 도구. 
4. 이전 한계 — 어떤 컬럼이 PII인지 사람이 직접 스캔해서 찾아야 했음. 
5. 왜 요즘 쓰나 — 자동 탐지로 놓치는 PII 컬럼을 줄여줌. 컴플라이언스 감사 대응에서 체감 큼




# **패턴 #47: Pseudo-Anonymizer (의사 익명화기) — 최종 정리본**

한 줄 정의: PII를 완전히 제거하는 대신, 비즈니스 의미를 일부 남기면서 원본 식별을 어렵게 변형하는 패턴.

## (1) 문제상황

핵심 고통: Anonymizer로 PII를 지웠더니, 이번엔 데이터가 쓸모없어져서 분석팀이 못 쓴다.

> (엔지니어 독백)
> 
> 지난번 Anonymizer 패턴 적용해서 외부 분석 업체에 데이터셋 넘겼다. birthday는 제거하고 email은 Faker로 치환해서, 규정은 지켰다고 생각했다.
> 
> 근데 일주일 뒤 분석팀에서 컴플레인이 왔다. "국가별 매출 비교하고 싶은데 country 컬럼이 아예 없다", "SSN 뒷자리로 지역 그룹핑하던 로직이 안 돌아간다" — 대부분의 비즈니스 쿼리를 못 답한다는 거다.
> 
> 컬럼을 통째로 지우거나 완전히 랜덤한 값으로 바꾸니 보안은 확실한데, 데이터의 비즈니스 의미까지 같이 날아간 거다. PII는 숨기되 통계적 유용성은 남겨야 하는 상황 — 이걸 Pseudo-Anonymizer가 푼다.

**핵심 차이 한 줄: Anonymizer는 원본을 복원 불가능하게 완전히 지우고, Pseudo-Anonymizer는 원본과의 연관성을 일부 남긴 채 변형한다.**
- Anonymizer
	- 재식별 불가 
- Pseudo-Anonymizer
	- 다른 데이터셋과 재식별 가능 


## **(2) 솔루션**

주요컨셉: 원본 값을 완전히 제거하지 않고, 실제 값과 어느 정도 연관성을 유지한 채 다른 값으로 치환한다.

1. **Data Masking (데이터 마스킹)**
    - 민감 값 일부만 의미없는 문자 또는 좀 더 그럴듯한 대체값으로 가림
    - 예: SSN `999-55-1040` → `XXX-XX-1040`
    - 실무 선호: 구현이 제일 간단해서, 자릿수 일부만 보존해도 되는 경우(로그 확인용 등) 기본으로 씀. 단, 여러 유저가 같은 마스킹 결과로 겹칠 수 있음(뒤 결과 참고)
2. **Data Tokenization (데이터 토큰화)**
    - 원본 값을 가짜 값으로 치환하고, 원본↔가짜 매핑을 별도 token vault에 저장
    - 매핑 정보가 있어서 필요시 원복 가능. 대신 vault 접근 권한이 뚫리면 전체 역산 가능해서, vault 보안이 핵심
    - 실무 선호: 나중에 원본 복원이 필요한 워크플로우(고객 지원팀이 실제 값 조회해야 하는 경우 등)에 적합
3. **Hashing (해싱)**
    - SHA-256 등으로 완전히, 비가역적으로 치환
    - 예: `contact@waitingforcode.com` → SHA-256 해싱 후 Base64 인코딩된 문자열
    - 실무 선호: 원본 복원이 아예 필요 없고, 동일 입력이 항상 동일 해시로 나오는 특성(join key로 활용 가능)이 필요할 때
4. **Encryption (암호화)**
    - 컬럼/row 단위로 암호화 키를 적용. 키를 가진 사람만 원복 가능
    - 실무 선호: 접근 권한을 키 단위로 세밀하게 통제하고 싶을 때. Encryptor 패턴(#43)과 메커니즘이 겹침

실무에서는 컬럼 성격에 따라 방식을 섞어 씀. 예: 국가는 지역 단위로 일반화(generalization), SSN은 마스킹, salary는 구간화(binning).


## **(3) 결과**

결론: 비즈니스 유용성은 얻지만, Anonymizer보다 약한 보안 보장을 감수해야 한다.

1. **거짓 안전감(False sense of security)**
    - 배경: 하나의 테이블만 보면 개인 특정이 안 되어 보임
    - 문제: 다른 pseudo-anonymized 테이블과 조인하면 재식별될 수 있음. 책 예시로는, food preference 테이블은 완벽히 가려 보여도 registration 테이블의 country/role 컬럼과 결합하면 "산마리노 거주, 이 프레임워크 발명자" 식으로 특정 인물이 좁혀짐
    - 대응: 공유 전 다른 데이터셋과 조인했을 때 재식별 가능성까지 검토해야 함. 완전 익명화(Anonymizer)였다면 country/role 자체가 "Europe", "Software engineer"처럼 일반화되어서 이런 결합 재식별이 원천적으로 안 됨
2. **정보 손실**
    - 배경: 마스킹은 일부 자릿수를 버림
    - 문제: 서로 다른 두 SSN이 같은 마스킹 결과로 겹칠 수 있음(`999-55-1040`, `999-13-1040` 모두 `XXX-XX-1040`). 여기에 더해 generalization처럼 숫자→구간 문자열 변환은 데이터 타입 자체가 바뀌는 손실도 발생
    - 대응: 정확한 값 구분이 중요하면 마스킹 대신 타입 손실 없는 tokenization 고려

> (엔지니어 독백)
> 
> Pseudo-Anonymizer 쓸 때 제일 많이 하는 실수가 "이 컬럼 하나만 보면 안전하니까 됐다"는 판단이다. 실제 공유 시나리오에서는 상대방이 이미 다른 데이터셋을 갖고 있을 수 있다는 전제하에 재식별 리스크를 봐야 한다. 완전 익명화가 필요한지, 의사 익명화로 충분한지는 "이 데이터가 다른 데이터와 결합될 가능성이 있는가"로 판단하는 게 제일 정확했다.


## **(4) 예시**

> (엔지니어 독백)
> 
> 유저 테이블에 country, ssn, salary가 있다. 분석팀이 국가별·소득구간별 통계는 봐야 하니 완전 삭제는 안 되고, pseudo-anonymize로 처리한다.

원본 테이블:
```
+-------+-------+--------------+------+
|user_id|country|           ssn|salary|
+-------+-------+--------------+------+
|      1| Poland|0940-0000-1000| 50000|
|      2| France|0469-0930-1000| 60000|
|      3|the USA|1230-0000-3940| 80000|
|      4|  Spain|8502-1095-9303| 52000|
+-------+-------+--------------+------+
```


**케이스 1. country/ssn 치환 — mapInPandas 사용**
```python
def pseudo_anonymize_users(input_pandas: pandas.DataFrame) -> pandas.DataFrame:
    def pseudo_anonymize_country(country: str) -> str:
        countries_area_mapping = {
            'Poland': 'eu', 'France': 'eu', 'Spain': 'eu', 'the USA': 'na'
        }
        return countries_area_mapping[country]

    def pseudo_anonymize_ssn(ssn: str) -> str:
        return f'{ssn[0]}***-{ssn[5]}***-{ssn[10]}***'

    for rows in input_pandas:
        rows['country'] = rows['country'].apply(lambda c: pseudo_anonymize_country(c))
        rows['ssn'] = rows['ssn'].apply(lambda ssn: pseudo_anonymize_ssn(ssn))
        yield rows
```

- 한 줄 요약: country는 국가명→지역코드(generalization)로, ssn은 일부 자릿수만 남기고 마스킹
- 실행 순서:
    1. `users.mapInPandas(pseudo_anonymize_users, schema)` 호출 시점엔 아직 실행 안 됨(lazy)
    2. 이후 write 같은 action이 트리거되면, Spark가 파티션 단위 pandas.DataFrame을 `pseudo_anonymize_users` 함수에 넘김
    3. 함수 내부에서 `rows['country'].apply(...)`로 country 값을 지역코드로 매핑
    4. `rows['ssn'].apply(...)`로 ssn 값을 마스킹된 문자열로 변환
    5. 변환된 DataFrame을 `yield`로 Spark에 돌려줌
- ★ 핵심: `rows['country'].apply(lambda c: pseudo_anonymize_country(c))` — 국가명을 지역 단위로 일반화하는 부분. 개별 국가는 특정 못 하게 하면서도 "유럽 매출 vs 북미 매출" 분석은 여전히 가능하게 함



**케이스 2. salary 치환 — 컬럼 기반 구간화**
- salary는 타입이 int→string(구간)으로 바뀌어서 mapInPandas 밖, 컬럼 기반 변환으로 별도 처리
```python
pseud_anonymized_users = (
    users.mapInPandas(pseudo_anonymize_users, users.schema)
    .withColumn('salary', functions.expr('''
        CASE WHEN salary BETWEEN 0 AND 50000 THEN "0-50000"
             WHEN salary BETWEEN 50000 AND 60000 THEN "50000-60000"
             ELSE "60000+" END'''))
)
pseud_anonymized_users.write.parquet('s3://shared-bucket/users_pseudo_anonymized/')
```
- ★ 핵심: salary를 정확한 숫자 대신 구간 문자열로 바꿔서, 정확한 소득은 숨기되 소득 구간별 통계는 유지
- `.write.parquet(...)` — 이 write 시점에 비로소 위 mapInPandas와 withColumn 계획이 전체 실행됨

결과:
```
+-------+-------+--------------+-----------+
|user_id|country|           ssn|     salary|
+-------+-------+--------------+-----------+
|      1|     eu|0***-0***-1***|    0-50000|
|      2|     eu|0***-0***-1***|50000-60000|
|      3|     na|1***-0***-3***|     60000+|
|      4|     eu|8***-1***-9***|50000-60000|
+-------+-------+--------------+-----------+
```

> (엔지니어 독백)
> mapInPandas는 row 여러 개를 pandas DataFrame 단위로 받아 처리하니까, 함수 안에서 딕셔너리 매핑이나 문자열 슬라이싱 같은 파이썬 로직을 자유롭게 쓸 수 있어서 편하다. 다만 타입이 바뀌는 컬럼(salary처럼 int→string)은 mapInPandas 스키마 정의가 복잡해지니까, 컬럼 단위 `withColumn`으로 분리해서 처리하는 게 유지보수하기 편했다.

## **(5) 최신트렌드**

**1. Databricks Unity Catalog — Column Mask / Row Filter**
- 정체: 테이블 접근 권한 레이어에서 컬럼 마스킹 규칙을 SQL 함수로 선언적으로 걸 수 있는 기능
- 이전 한계: 예시 코드처럼 마스킹/토큰화 로직을 파이프라인 코드 안에 매번 하드코딩해야 했음. 파이프라인마다 로직이 흩어져서 규칙 일관성 관리가 어려움
- 왜 요즘 쓰나: 마스킹 규칙을 데이터가 아니라 카탈로그 레벨 정책으로 관리 → 쿼리하는 사람의 권한에 따라 자동으로 마스킹된 값이 보임. 파이프라인 코드를 안 건드리고 규칙만 바꿔도 전체 반영됨

**2. AWS Glue DataBrew / Amazon Macie**
- 정체: PII 자동 탐지 + 마스킹 룰 적용 도구
- 이전 한계: 어떤 컬럼이 PII인지 사람이 직접 스캔해서 찾아야 했음. 신규 컬럼 추가될 때마다 놓치는 경우 발생
- 왜 요즘 쓰나: 자동 탐지로 PII 컬럼 누락 리스크를 줄여줌. 컴플라이언스 감사 대응할 때 "탐지 이력" 자체가 증빙 자료가 됨

**3. Tokenization 전용 SaaS (Skyflow, Protegrity 등)**
- 정체: token vault를 직접 운영 안 하고 외부 관리형 서비스로 위임하는 방식
- 이전 한계: vault를 직접 구축·보안 관리해야 해서 운영 부담이 컸음 (접근 통제, 암호화, 백업까지 전부 자체 책임)
- 왜 요즘 쓰나: vault 보안 책임을 전문 업체에 넘기고, 우리는 API 호출만으로 토큰화/복원 처리 가능

**실무 선호 정리**
- 이미 Databricks/Snowflake 같은 카탈로그 기반 플랫폼을 쓰는 조직 → 1번(Unity Catalog)이 가장 자연스러운 선택. 별도 도구 도입 없이 기존 인프라에서 해결
- PII 탐지 자체가 어려운 대규모 미정형 데이터(로그, 텍스트) → 2번(DataBrew/Macie) 우선 검토
- 원본 복원이 자주 필요하고 vault 운영 리소스가 부족한 소규모 팀 → 3번(Tokenization SaaS)

> (엔지니어 독백)
> 세 가지가 서로 대체재라기보다 계층이 다르다. Unity Catalog는 "이미 있는 테이블을 어떻게 보여줄지"를 다루고, DataBrew/Macie는 "애초에 뭐가 PII인지 어떻게 찾을지"를 다루고, Tokenization SaaS는 "값을 어떻게 바꾸고 복원할지"를 다룬다. 실무에서는 이 셋을 조합해서 쓰는 경우가 많다 — Macie로 PII 탐지하고, 탐지된 컬럼에 Unity Catalog 마스킹 정책을 걸고, 복원 필요한 값만 외부 tokenization 서비스로 따로 처리하는 식이다.




# **패턴 #48: Secrets Pointer (비밀 포인터)

한 줄 정의: credential(login/password/API key)을 코드나 설정 파일에 직접 저장하지 않고, secrets manager에 저장된 값을 가리키는 참조(이름)만 코드에 남기는 패턴.

## **(1) 문제상황**

핵심 고통: 코드에 credential을 직접 박아뒀다가 실수로 유출되면, 돈이 샌다.

> (엔지니어 독백)
> 
> 실시간 방문 이벤트 처리 파이프라인에 geolocation enrichment를 붙이는 작업을 맡았다. 외부 API 업체가 제공하는 서비스인데, 인증 방식이 login/password 조합뿐이다.
> 
> 그런데 예전에 다른 API용 login/password를 실수로 코드 저장소에 그대로 올렸다가 유출된 적이 있다. 그 API가 요청 건당 과금되는 구조라서, 유출된 credential로 누군가 계속 호출을 날려서 비용이 확 튀었다.
> 
> 이번 API도 똑같이 코드에 login/password를 하드코딩하면 같은 사고가 반복될 게 뻔하다. credential을 아예 코드 어디에도 저장하지 않는 방법이 필요하다 — 이게 Secrets Pointer가 푸는 지점이다.

## **(2) 솔루션**

주요컨셉: consumer가 credential 값을 직접 들고 있지 않고, secrets manager(예: AWS Secrets Manager, Google Cloud Secret Manager)에 저장된 값의 이름(참조)만 코드에 남긴다.

1. **Secret 등록**
    - secrets manager에 login/password/API key 같은 민감 값을 저장. 각 값에 이름(예: `prod/geo-api/user`)을 붙여서 관리
    - 등록 주체는 사람 관리자 또는 IaC(Terraform 등). IaC가 DB 생성 시 랜덤 값을 만들어 자동 등록하는 방식이 흔함
2. **참조 기반 조회**
    - consumer(잡, 애플리케이션)는 실제 값이 아니라 이 이름만 코드에 갖고 있음
    - 런타임에 이 이름으로 secrets manager에 조회 요청을 보내서, 그 순간 실제 값을 받아옴
    - 통신 비용을 아끼려면 조회한 값을 로컬에 잠깐 캐시할 수 있음(단, 이건 (3)결과에서 다룰 트레이드오프를 만듦)
3. **이중 보안 구조**
    - 1단계: consumer가 secrets manager 자체에 접근 권한이 없으면, 참조 이름만으로는 값을 못 받아옴 (Fine-Grained Accessor for Resources 패턴과 결합 가능)
    - 2단계: 값을 받아도 그 credential이 실제로 유효하지 않으면, 최종 API/DB 접근 자체가 막힘
4. **다중 환경 대응**
    - dev/staging/prod 각각 같은 이름 구조(`dev/geo-api/user`, `prod/geo-api/user`)로 secret을 만들고, 실제 값만 환경별로 다르게 IaC가 자동 생성
    - 코드는 환경 무관하게 참조 이름만 사용하므로, 환경별 설정 파일을 따로 관리할 필요가 없어짐

## **(3) 결과**

결론: credential이 코드에서 사라지는 대신, 캐시 갱신·로그 유출·최초 생성이라는 새 작업을 감수해야 한다.

1. **캐시 무효화 — 특히 스트리밍 잡**
    - 배경: 매번 secrets manager에 조회하면 비용/지연이 쌓이니, consumer가 조회한 값을 로컬(프로세스 메모리)에 캐시해둠
    - 문제: 캐시된 값이 최신인지 알 수 없어서, secret이 로테이션되면 연결 오류가 발생. 배치 잡은 매 실행마다 새 프로세스로 뜨니 이 문제가 거의 안 보이지만, 며칠씩 계속 떠있는 스트리밍 잡은 로테이션 이후 계속 옛날 값을 씀
    - 대응: 가장 단순한 방법은 잡을 그냥 실패시키고, 재시작 때 새 값을 다시 로드하게 두는 것. 이때 재시도로 데이터가 중복 처리되지 않도록 idempotency 패턴을 같이 적용해야 함. 비동기 갱신 프로세스도 가능하지만, 갱신 시점과 실제 쓰기 시점이 어긋나면 쓰기 오류가 날 수 있음
    - (엔지니어 독백) 이건 실제로 겪어본 문제다. 배치 잡은 캐시 문제가 거의 안 보이는데, 스트리밍 잡에서 어느 날 갑자기 다 실패하기 시작한다. 처음엔 API 쪽 장애인 줄 알고 한참 헤맸는데, 알고 보니 보안팀이 정기 로테이션 정책으로 password를 바꾼 거였다. "잡이 실패하면 재시작 때 새로 로드한다"는 전략 자체는 나쁘지 않지만, 로테이션 이벤트를 사전에 알림받는 장치가 없으면 원인 파악에 시간을 다 쓴다.
2. **로그 유출**
    - 배경: secrets manager 자체는 안전한 저장소
    - 문제: consumer 코드가 조회한 credential 값을 실수로 로그에 남기면, secrets manager를 우회해서 그대로 유출됨. 예를 들어 `logger.error(f"연결 실패: {e}")`처럼 예외 객체를 통째로 로그에 넘기면, 그 안에 password 문자열이 섞여있을 수 있음
    - 대응: 커스텀 예외 메시지만 로그에 남기고 원본 예외 객체는 로그에 안 넘김. 로그 마스킹 필터, CloudWatch Logs 접근 권한 최소화도 방어선으로 추가
    - (엔지니어 독백) 실무에서 가장 흔한 실수다. Secrets Manager 권한은 다들 꼼꼼히 좁히면서, 정작 값을 다루는 코드 안의 로그 한 줄은 놓친다. Secrets Pointer 도입했다고 credential 보안이 끝난 게 아니다. 그 값을 다루는 코드 전체를 로그 관점에서 별도로 점검해야 한다.
3. **secret은 결국 어딘가에서 최초 생성돼야 함**
    - 배경: consumer는 참조만 쓰지만, secret 값 자체는 누군가 처음에 채워 넣어야 함
    - 문제: 최초 생성 주체(사람 관리자 또는 IaC)가 없으면 패턴 자체가 성립 안 함. 사람이 콘솔에서 값을 입력하는 방식이면 그 순간 슬랙이나 화면 공유로 노출될 위험이 생김
    - 대응: DB 생성과 secret 등록을 같은 IaC 스택 안에서 묶어서, 사람이 값을 보는 구간을 아예 없앰

## **(4) 예시**

**기술1: AWS Secrets Manager — PostgreSQL 연결 (책 Example 7-27)**

- PostgreSQL 테이블을 Spark로 읽어서 JSON으로 떨구는 잡. user/password를 코드에 직접 안 넣고 참조로 처리

```python
secretsmanager_client = boto3.client('secretsmanager')
db_user = secretsmanager_client.get_secret_value(SecretId='user')['SecretString']
db_password = secretsmanager_client.get_secret_value(SecretId='pwd')['SecretString']

spark_session.read.option('driver', 'org.postgresql.Driver').jdbc(
    url='jdbc:postgresql:dedp', table='dedp.devices',
    properties={'user': db_user, 'password': db_password})
```
- 한 줄 요약: boto3로 Secrets Manager에서 user/password 값을 조회한 뒤, 그 값으로 PostgreSQL JDBC 연결을 구성
- 실행 순서:
    1. `boto3.client('secretsmanager')`로 클라이언트 생성
    2. `get_secret_value(SecretId='user')` 호출 — 'user'라는 이름으로 Secrets Manager에 조회 요청
    3. 응답 딕셔너리에서 `['SecretString']`으로 실제 값 추출 → `db_user`에 저장
    4. 동일하게 'pwd' 이름으로 조회해서 `db_password`에 저장
    5. `spark_session.read.jdbc(...)` 호출 시 `properties`에 이 값들을 넘겨서 실제 연결 수행
- ★ 핵심: `SecretId='user'` — 코드에 있는 건 실제 login 값이 아니라 이름(참조). 이 코드가 저장소 전체 공개돼도 이 이름만으로는 아무것도 못 함
- (엔지니어 독백) 이 코드가 다중 환경에서 특히 빛을 본다. dev/staging/prod 모두 `SecretId='user'`라는 같은 이름을 쓰고, 실제 값은 Terraform이 환경별로 다르게 채워준다. 예전엔 환경별 config 파일에 실제 값을 넣어뒀는데, 배포 스크립트 실수로 prod 값이 staging 브랜치에 커밋된 적이 있었다. 이 구조로 바꾸고 나서는 코드에 값 자체가 없으니 그런 실수가 원천 차단됐다.

**기술2: AWS 실행 role 권한 설정 (책에는 없지만 실무 필수 단계)**

- 위 코드가 동작하려면, Spark 잡을 실행하는 EC2/EMR/ECS의 IAM role에 `secretsmanager:GetSecretValue` 권한을 이 secret ARN 하나로 한정해서 부여해야 함

```json
{
  "Effect": "Allow",
  "Action": "secretsmanager:GetSecretValue",
  "Resource": "arn:aws:secretsmanager:ap-northeast-2:123456789012:secret:prod/geo-api/credentials-*"
}
```

- 한 줄 요약: 이 IAM role이 지정된 secret ARN 하나에 대해서만 조회 권한을 갖도록 제한
- ★ 핵심: `Resource`를 `*`가 아니라 구체적인 ARN으로 한정 — 이게 (2)솔루션에서 말한 "1단계 보안"을 실제로 구현하는 부분
- (엔지니어 독백) 여기서 제일 많이 보는 실수가 `Resource: "*"`로 열어두는 거다. "일단 되게만 하자"고 열어두면, 나중에 다른 잡이 이 role을 재사용할 때 전혀 상관없는 secret까지 다 읽어버린다. 처음부터 ARN을 딱 지정하는 게 5분 더 걸려도 나중 사고를 막는다.

**분류 요약**

|기술|역할|담당 계층|
|---|---|---|
|boto3 + Secrets Manager API|런타임에 참조 이름으로 실제 값 조회|애플리케이션(Spark 잡) 코드|
|IAM policy|어떤 role이 어떤 secret을 조회할 수 있는지 통제|클라우드 접근 통제 계층|

## **(5) 최신트렌드**

**1. AWS Secrets Manager 자동 로테이션 (RDS 연동)**
- 정체: Lambda 함수를 트리거로 걸어서 일정 주기마다 DB password를 자동으로 바꾸고, secret에 새 값을 갱신하는 기능. `createSecret → setSecret → testSecret → finishSecret` 4단계로 진행되며, 검증 통과 후에만 새 값이 `AWSCURRENT`로 승격됨
- 이전 한계: password 로테이션을 사람이 수동으로 하거나 별도 스크립트를 cron으로 돌려야 했음. 로테이션 도중 애플리케이션이 옛날 값을 쓰다 연결 실패하는 타이밍 이슈도 직접 관리해야 했음
- 왜 요즘 쓰나: RDS 같은 관리형 DB와 결합하면 로테이션 자체를 AWS가 대행하고, 이전 값과 새 값을 동시에 유효하게 유지하는 구간까지 지원해서 (3)에서 본 캐시 무효화 문제를 서비스 차원에서 어느 정도 흡수해줌

**2. External Secrets Operator (Kubernetes 환경)**
- 정체: 클라우드 secrets manager 값을 Kubernetes Secret 오브젝트로 자동 동기화해주는 오퍼레이터
- 이전 한계: 각 파드 안의 애플리케이션 코드가 매번 boto3로 직접 Secrets Manager를 호출하게 짜야 했음. 코드마다 조회 로직이 중복되고, IAM 권한도 파드 단위로 세밀하게 관리해야 했음
- 왜 요즘 쓰나: 애플리케이션 코드는 그냥 일반 K8s Secret을 마운트해서 쓰면 되고, 실제 조회·갱신은 오퍼레이터가 백그라운드에서 처리. 코드가 AWS SDK 의존성을 안 가져도 됨


**실무 선호 정리**
- RDS/Aurora 같은 AWS 관리형 DB를 쓰는 경우 → 1번(자동 로테이션)이 기본 선택. 별도 구현 없이 로테이션 문제를 서비스가 흡수
- Spark를 Kubernetes(EKS 등) 위에서 운영하는 팀 → 2번(External Secrets Operator)이 코드 복잡도를 크게 줄여줌
- 둘은 배타적이지 않고 같이 씀: RDS는 1번으로 로테이션 자동화, 그 결과를 K8s 파드에 전달하는 건 2번이 담당

> (엔지니어 독백)
> 처음엔 "자동 로테이션 켜놨으니 캐시 문제도 자동으로 해결되겠지" 하고 착각했었다. 실제론 아니다. Secrets Manager 쪽 값이 바뀌는 것과, 잡이 그 새 값을 가져다 쓰는 것은 완전히 별개 이벤트다. 둘을 연결하는 다리가 없으면, 로테이션은 잘 되는데 잡은 여전히 옛날 값 쓰다 실패한다. 그래서 자동 로테이션 켜는 것과, 잡 쪽에 TTL 기반 재조회 로직 넣는 것, 이 둘을 항상 세트로 묶어서 생각해야 한다.




# **패턴 #49: Secretless Connector (비밀 없는 커넥터) — 최종 정리본**

한 줄 정의: credential(login/password/API key) 자체를 아예 관리하지 않고, cloud identity(IAM role, service account, certificate)를 통해 리소스 접근을 인가받는 패턴.

## **(1) 문제상황**

핵심 고통: Secrets Pointer로 credential을 감췄지만, 여전히 "관리해야 할 credential"이 남아있다.

> (엔지니어 독백)
> 
> 작은 팀 하나가 새로운 데이터 처리 서비스를 붙이겠다고 찾아왔다. 인터넷에서 찾은 예제 코드가 죄다 API key 인증 방식을 쓰는데, 이 팀은 인원이 적어서 API key 발급, 로테이션, Secrets Manager 등록, 권한 관리까지 이 전체 사이클을 감당할 여력이 없다고 한다.
> 
> Secrets Pointer 패턴을 쓰면 코드에 값이 안 남는 건 맞다. 근데 그 값 자체는 여전히 어딘가에 존재하고, 누군가 로테이션·최초 발급·폐기까지 다 챙겨야 한다. 이 팀이 정말 원하는 건 "코드에 값을 숨기는 것"이 아니라 "애초에 관리할 credential 자체가 없는 것"이다. 이게 Secretless Connector가 푸는 지점이다.

## **(2) 솔루션**

주요컨셉: consumer(잡, 애플리케이션)에게 cloud identity(IAM role, service account)를 부여해서, credential 값 없이 IAM/인증서 검증만으로 리소스 접근을 허용한다.

1. **IAM 기반 접근**
    - 관리자가 사람이 아니라 "애플리케이션 사용자"(잡, 서비스)에게 IAM role/service account를 할당하고, 이 role에 읽기/쓰기 등 권한을 정책(policy)으로 부여
    - 동작 흐름:
        1. 애플리케이션(잡)이 클라우드 서비스(예: object store)에 요청을 보냄
        2. 서비스가 직접 응답하지 않고, IAM 서비스에 "이 요청자가 권한이 있는지" 확인 요청
        3. IAM이 권한 범위(scope)를 응답
        4. 서비스가 권한 있으면 요청 처리, 없으면 에러 반환
    - 애플리케이션은 login/password를 아예 안 들고 있고, "나는 이 role이다"라는 신원(identity)만으로 인증됨
2. **인증서 기반 접근(Certificate-based authentication)**
    - IAM 대신 CA(certificate authority)가 연결 시점에 인증서를 검증
    - PostgreSQL 같은 DB 연결처럼 클라우드 IAM이 직접 개입 못 하는 구간에서 주로 씀 — password 없이 서버·클라이언트가 서로 인증서를 검증

실무 선호: 클라우드 네이티브 리소스(S3, GCS 등)는 **IAM 기반**이 표준. DB 연결처럼 클라우드 IAM이 직접 개입 못 하는 구간(자체 호스팅 PostgreSQL 등)은 **인증서 기반**을 씀. 어떤 걸 쓸지는 "리소스가 클라우드 관리형이냐 아니냐"로 갈림.

## **(3) 결과**

결론: credential 관리 부담은 없어지지만, "설정 자체"와 "회전(rotation)"이라는 새 작업이 남는다.

1. **Workless라는 착각**
    - 배경: "-less"라는 이름 때문에 아무 설정도 필요 없을 거라 기대하고 시작
    - 문제: 실제로는 엔티티가 credentialless access를 쓰도록 설정하는 작업이 여전히 필요. AWS 기준으로는 assume role 권한을 설정해서, 엔티티가 STS(Security Token Service)로부터 임시 credential을 발급받도록 세팅해야 함
    - (엔지니어 독백) 처음 이 패턴 도입할 때 "credential 관리 안 해도 된다"는 말만 듣고 편해질 줄 알았다. 근데 IAM role 설계, trust policy 작성, 최소 권한 원칙 적용하는 작업량이 만만치 않았다. credential _값_을 관리 안 하는 거지, 접근 _권한 구조_ 관리는 오히려 더 정교하게 해야 한다는 걸 나중에 깨달았다.
2. **인증서 회전(Rotation)**
    - 배경: 인증서 기반 인증에서 정기적으로 인증서를 교체해야 함(보안 모범 사례)
    - 문제: 새 인증서를 배포하는 동안, 아직 마이그레이션 안 한 consumer도 여전히 접근 가능해야 하니까 신/구 인증서를 동시에 지원해야 함. 모든 consumer가 새 인증서로 전환 완료된 걸 확인한 뒤에야 구 인증서를 폐기할 수 있음
    - (엔지니어 독백) IAM role 기반은 STS가 임시 credential을 자동 발급/만료시켜주니까 이 회전 문제가 거의 안 느껴진다. 반면 인증서 방식은 회전 작업을 사람이 직접 스케줄링해야 해서, 가능하면 IAM 방식을 우선 고려하고 인증서는 IAM이 안 되는 경우의 대안으로 남겨두는 게 실무에서 편했다.


## **(4) 예시**

**기술1: 인증서 기반 — Apache Spark → PostgreSQL (책 Example 7-28)**

- password 없이 SSL 인증서로 연결
```python
input_data = spark.read.option('driver', 'org.postgresql.Driver').jdbc(
    url='jdbc:postgresql:dedp', table='dedp.devices',
    properties={'ssl': 'true', 'sslmode': 'verify-full',
                'user': 'dedp_test', 'sslrootcert': 'dataset/certs/ssl-cert-snakeoil.pem'})
```

- 한 줄 요약: `password` 속성 자체가 없고, `sslrootcert`로 지정한 인증서로 서버를 검증해서 연결
- 실행 순서:
    1. `spark.read.jdbc(...)` 호출 시 `properties`에 `ssl`, `sslmode`, `sslrootcert` 전달
    2. Spark 드라이버가 PostgreSQL 서버에 연결 시도하면서 `sslrootcert`에 지정된 CA 인증서를 로드
    3. 서버가 제시하는 인증서가 이 CA로 서명됐는지 검증
    4. `sslmode='verify-full'`이므로 인증서에 적힌 hostname이 실제 접속 hostname과 일치하는지까지 추가 검증
    5. 모든 검증 통과 시에만 연결 성립, password 없이 인증 완료
- ★ 핵심: `sslmode: 'verify-full'` — 가장 엄격한 검증 모드. 이게 없으면 중간자 공격에 취약해질 수 있음
- (엔지니어 독백) `verify-full`이 아니라 `require`나 `verify-ca`로 느슨하게 설정하는 코드도 종종 보는데, 이러면 암호화는 되지만 "내가 접속한 서버가 진짜 그 서버인지" 확인을 안 하는 거라 사실상 반쪽짜리 보안이다. 인증서 기반을 쓸 거면 `verify-full`이 기본값이어야 한다.



**기술2: IAM 기반 — GCP Dataflow → GCS (책 Example 7-29, 7-30)**

- Service Account(GCP에서 "애플리케이션 사용자"를 부르는 이름)를 만들고, 이 계정에 GCS 버킷 읽기 권한만 부여
```python
resource "google_service_account" "visits_job_sa" {
    account_id   = "dedp"
    display_name = "Dataflow SA for processing visits from GCS"
}

resource "google_storage_bucket_iam_binding" "visits_access" {
    bucket  = "visits"
    role    = "roles/storage.objectViewer"
    members = ["serviceAccount:${google_service_account.visits_job_sa.email}"]
}

resource "google_dataflow_job" "visits_aggregator" {
    # ...
    service_account_email = google_service_account.visits_job_sa.email
}
```

- 한 줄 요약: Terraform으로 Service Account를 만들고, 이 계정에 `objectViewer`(읽기 전용) 역할을 GCS 버킷에 바인딩한 뒤, Dataflow 잡이 이 계정 신분으로 실행되도록 지정
- 실행 순서:
    1. `google_service_account` 리소스로 `visits_job_sa`라는 Service Account 생성
    2. `google_storage_bucket_iam_binding`으로 `visits` 버킷에 대해 이 계정에 `objectViewer` 역할 바인딩
    3. `google_dataflow_job`의 `service_account_email`에 이 계정을 지정
    4. Dataflow 잡이 실행되면, 이 Service Account 신분으로 GCS에 접근 요청
    5. GCS가 IAM에 이 신분의 권한을 확인 → `objectViewer` 권한 있으므로 읽기 허용
- ★ 핵심: `service_account_email = google_service_account.visits_job_sa.email` — 잡 코드 어디에도 API key나 password가 없고, 잡 자체가 "이 Service Account다"라는 신분으로 실행되는 게 이 패턴의 본질. Secrets Pointer처럼 런타임에 값을 조회하는 코드조차 필요 없음
- (엔지니어 독백) Secrets Pointer와 비교하면 차이가 명확해진다. Secrets Pointer는 코드에 `SecretId='...'` 참조라도 남아있어서 "조회하는 코드"가 존재하는데, 이 방식은 그 조회 코드 자체가 없다. 잡의 신분(identity)이 곧 인증 수단이라, 코드에서 credential 관련 로직이 완전히 사라진다는 게 제일 큰 차이다.

**분류 요약**

|기술|인증 방식|적합한 리소스|
|---|---|---|
|SSL 인증서 (verify-full)|인증서 기반|클라우드 IAM이 직접 개입 못 하는 DB (자체 호스팅 PostgreSQL 등)|
|GCP Service Account + IAM binding|IAM 기반|클라우드 네이티브 관리형 리소스 (GCS, S3 등)|

**(5) 최신트렌드**

**1. IRSA / Pod Identity (Kubernetes + AWS)**
- 정체: EKS(Kubernetes on AWS)에서 파드 단위로 IAM role을 매핑해주는 기능
- 이전 한계: K8s 파드에서 AWS 리소스 접근하려면 access key를 Secret으로 마운트하거나, 노드 전체에 broad한 IAM role을 붙이는 수밖에 없었음(파드별 세분화 불가)
- 왜 요즘 쓰나: 파드 단위로 최소 권한 IAM role을 붙일 수 있어서, credential 없이도 파드별 세밀한 접근 통제가 가능해짐. Secretless Connector의 IAM 방식을 K8s 환경에 그대로 확장한 형태

**2. Workload Identity Federation (GCP/멀티클라우드)**
- 정체: 외부 IdP(GitHub Actions, AWS 등)의 신원을 GCP IAM과 연동해서, 별도 Service Account key 파일 없이 인증하는 기능
- 이전 한계: 외부 시스템에서 GCP 접근하려면 Service Account key(JSON 파일)를 다운받아 저장해야 했고, 이 key 파일 자체가 또 다른 유출 지점이었음
- 왜 요즘 쓰나: key 파일이라는 실체 있는 credential이 아예 사라지고, 외부 시스템의 신원을 그대로 신뢰하는 구조라 유출 지점이 원천적으로 줄어듦

**실무 선호 정리**
- Spark를 EKS 위에서 운영하는 팀 → 1번(IRSA/Pod Identity)이 기본. 파드 생성 시 IAM role 매핑만 설정하면 됨
- CI/CD 파이프라인(GitHub Actions 등)에서 GCP 리소스 접근 → 2번(Workload Identity Federation)으로 key 파일 관리 자체를 없앰

> (엔지니어 독백)
> 두 트렌드 다 공통점이 있다. "credential 파일을 아예 실체로 안 만든다"는 방향으로 계속 진화하고 있다는 거다. 예전엔 Service Account key JSON을 다운받아서 GitHub Secrets에 등록하는 게 표준이었는데, 이제는 그 key 파일 자체를 안 만들고 신원 연동만으로 인증하는 쪽으로 넘어갔다. Secretless Connector 패턴이 처음 나온 취지(credential 자체를 없애자)가 인프라 레벨에서 점점 더 철저하게 구현되고 있다고 보면 된다.

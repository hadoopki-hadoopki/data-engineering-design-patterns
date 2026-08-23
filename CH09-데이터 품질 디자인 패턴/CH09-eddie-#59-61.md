
### 챕터 목적
- 데이터 품질을 보장해서 **불완전하거나, 일관성 없거나, 부정확한 데이터셋**을 소비자에게 넘기지 않는 것
- 소제목
	- **품질 강제(Quality Enforcement)** 
	- **스키마 일관성(Schema Consistency)**  
	- **품질 관찰(Quality Observation)** 

### 패턴 관계 
| 구역                           | 패턴                               | 하는 일                                                                     |
| ---------------------------- | -------------------------------- | ------------------------------------------------------------------------ |
| 품질 강제 (Quality Enforcement)  | Audit-Write-Audit-Publish (AWAP) | 파이프라인 안에 데이터 검증 단계를 끼워넣어서, 나쁜 품질 데이터가 아예 퍼블리시 안 되게 막음                    |
| 품질 강제 (Quality Enforcement)  | Constraints Enforcer             | 데이터베이스 레벨에서 제약조건(NOT NULL, UNIQUE 등)을 걸어서, producer가 애초에 나쁜 데이터를 못 넣게 막음 |
| 스키마 일관성 (Schema Consistency) | Schema Compatibility Enforcer    | 스키마가 바뀌어도 기존 consumer가 안 깨지게, 변경 전에 호환성을 체크                              |
| 스키마 일관성 (Schema Consistency) | Schema Migrator                  | 스키마를 실제로 마이그레이션할 때, 기존 consumer 안 깨뜨리고 옮기는 방법                            |
| 품질 관찰 (Quality Observation)  | Offline Observer                 | 파이프라인과 분리된 별도 컴포넌트가, 이미 쌓인 데이터를 나중에 검사                                   |
| 품질 관찰 (Quality Observation)  | Online Observer                  | 파이프라인 안에서 실시간으로 데이터를 관찰                                                  |

### 패턴 간 관계 — 왜 이 순서인가
- 1단계: AWAP + Constraints Enforcer
    - "정의해둔 규칙"을 어기는 데이터를 막는 것 (사전 방어)
- 2단계: Schema Compatibility Enforcer + Schema Migrator
    - 스키마라는 특수한 종류의 "규칙"을 별도로 다룸 (스키마는 자주 바뀌니까 전용 패턴 필요)
- 3단계: Offline/Online Observer
    - 1, 2단계가 다 규칙을 "미리 정의"해야 작동하는데, 미처 정의 못 한 새로운 문제를 나중에라도 발견하기 위한 안전망

### 한 줄 요약 (엔지니어 독백)
> 1, 2단계는 "내가 아는 위험"을 막는 거고, 
> 3단계는 "내가 모르는 위험"을 찾는 거다. 
> AWAP 아무리 잘 짜놔도, 애초에 검증 룰 자체를 안 만든 케이스는 못 잡는다.
>  그래서 마지막에 Observer 패턴으로 "미리 정의 안 된 이상 징후"까지 잡는 안전망을 깐다.


<br><br><br><br>
# 패턴 #59: 감사－쓰기－감사－배포 (Audit-Write-Audit-Publish, AWAP)

## (1) 문제상황

**핵심 고통: 파이프라인이 에러 없이 성공했는데, 결과 숫자가 틀렸다.**

- 등장 컴포넌트
    - **daily batch ETL job**: 매일 도는 배치 파이프라인. user visit 로그를 읽어서 고유 방문자 수(unique visitor) 같은 통계를 집계
    - **product team**: 이 통계를 보고 마케팅 캠페인 같은 비즈니스 의사결정을 내리는 조직
- 엔지니어 독백

> 매일 도는 배치 ETL이 user visit 통계를 뽑고 있었다. 그런데 지난 일주일간 고유 방문자 수가 50%나 떨어졌다.
> 
> product team 입장에서는 이게 심각한 문제였다. "방문자가 반토막 났다"고 판단하고, 트래픽을 끌어오려고 새 마케팅 캠페인까지 시작했다.
> 
> 그러다 내가 이 잡의 다른 기능을 손보던 중에 발견했다. 고유 방문자 수 집계 로직 자체가 잘못 짜여 있었다. 실제로 방문자가 줄어든 게 아니라, 집계 코드가 틀려서 숫자가 잘못 나온 거였다.
> 
> product team한테 알렸고, 그쪽은 캠페인을 즉시 중단시켰다. 근데 문제는 여기서 끝이 아니었다. "이런 일이 다시는 없게 해달라"는 요구를 받았다.
> 
> 여기서 진짜 문제가 뭐냐면 — 잡은 에러 없이 성공(SUCCESS)으로 끝났다는 거다. 파이프라인 모니터링 대시보드는 초록불이었다. 데이터가 틀렸다는 걸 알려줄 장치가 파이프라인 안에 아무것도 없었다.

- 개선하니 생긴 지점 (= 이 패턴이 푸는 것)
    - "에러 없이 성공"과 "결과가 맞음"은 다른 얘기다. 파이프라인이 성공/실패 여부만 체크하는 게 아니라, **결과 데이터 자체가 기대치를 만족하는지**를 검증하는 단계가 필요하다.
    - 이 검증을, 코드에 대해 하는 unit test처럼, **데이터에 대해 하는 assertion**으로 파이프라인 안에 끼워넣는 것 — 이게 AWAP의 출발점.
- 한 줄 정리
    - 지금까지 배운 패턴들(Merger, Transactional Writer 등)은 "데이터를 어떻게 쓰느냐"를 다뤘다면, AWAP은 "쓰기 전후로 그 데이터가 맞는 데이터인지 어떻게 확인하느냐"를 다룬다.


## (2) 솔루션 
**주요컨셉: 쓰기 전후로 데이터를 검증하는 4단계(Audit-Write-Audit-Publish)로 파이프라인을 감싼다.**

### 4단계 흐름
- 1단계 Audit (입력 검증)
    - 원본 입력 데이터셋이 최소 기준 만족하는지 가볍게 체크
    - 예: 파일 크기 너무 작지 않은지, 필수 컬럼 있는지
- 2단계 Write (변환 실행)
    - 실제 비즈니스 로직으로 데이터 가공
    - 지금까지 배운 변환/집계 로직이 여기서 실행
- 3단계 Audit (출력 검증)
    - 변환된 결과 데이터셋이 기대치 만족하는지 체크
    - 예: row 수 전날 대비 급감/급증 여부, 특정 컬럼 NULL 비율 임계치 초과 여부
- 4단계 Publish (공개)
    - 3단계 audit 통과한 경우에만 최종 저장소로 내보내서 consumer 접근 가능하게 함

### 흐름도

```
[입력 데이터] ──▶ Audit(입력) ──▶ Write(변환) ──▶ Audit(출력) ──▶ Publish(공개) ──▶ [consumer]
                      │                                │
                  기준 미달 시                       기준 미달 시
                  파이프라인 중단                     파이프라인 중단
                  (Publish 단계까지 도달 안 함)
```

- 핵심: Audit 2번(입력 1번, 출력 1번) 다 통과해야 Publish까지 감
- 둘 중 하나라도 실패 → 파이프라인 즉시 중단 → 나쁜 데이터는 consumer한테 절대 노출 안 됨

### 왜 Audit이 입력/출력 2번 필요한가

- 입력 Audit
    - 목적: producer(데이터 넘겨준 쪽)의 문제를 조기에 잡기
    - 파일 깨짐, 너무 작음 등 → 여기서 바로 중단, 뒤에서 비싼 변환 로직 돌릴 필요조차 없음
- 출력 Audit
    - 목적: 내 변환 로직(Write 단계) 자체의 버그를 잡기
    - 문제 상황에서 겪었던 "고유 방문자 수 집계 로직 버그"는 입력이 아니라 내 코드 문제 → 이건 출력 Audit만 잡을 수 있음
- 하나만 하면 절반만 방어됨
    - 입력만 체크 → 내 코드 버그는 못 잡음(내가 작성한 집계로직)
    - 출력만 체크 → 나쁜 입력 때문에 생긴 문제의 원인 파악 늦어짐

### Unit Test와의 관계 — 대체가 아니라 보완
- Unit test : 코드 한 조각(함수, 메서드)이 특정 입력에 대해 예상한 출력을 내는지, 개발자가 미리 정해둔 가짜(mock) 입력값으로 검증하는 테스트 코드
	- 배포 전에 개발단계에서 출력 수치값 테스트해보는거임
- Unit test
    - 코드 짜는 시점에 "예상하는 입력"으로 미리 검증
    - 정적(static) → 미래의 실제 데이터 패턴은 반영 못 함
- AWAP audit
    - 실제 운영 데이터로, 매 실행마다 실시간 검증
    - 동적(dynamic) → 미래에 데이터 달라져도 계속 체크
- 실무 선호: 둘 다 쓴다
    - Unit test → 1차 방어선 (비즈니스 로직 구현 실수)
    - AWAP audit → 2차 방어선 (실제 데이터의 예측 불가능한 변화)

### Audit 실패 시 선택지 — 항상 파이프라인 중단만은 아님

- 1. 파이프라인 중단 (기본값)
    - audit 실패 시 그냥 멈춤
    - 가장 안전, 가장 엄격
- 2. Data dispatching
	- dispatching
		- 발송, 급파, 배치 (어디 보내는거)
		- 작업, 요청, 또는 제어 흐름을 적절한 처리 주체(프로세서, 핸들러, 함수 등)에게 전달하고 실행을 할당하는 메커니즘
    - 출력 데이터셋 중 일부만 유효하지 않을 때, 유효한 부분은 publish하고 나머지는 별도 저장소로 분리
    - Dead-Letter 패턴 비교 
	    - Dead-Letter 패턴 → 이건 런타임 에러
	    - Data dispatching → 내가 명시적으로 정의한 검증 로직 결과
- 3. Nonblocking audit (그냥 일단 보내, 표시만 남겨)
    - 문제 있어도 일단 publish, "이 데이터셋에 이런 이슈 있다"고 annotation 남김
    - 예: 특정 컬럼 NULL 비율 튀었지만 나머지 정상 → 그 컬럼 안 쓰는 consumer는 그냥 사용, 쓰는 consumer는 annotation 보고 자기 임계치 기준으로 판단

### 실무 선호 — 상황별
- 심각한 문제(스키마 깨짐, 필수 컬럼 NULL 등) → 파이프라인 중단
    - 잘못된 데이터 내보내는 것보다 늦게 내보내는 게 나음
- 일부 row만 문제, 나머지 정상 → data dispatching
    - 완전 중단은 과한 반응, 살릴 수 있는 데이터는 살림
- 문제가 치명적이지 않고 판단을 consumer한테 맡길 수 있음 → nonblocking audit
    - 단, consumer가 annotation을 실제로 확인하는 문화 있어야 의미 있음





## (3) 결과
**결론: 나쁜 품질 데이터의 유출은 막지만, 그 대가로 연산 비용·완벽하지 않은 방어·지연시간을 감수한다.**

### 단점 1: 연산 비용 증가

- 배경: audit 단계도 결국 데이터셋을 훑어야 검증 가능
- 문제: 같은 검증 함수를 입력 audit, 출력 audit 양쪽에서 쓰면 데이터셋을 여러 번 스캔하게 됨 → 연산 비용 증가
- 대응
    - 검증 로직을 "가장 포괄적인 지점"에 몰아서 배치
    - 예: NULL 검증이라면, 입력에서 온 NULL과 내 변환 로직이 만들어낸 NULL을 둘 다 잡을 수 있는 **출력 audit 쪽**에 검증을 몰아넣는 게 유리 (입력 audit에서 또 반복할 필요 없음)

### 단점 2: 완벽한 방어가 아님 — 정의 안 한 규칙은 못 잡음

- 배경: audit은 결국 사람이 미리 정의해둔 규칙만 체크
- 문제: 규칙을 깜빡했거나, 시간이 지나면서 데이터 패턴이 바뀌어서 기존 규칙이 낡아버리면, audit을 통과해도 실제로는 문제가 있는 데이터가 새어나감
- 대응: 이건 AWAP 혼자서 완전히 못 막음 → Chapter 9 뒷부분의 **Quality Observation 패턴(Offline/Online Observer)**이 이 빈틈을 메우는 역할

### 단점 3: 스트리밍 환경에서 지연시간(latency) 추가
- 배경: 스트리밍 파이프라인은 원래 저지연이 목적
- 문제: 예를 들어 "특정 시간 윈도우 안 NULL 비율"처럼 윈도우 단위 집계를 audit 조건으로 걸면, 그 윈도우가 다 찰 때까지 기다려야 audit이 끝남 → 데이터 전달이 그만큼 늦어짐
- 대응: 실무에서는 audit 조건 자체를 저지연 스트리밍용으로 가볍게(레코드 단위) 설계하거나, 무거운 집계성 검증은 배치로 분리

### 추가 주의: "이상 = 문제"가 항상 맞는 건 아님
- 배경: 데이터는 살아있는 것이라, 예상 밖 변화가 항상 나쁜 신호는 아님
- 문제: 예를 들어 블로그가 SNS에서 갑자기 화제가 돼서 방문자 수가 폭증하면, audit이 "row 수 급증"으로 알람을 울린다 — 근데 이건 실제 문제가 아니라 좋은 일
- 대응: audit 실패를 무조건 "치명적 오류"로 취급하지 말고, 일부는 알람만 띄우고 사람이 판단하게 두는 것도 방법

### 엔지니어 독백
> AWAP 처음 세팅할 때 흔한 실수가 "일단 파이프라인 죽여버리자" 식으로 audit 규칙을 다 hard failure로 걸어두는 거다. 근데 실제로 운영해보면, 급증/급감 알람이 매번 진짜 사고가 아니다. 캠페인 대박 나서 트래픽 튀는 날도 있고, 명절이라 방문자가 줄어드는 날도 있다.
> 
> 그래서 audit 규칙을 만들 때부터 "이건 반드시 막아야 하는 것(스키마 깨짐, 필수 컬럼 NULL)"과 "일단 사람한테 알려주고 판단은 맡길 것(수치 이상치)"을 나눠서 설계해야 한다. 다 hard failure로 걸면 나중에 진짜 알람이 왔을 때도 "또 오작동이겠지" 하면서 무시하게 되는 게 제일 위험하다 — alert fatigue라고 부르는 현상이다.




## (4) 예시

> AWAP을 실무에서 제일 많이 구현하는 방식이 Airflow DAG다. 
> Task 하나하나가 Audit-Write-Audit-Publish의 각 단계를 그대로 대응시킨다. 
> "이 task가 실패하면 다음 task로 안 넘어간다"는 Airflow의 기본 동작 자체가 
> AWAP의 "audit 실패 시 파이프라인 중단"을 자연스럽게 구현해준다. 
> 그래서 별도 프레임워크 없이 그냥 Airflow task 순서만 잘 짜면 AWAP이 된다.

책 예시는 **Airflow + PostgreSQL 기반 배치 파이프라인**이다. JSON 파일을 읽어서 CSV로 변환한 뒤 PostgreSQL 테이블에 적재하는 흐름이고, 총 4단계로 나눠서 본다.

- 책 예시: Airflow + PostgreSQL 기반 배치 파이프라인
- 이 예시가 택한 전략: **1. 파이프라인 중단 (기본값)**
    - 택한 이유
        - CSV 포맷 자체가 제약조건(constraint) 기능이 없는 constraintless 포맷 → DB가 대신 막아주지 않음
        - 필수 컬럼 NULL 같은 문제는 다운스트림(마케팅 캠페인 의사결정 등)에 바로 영향 → 일부만 살리는 게 더 위험
        - 그래서 "조금이라도 이상하면 무조건 멈춘다"는 가장 엄격한 전략 선택
    - Data dispatching/Nonblocking audit을 안 쓴 이유
        - Data dispatching: row 단위로 유효/무효를 가를 근거(어떤 row가 왜 무효인지)가 이 예시엔 없음 — 검증이 파일 전체/컬럼 전체 단위라 "일부만 골라내기"가 안 맞음
        - Nonblocking audit: 필수 컬럼 NULL은 애초에 "그냥 써도 되는 수준의 이슈"가 아니라서 annotation만 남기고 흘려보내면 안 됨

### 이 예시가 다루는 도메인: 웹사이트 방문 로그 (Chapter 1 케이스 스터디)

- 배경: 이 책 전체가 하나의 가상 회사 시나리오를 씀 — **블로그/웹사이트 운영 회사**가 자기 웹사이트 방문자 로그를 수집·분석하는 상황
- `visits`: 사용자가 웹페이지에 방문할 때마다 생기는 이벤트 로그. "누가, 언제, 어디서(어느 페이지), 어떤 환경(브라우저/기기)으로 방문했나"를 기록

### 필수 컬럼(required_columns) 11개 — 각각 뭘 담는가

- `visit_id`: 방문 이벤트 하나를 구분하는 고유 ID
- `event_time`: 방문이 발생한 시각
- `user_id`: 방문한 사용자를 식별하는 ID (앞서 One Big Table 예시에서 본 그 user_id)
- `page`: 방문한 페이지 경로 (예: home.html, product.html)
- `ip`: 방문자의 IP 주소
- `login`: 로그인 여부 또는 로그인 계정 정보
- `browser`: 사용한 브라우저 이름 (Chrome, Safari 등)
- `browser_version`: 브라우저 버전
- `network_type`: 네트워크 종류 (wifi, cellular 등)
- `device_type`: 기기 종류 (mobile, desktop, tablet)
- `device_version`: 기기 버전/모델

### 왜 이 11개가 "필수"인가 — 도메인적 이유

- `visit_id`, `event_time`, `user_id`, `page`: 이게 없으면 "누가 언제 어디를 봤는지"라는 방문 로그의 존재 이유 자체가 성립 안 함 — 가장 핵심적인 분석 축
- `browser`, `device_type` 등: 마케팅/프로덕트 팀이 "어떤 기기/브라우저 사용자가 많은가"를 분석해서 UI 우선순위를 정하는 데 씀 — 없으면 이 분석이 불가능
- `login`: 로그인 사용자와 비로그인 사용자를 구분해서 퍼널 분석(가입 전환율 등)을 할 때 필요

### 이 도메인이 앞선 문제상황과 연결되는 지점

- (1)문제상황에서 "고유 방문자 수(unique visitor)가 50% 급감"했다는 게 바로 이 `visits` 테이블의 `user_id` 기준 distinct count였음
- (4)예시의 `audit_transformed_file`이 검증하는 것도 결국 이 11개 컬럼 — 특히 `user_id`에 NULL이 섞이면, 애초에 (1)에서 겪은 "집계가 잘못 나오는" 문제의 원인이 될 수 있는 컬럼

### 엔지니어 독백
> 이 visits 테이블이 계속 반복해서 나오는 이유가 있다. 실무에서 "방문 로그"는 거의 모든 회사(이커머스, 미디어, SaaS 할 것 없이)가 다루는 가장 기본적이면서도 데이터 품질 이슈가 제일 자주 터지는 도메인이다. 컬럼 개수가 많고(11개), 각 컬럼이 서로 다른 소스(브라우저가 보내는 값, 서버가 붙이는 값, 로그인 시스템이 붙이는 값)에서 온다. 소스가 여러 개라는 게 곧 "어느 한 곳에서 값이 안 올 가능성"이 크다는 뜻이고, 그래서 AWAP 같은 검증 패턴이 실무에서 정말 자주 쓰인다.


### 1단계: Airflow task 정의 및 순서 연결 (AWAP 뼈대)
```python
audit_file_to_load = PythonOperator(
    task_id='audit_file_to_load',
    python_callable=local_validate_the_file_before_processing
)
transform_file = PythonOperator(
    task_id='transform_file',
    python_callable=flatten_input_visits_to_csv
)
audit_transformed_file = PythonOperator(
    task_id='audit_transformed_file',
    python_callable=local_validate_flatten_visits
)
load_flattened_visits_to_final_table = PostgresOperator(
    task_id='load_flattened_visits_to_final_table',
    sql='/sql/load_file_to_visits_table.sql'
)

(next_partition_sensor >> audit_file_to_load >> transform_file
 >> audit_transformed_file >> load_flattened_visits_to_final_table)
```

- 하는 일
    - Airflow task 4개 정의, `>>`로 실행 순서 강제
- 핵심 문법
    - `PythonOperator`: 파이썬 함수 실행하는 task
    - `PostgresOperator`: SQL 파일을 PostgreSQL에 실행하는 task
    - `>>`: task 의존성 연산자. `A >> B` = "A 성공해야 B 실행"
- AWAP 매핑
    - audit_file_to_load = 1단계 Audit(입력)
    - transform_file = 2단계 Write
    - audit_transformed_file = 3단계 Audit(출력)
    - load_flattened_visits_to_final_table = 4단계 Publish
    
- 핵심 지점: `audit_file_to_load`가 내부에서 예외(Exception)를 던지면, Airflow는 자동으로 그 뒤 task(`transform_file` 이하)를 실행 안 하고 파이프라인을 멈춘다. 이게 "audit 실패 시 파이프라인 중단"의 실제 구현이다.

1단계에서 전체 뼈대(task 순서)를 정의했다. 2단계에서는 첫 번째 audit task(`audit_file_to_load`)가 실제로 뭘 검증하는지 본다.


### 2단계: 입력 Audit — 파일 검증 로직
```python
if f_size < min_size:
    validation_errors.append(
        f'File is too small. Expected at least {min_size} bytes but got {f_size}')
if lines < min_lines:
    validation_errors.append(
        f'File is too short. Expected at least {min_lines} lines but got {lines}')
if invalid_json_line:
    validation_errors.append(
        f'File contains some invalid JSON lines. The first error found was '
        f'{invalid_json_line}, line {invalid_json_line_number}')
if validation_errors:
    raise Exception('Audit failed for the file:\n-' + "\n-".join(validation_errors))
```
- 하는 일
    - 입력 JSON 파일의 크기, 라인 수, JSON 형식 체크
    - 문제 있으면 예외 발생시켜 파이프라인 중단
- 핵심 문법
    - `validation_errors.append(...)`: 문제 발견될 때마다 리스트에 쌓음. 하나 걸리고 바로 멈추는 게 아니라, 전체 검증 다 돌고 한 번에 보고
    - `raise Exception(...)`: 파이썬 예외 발생 문법. Airflow는 `PythonOperator` 안에서 예외 나면 그 task를 FAILED 처리
- 검증 항목 성격
    - 파일 크기·라인 수·JSON 형식 — 전부 메타데이터 수준의 가벼운 체크

2단계는 원본 파일이 "읽을 가치가 있는 파일인가"만 빠르게 확인한다. 이걸 통과해야 3단계(Write, `transform_file`)가 실행된다. Write 단계 자체는 이번 예시에서 자세히 안 다루고, 4단계로 넘어가서 그 결과를 검증하는 로직을 본다.


### 3단계: 출력 Audit — 변환된 데이터의 NULL 검증
```python
required_columns = ['visit_id', 'event_time', 'user_id', 'page', 'ip', 'login',
                     'browser', 'browser_version', 'network_type', 'device_type',
                     'device_version']
cols_w_nulls = []
visits = pandas.read_csv(partition_file(context, 'csv'), sep=';', header=0)
for validated_column in required_columns:
    if visits[validated_column].isnull().any():
        cols_w_nulls.append(validated_column)
if cols_w_nulls:
    raise Exception('Found nulls in not nullable columns:' + ','.join(cols_w_nulls))
```

- 하는 일
    - 2단계(transform_file)에서 만든 CSV를 읽어서, 필수 컬럼에 NULL 있는지 체크
    - 있으면 예외 발생
- 핵심 문법
    - `pandas.read_csv(..., sep=';', header=0)`: CSV를 pandas DataFrame으로 읽음. `sep=';'`=구분자 세미콜론, `header=0`=0번째 row가 컬럼명
    - `.isnull().any()`: 해당 컬럼에 NULL이 하나라도 있으면 True
- 왜 "출력" audit인가
    - 원본 JSON엔 없던 검증 — Write 단계에서 flatten하며 생긴 결과물을 검증하는 것
    - flatten 로직 버그로 컬럼이 비면 여기서 잡힘
- (2)솔루션에서 다룬 "왜 입력/출력을 나눠서 하나" 원칙이 여기서 실제로 보임: 이 CSV 포맷은 데이터베이스 제약조건(Constraints Enforcer)이 없는 constraintless 포맷이라, 이 NULL 체크를 여기서 명시적으로 안 하면 아무도 안 잡아준다.

### 4단계: 실제 사용 시나리오 — 전체 흐름이 맞물려 동작하는 모습
```
1. next_partition_sensor 트리거 → 새 파티션(오늘자 visits.json) 도착 감지
2. audit_file_to_load 실행
   → 파일 크기 OK, 라인 수 OK, JSON 형식 OK → 통과
3. transform_file 실행
   → visits.json을 읽어서 flatten된 visits.csv로 변환
4. audit_transformed_file 실행
   → visits.csv의 필수 컬럼 11개 NULL 체크 → 전부 통과
5. load_flattened_visits_to_final_table 실행
   → visits.csv를 PostgreSQL의 최종 visits 테이블로 INSERT (Publish)


[2026-08-22, 03:15:01] audit_file_to_load: SUCCESS 
[2026-08-22, 03:15:30] transform_file: SUCCESS 
[2026-08-22, 03:16:40] audit_transformed_file: SUCCESS 
[2026-08-22, 03:17:05] load_flattened_visits_to_final_table: SUCCESS [2026-08-22, 03:17:05] DAG visits_pipeline: SUCCESS
```

- 만약 2단계에서 파일이 너무 작았다면? → 1번에서 바로 예외 발생, 3~5번은 아예 실행 안 됨.
- 만약 4단계에서 `user_id` 컬럼에 NULL이 섞여 있었다면? → 여기서 예외 발생, 5번(Publish)까지 도달 못 함 → 나쁜 데이터가 최종 테이블에 절대 안 들어감.

### 최종 요약
```
한 줄 요약: Airflow task 순서(>>)로 AWAP의 4단계를 강제하고, 각 audit task 안에서 예외를 던지는 방식으로 파이프라인을 중단시킨다.

실행 순서: audit_file_to_load(입력검증) → transform_file(변환) → audit_transformed_file(출력검증) → load_flattened_visits_to_final_table(공개)

핵심 라인: raise Exception(...) — audit 함수 안에서 이 한 줄이 실행되는 순간, Airflow가 이후 모든 task를 중단시킨다. 이게 AWAP의 "Publish 이전 관문" 역할을 하는 실제 메커니즘.
```




## (5) 최신트렌드

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


<br><br><br><br>
<br><br><br><br>


# 패턴 #60: 제약 조건 적용자 (Constraints Enforcer)

AWAP 패턴 - (5)최신트렌드에서 본 "**3. DB 네이티브 제약(Delta Lake CHECK constraint)**"이 바로 지금 배우는 **Constraints Enforcer 패턴**.

## (1) 문제상황

**핵심 고통: 파이프라인 코드는 멀쩡한데, 어느 날부터 필수 컬럼에 NULL이 랜덤하게 섞여 들어온다.**

- 등장 컴포넌트
    - **batch pipeline**: visits 데이터를 처리해서 테이블에 쓰는 배치 잡. 몇 달간 문제없이 잘 돌았음.
    - **최종 테이블**: 이 배치 잡이 결과를 적재하는 대상 테이블.

- 엔지니어 독백
> 몇 달째 아무 문제 없이 돌던 배치 파이프라인이었다. 그런데 어느 날부터 최종 테이블에 필수 필드 값이 랜덤하게 NULL로 들어오기 시작했다.
> 
> 원인을 찾으려니 골치가 아팠다. 파이프라인 코드 자체는 안 바뀌었다. 그럼 뭐가 문제냐 — 입력 데이터의 특정 케이스가 내 변환 로직을 타면서 예상 못 한 방식으로 NULL을 만들어내고 있었다.
> 
> 여기서 AWAP처럼 파이프라인 안에 audit 단계를 추가하는 방법도 있다. 근데 이미 이 잡은 복잡했다. 여기에 검증 로직까지 얹으면 코드가 더 지저분해지고, 유지보수 부담이 커진다.
> 
> 내가 원한 건 이거다: **적재(loading) 과정 자체가 데이터 품질 오류가 있으면 알아서 실패하게 만드는 것.** 파이프라인 코드에 검증 로직을 추가하는 게 아니라.

- 개선하니 생긴 지점 (= 이 패턴이 푸는 것)
    - AWAP은 "내 파이프라인 코드 안에서" 검증한다. 검증 로직을 내가 직접 짜고 유지보수해야 한다.
    - 이번엔 반대로, 검증 책임을 **데이터베이스/저장 포맷 자체에 위임**하는 방법이 필요하다 — 이게 Constraints Enforcer.
- 한 줄 정리
    - AWAP: 파이프라인 코드가 "능동적으로" 검증한다 (명령형, imperative)
    - Constraints Enforcer: 테이블에 규칙만 선언해두면 DB가 "알아서" 막아준다 (선언형, declarative)


## (2) 솔루션

**주요컨셉: 데이터 검증 책임을 파이프라인 코드가 아니라 데이터베이스/저장 포맷에 위임한다.**

### 구현 순서
- 1단계: 제약을 걸어야 할 속성(attribute) 식별
    - 매우 비즈니스 특화된 단계 — 어떤 컬럼이 "반드시 있어야 하는지"는 product team이나 법규가 정함
    - 예: orders 데이터셋이면 주문 금액(order amount), 구매자 청구지 주소(billing address) 같은 게 필수 속성으로 지정됨
- 2단계: 제약 종류 결정 (아래 3가지)
- 3단계: DB/저장 포맷에 제약을 실제로 선언

### 제약 종류 4가지
- **1. Type constraint (타입 제약)**
    - 컬럼이 지정된 타입(STRING, INTEGER, TIMESTAMP 등)을 벗어난 값을 못 받게 막음
    - 가장 기본적인 제약, 대부분의 저장 포맷/DB가 기본 지원
    - Schema Consistency 패턴의 뼈대가 되는 제약이기도 함 (스키마 자체가 타입 정의니까)
- **2. Nullability constraint (NULL 허용 여부 제약)**
    - `NOT NULL` — 특정 컬럼은 값이 반드시 있어야 한다고 선언
    - 반대로 `nullable`로 설정하면, consumer 쪽에 "이 컬럼은 NULL일 수 있으니 필터링이 필요할 수 있다"는 신호도 같이 전달됨
    - 문제상황에서 겪은 "랜덤 NULL" 문제를 정확히 여기서 막음
- **3. Value constraint (값 범위/조건 제약)**
    - 하나 혹은 여러 개의 허용값·조건식과 비교 연산자를 써서, 삽입되는 값이 그 조건을 만족하는지 검증
    - 예: `x <= NOW()` — x가 미래 시각이면 안 됨 / `x BETWEEN 1901 AND 2000` — x가 20세기여야 함
    - 조건이 거짓이면 해당 row는 실패로 거부됨
- **4. Integrity constraint (참조 무결성 제약)**
    - 한 테이블의 값이 다른 테이블에 실제로 존재하는 값을 참조하는지 검증
    - 주로 Normalizer 패턴으로 정규화된 트랜잭션 DB에서 씀
    - 예: `visits` 테이블의 `page_id`가 `pages` 테이블에 존재하지 않으면, 그 visit row 자체가 거부됨 — star/snowflake schema의 fact-dimension 관계가 깨지지 않게 막아주는 게 이 제약

### 4가지 제약의 관계 — 점점 엄격해짐
```
Type constraint          → "타입이 맞나?" (가장 느슨, 거의 모든 곳에서 기본 지원)
    ↓
Nullability constraint   → "값이 존재하나?"
    ↓
Value constraint          → "값이 비즈니스 규칙에 맞나?"
    ↓
Integrity constraint     → "다른 테이블과의 관계가 맞나?" (가장 엄격, 지원 여부 제일 제한적)
```
- Type/Nullability는 거의 모든 DB/포맷이 기본 지원하는 표준 기능
- Value constraint는 표현력이 제일 높지만, 지원 여부가 저장 포맷/DB마다 다름 (Delta Lake CHECK, Protobuf validate 등)

### 어디서 쓸 수 있나 — DB뿐 아니라 파일 포맷에도 존재
- 전통적 관계형 DB: 4가지 다 지원하는 게 일반적
- Delta Lake: `CHECK` 연산자로 value constraint 지원, `NOT NULL`도 지원
- Apache Avro, Apache Protobuf: type constraint는 네이티브로 지원. value constraint는 확장 라이브러리(예: protovalidate) 설치 시 지원

### 이 패턴의 성격 — producer와 consumer 양쪽에 미치는 효과
- consumer 입장: 제약이 "이 데이터셋의 형태와 가능한 값 범위가 뭔지" 미리 알려주는 정보(informative) 역할
- producer 입장: 검증을 통과 못 하면 애초에 데이터를 못 넣게 막는 상호작용(interactive) 역할

### 실무 선호 — 상황별
- **Type + Nullability constraint**: 거의 모든 테이블에 기본적으로 걸어야 함. 구현 난이도 낮고, 대부분의 저장소가 기본 지원 — 안 걸 이유가 없음
- **Value constraint**: 비즈니스 로직이 얽힌 복잡한 규칙일 때 사용. 저장 포맷이 지원 안 하면(순수 Parquet/CSV 등) AWAP처럼 파이프라인 코드로 대신 검증해야 함
- **Integrity constraint**: 정규화된 트랜잭션 DB에서 fact-dimension 관계를 지켜야 할 때 필수. lakehouse 계열(Delta, Iceberg)은 이 제약 지원이 제한적이라, 지원 안 되면 dbt test의 `relationships` 같은 별도 도구로 보완

### AWAP과의 결정적 차이 — 검증 시점과 실패 시 동작

|구분|AWAP|Constraints Enforcer|
|---|---|---|
|검증 위치|파이프라인 코드 (내가 짬)|DB/저장 포맷 (선언만 하면 자동)|
|검증 시점|audit 단계에서 명시적으로 실행|INSERT/쓰기 시점에 자동으로 걸림|
|실패 시 동작|내가 짠 예외처리 로직대로 (파이프라인 중단/dispatch/nonblocking 선택 가능)|트랜잭션 전체가 거부됨 (all-or-nothing) — 선택지가 없음|
|구현 난이도|검증 로직 직접 작성 필요|제약 선언 한 줄이면 끝|
|표현력|프로그래밍 언어가 되는 한 뭐든 가능|저장 포맷이 지원하는 문법 안에서만 가능|

### 엔지니어 독백
> AWAP과 Constraints Enforcer 중 뭘 쓸지 고민될 땐 이렇게 판단한다. "이 규칙이 단순한가, 복잡한가?" NOT NULL이나 값 범위처럼 단순한 건 무조건 DB 제약으로 건다 — 코드 한 줄 안 짜도 되고, DB가 훨씬 안정적으로 지켜준다. row 수 급감 감지처럼 통계적이고 복잡한 건 DB 제약으로 표현이 안 되니까 AWAP으로 간다.
> 
> 신입 때 실수하기 쉬운 게, 이 둘을 양자택일이라고 생각하는 거다. 실무에선 거의 항상 같이 쓴다 — 테이블엔 NOT NULL 제약 걸어두고, 그 위에 AWAP으로 복잡한 통계 검증까지 얹는 식으로 이중 방어를 한다.



## (3) 결과

**결론: 데이터 품질을 확실하게 지키는 대신, all-or-nothing이라는 경직성과 producer 중심 설계라는 한계를 감수한다.**

### 단점 1: All-or-nothing semantics (전부 아니면 전무)
- 배경: DB 레벨 제약은 대부분 트랜잭션 단위로 동작
- 문제
    - 입력 row 중 하나라도 규칙을 어기면, **그 트랜잭션의 모든 row가 다 거부됨** — 일부만 살리는 게 불가능
    - DB는 보통 **첫 번째로 만난 에러에서 멈춤** → producer가 문제를 다 알려면 여러 번 왔다 갔다 하며 하나씩 고쳐야 함 (에러 1개 고치고 재시도 → 다음 에러 발견 → 또 고치고 재시도...)
- 대응
    - 문제 전체를 한 번에 다 보고 싶으면, producer 쪽에서 미리 검증 로직을 짜는 방법이 있음
    - 근데 그러면 Constraints Enforcer의 장점(선언만 하면 자동 검증되는 편리함)을 잃어버림 — 결국 AWAP처럼 코드를 다시 짜는 셈

### 단점 2: Data producer shift (producer 중심으로 설계됨)
- 배경: Constraints Enforcer는 데이터를 쓰는 쪽(producer)에게 규칙을 강제하는 방식
- 문제: consumer마다 기대하는 규칙이 다를 수 있음
    - 예: DB엔 `nullable`로 설정된 컬럼인데, 어떤 consumer 입장에선 그 컬럼이 필수여야 함
- 대응: 이런 경우 consumer가 자기 쪽에서 추가로 필터링/검증 로직을 짜야 함 — 즉, Constraints Enforcer 하나로 모든 consumer의 요구를 다 못 만족시킴

### 단점 3: Constraints coverage (표현 가능한 범위의 한계)
- 배경: 저장 포맷/DB마다 지원하는 제약 문법이 다름
- 문제: 특히 파일 기반 포맷(Parquet, CSV 등)은 integrity constraint 같은 걸 아예 지원 안 하는 경우가 많음
- 대응: DB 제약으로 못 커버하는 부분은 AWAP의 검증 로직으로 보완 — 프로그래밍 언어로 짜는 거라 표현력엔 제한이 없음

### 엔지니어 독백
> All-or-nothing이 진짜 실무에서 골치 아픈 지점이다. producer가 매일 수만 건 insert하다가 그중 딱 1건이 제약을 어기면, 나머지 9,999건도 다 같이 튕겨 나간다. "왜 정상인 데이터까지 다 실패하냐"는 항의를 종종 받는다.
> 
> 그래서 이 경직성이 부담스러우면, 제약을 아예 안 걸기보다는 **입력 단에서 미리 걸러내는 계층(staging table)을 하나 더 두는 걸 추천**한다. staging에서 문제 있는 row만 먼저 걸러내고, 검증된 데이터만 최종 테이블로 넣으면 all-or-nothing의 부담을 줄일 수 있다. 이게 사실 AWAP의 "출력 audit" 개념과 Constraints Enforcer를 같이 쓰는 실무 패턴이다.




> Q.단점 2: Data producer shift 이 뭔말인지?

### 상황 설정
- **producer**: visits 파이프라인 (visits 테이블에 데이터를 씀)
- **consumer 1**: 마케팅 대시보드 팀 — visits 테이블을 읽어서 방문자 통계를 냄
- **consumer 2**: 결제 정산 팀 — visits 테이블을 읽어서 로그인 사용자의 구매 전환을 분석함


### 테이블 제약 설정 (producer가 정한 기준)
```sql
CREATE TABLE visits (
    visit_id STRING NOT NULL,
    user_id STRING NOT NULL,
    login STRING,   -- nullable, 제약 없음
    page STRING NOT NULL
) USING delta;

SELECT * FROM visits;

visit_id: v1001
user_id: u900
login: jane_kim
page: home.html
---
visit_id: v1002
user_id: u901
login: NULL
page: product.html
---
visit_id: v1003
user_id: u902
login: mike_lee
page: cart.html
---
visit_id: v1004
user_id: u903
login: NULL
page: home.html
```
- `login` 컬럼은 `NOT NULL`이 아니다. 즉 비로그인 방문자는 `login`이 비어있어도 DB가 그냥 받아준다.
- producer 입장에서는 이게 맞다: 비로그인 방문도 정상적인 visit이니까.


### 문제 발생 지점

**consumer 1 (마케팅 대시보드)**
```sql
SELECT COUNT(*) FROM visits;
```
- `login`이 NULL이든 아니든 상관없음 — 그냥 전체 방문 수만 세면 됨. **아무 문제 없음.**
- **`COUNT(*)`는 row(행) 개수를 세는 거지, 특정 컬럼 값이 있는지 없는지를 보는 게 아니다.** 
- 그래서 `login`이 NULL이든 아니든 상관없이 그냥 row가 존재하면 다 센다.

**consumer 2 (결제 정산 팀)**
```sql
SELECT user_id, login, purchase_amount
FROM visits
JOIN purchases USING (user_id);
```
- 이 팀 로직은 "로그인한 사용자만" 분석 대상이라고 가정하고 짜여 있음.
- 근데 `login`이 NULL인 row가 섞여 들어오면, 이 팀 쿼리/로직이 예상 못 한 값(NULL)을 만나서 오류가 나거나, 잘못된 집계가 나옴.
- **이 팀 입장에서는 `login`이 사실상 "필수(NOT NULL)"여야 했다.**

### 근데 왜 DB가 이걸 못 막아주나
```
DB(테이블) 제약: login은 nullable로 딱 하나만 정의되어 있음
                          │
              ┌───────────┴───────────┐
              ▼                       ▼
     consumer 1 기준              consumer 2 기준
     "login 없어도 됨"            "login 반드시 있어야 함"
     (문제 없음)                  (요구사항 다름, DB는 이걸 모름)
```
- 테이블 제약은 **테이블당 하나만** 걸린다. producer가 "login은 nullable"이라고 한 번 정하면, 그 제약은 이 테이블을 읽는 모든 consumer한테 동일하게 적용된다.
- consumer 2만을 위해 "login은 NOT NULL이어야 한다"는 별도 제약을 그 위에 얹을 방법이 DB 레벨에는 없다. (테이블은 하나, 제약도 하나)

### 그래서 consumer 2가 해야 하는 일 (대응)
```sql
-- consumer 2가 자기 파이프라인에서 추가로 필터링
SELECT user_id, login, purchase_amount
FROM visits
WHERE login IS NOT NULL   -- 이 필터를 consumer 2가 직접 추가해야 함
JOIN purchases USING (user_id);
```
- consumer 2는 자기 코드에서 `WHERE login IS NOT NULL` 같은 추가 검증/필터링을 직접 짜야 한다.
- 이게 바로 "Constraints Enforcer 하나로 모든 consumer의 요구를 다 못 만족시킨다"는 말의 실체다.

### 정리

|구분|내용|
|---|---|
|producer 기준|login은 선택값(nullable) — 비로그인 방문도 정상 데이터로 취급|
|consumer 1 요구|상관없음 — 이미 만족|
|consumer 2 요구|login은 사실상 필수여야 함 — 불만족|
|DB가 해줄 수 있는 것|테이블 전체에 걸리는 제약 하나뿐 (양쪽 다 만족 못 시킴)|
|consumer 2가 해야 하는 것|자기 쪽 코드에서 별도 필터링/검증 추가|

### 엔지니어 독백
> 이게 실무에서 진짜 자주 겪는 갈등이다. 결제팀이 와서 "왜 login이 NULL인 row가 있냐, 우리 파이프라인이 깨진다"고 항의하면, 나는 "그건 정상 데이터다, 비로그인 방문이다"라고 답할 수밖에 없다. producer 입장에선 그게 맞는 스키마니까.
> 
> 그래서 이런 갈등이 생기면 두 가지 선택지가 있다. 하나는 결제팀이 자기 쪽에서 필터를 추가하는 것(위에서 본 방법). 다른 하나는, 아예 결제팀 전용으로 "로그인 사용자만 걸러낸 뷰(view)나 별도 테이블"을 만들어주는 것. 후자가 더 깔끔한데, 그건 사실상 Denormalizer 패턴에서 배운 "consumer별로 최적화된 파생 테이블을 만든다"는 개념과 같은 해법이다.



## (4) 예시

엔지니어 독백부터 시작한다.
> Constraints Enforcer를 실무에서 제일 자연스럽게 쓰는 곳이 Delta Lake다. 
> 
> 테이블 스키마 정의할 때 `NOT NULL` 몇 개 박아두고, 
> 복잡한 비즈니스 룰은 `CHECK` 제약으로 얹는다. 
> 
> 코드 한 줄도 안 짜고 INSERT 시점에 DB가 알아서 막아주니까, 
> 이게 가능한 상황이면 무조건 이걸 먼저 쓴다. 
> Protobuf 쪽은 조금 다른데, 데이터가 DB에 들어가기도 전에 
> 애플리케이션 레벨(메시지 직렬화 시점)에서부터 검증하고 싶을 때 쓴다 
> — Kafka로 이벤트 보내기 전에 미리 걸러내는 식.

- Protobuf(Protocol Buffers)
	- Google이 만든 **데이터 직렬화(serialization) 포맷** — 구조화된 데이터를 바이너리(이진) 형태로 압축해서 주고받기 위한 규격
	- Protobuf 자체가 이미 **type constraint를 네이티브로 지원**한다 — `string visit_id`라고 정의하면, 정수나 다른 타입 값은 애초에 이 필드에 못 들어감 (컴파일 시점에 언어별 코드가 생성되면서 타입이 강제됨)

- 예시1: Delta Lake CHECK constraint
- 예시2: Protobuf + protovalidate

### (예시1) Delta Lake 
### 1단계: 테이블 생성 시 Type + Nullability constraint 걸기
```sql
CREATE TABLE default.visits (
    visit_id STRING NOT NULL,
    event_time TIMESTAMP NOT NULL
) USING delta;
```

- 하는 일
    - visits 테이블 생성하면서 
    - 두 컬럼에 타입 + NOT NULL 동시 적용
	    - visit_id, 
	    - event_time 
- 핵심 문법
    - `컬럼명 STRING NOT NULL`: 컬럼 뒤 NOT NULL → nullability constraint. 이 컬럼에 NULL 넣는 INSERT는 전부 거부
    - `USING delta`: 저장 포맷을 Delta Lake로 지정. CHECK 같은 고급 기능이 가능한 이유


### 2단계: 테이블 생성 후 Value constraint 추가
```sql
ALTER TABLE default.visits ADD CONSTRAINT
    event_time_not_in_the_future CHECK (event_time < NOW() + INTERVAL '1 SECOND');
```

- 하는 일
    -  `event_time` 제약
	    - 항상 "현재 시각+1초" 이전이어야 한다
- 핵심 문법
    - `ALTER TABLE ... ADD CONSTRAINT 제약이름 CHECK (조건식)`: 기존 테이블에 제약 나중에 추가하는 문법. 제약이름은 식별용
    - `CHECK (조건식)`: 조건식이 참인 row만 허용. 여기선 "미래 시각이면 안 된다"는 뜻. +1초는 시스템 클럭 오차 여유분
- 1단계에서 기본 골격(타입, NULL 여부) 잡고, 2단계에서 비즈니스 로직(미래 시각 금지) 얹는 관계

### 3단계: 실제 INSERT 시나리오 (제약 위반 시 결과)
```sql
INSERT INTO default.visits VALUES ('v1005', '2099-01-01 00:00:00');
```

**실행 결과**
```
Error: DELTA_VIOLATE_CONSTRAINT_WITH_VALUES
Reason: CHECK constraint 'event_time_not_in_the_future' 
        (event_time < NOW() + INTERVAL '1 SECOND') violated by row with values:
        event_time = 2099-01-01 00:00:00

>>> 이 트랜잭션의 row는 테이블에 기록되지 않음
```
- event_time이 미래 값(2099년)이라 CHECK 위반 → DELTA_VIOLATE_CONSTRAINT_WITH_VALUES 에러, INSERT 거부
- visit_id에 NULL 넣었다면 DELTA_NOT_NULL_CONSTRAINT_VIOLATED 에러 발생


### Q.파이프라인 중단?
**케이스 1: 예외를 안 잡으면 → 파이프라인 중단**

```python
spark.sql("INSERT INTO visits VALUES (...)")
# 예외 처리(try/except) 없음
```

```
실행 결과:
Traceback (most recent call last):
  ...
DeltaConcurrentModificationException / AnalysisException: DELTA_VIOLATE_CONSTRAINT_WITH_VALUES

>>> 이 Python 스크립트 자체가 여기서 죽음
>>> Airflow에서 돌고 있었다면 이 task가 FAILED 처리됨
>>> >> 로 연결된 다음 task들은 실행 안 됨 (앞서 배운 AWAP의 "파이프라인 중단"과 동일한 결과)
```

**케이스 2: 예외를 잡아서 처리하면 → 파이프라인은 계속 감**
```python
try:
    spark.sql("INSERT INTO visits VALUES (...)")
except Exception as e:
    print(f"이 row 실패, 별도 저장소로 보냄: {e}")
    write_to_dead_letter_storage(bad_row)
    # 파이프라인은 여기서 안 죽고 다음 로직 계속 진행
```

> 실무에서 Constraints Enforcer 쓸 때 예외 처리를 꼭 짜야 하는 이유가 이거다. 예외를 안 잡아두면, 하루 배치 100만 건 중 딱 1건이 이상해서 전체 배치가 그냥 죽어버린다. 새벽에 알람 받고 일어나서 로그 뒤지는 신세가 된다.
> 
> 그래서 보통 이렇게 짠다: try/except로 감싸고, 실패한 트랜잭션이면 그 배치를 더 작은 단위(row 단위나 작은 chunk 단위)로 쪼개서 재시도한다. 그러면 문제 있는 row만 정확히 걸러지고, 나머지는 정상적으로 들어간다. 이게 사실상 AWAP의 "data dispatching" 전략을 Constraints Enforcer 실패 시에도 그대로 적용하는 거다.



### (예시2) Protobuf + protovalidate —
### 실무 시나리오 설정
- 등장 컴포넌트
    - **웹/앱 클라이언트**: 사용자가 페이지 방문할 때마다 이벤트를 만들어내는 곳 (프론트엔드 서버 또는 SDK)
    - **Kafka producer 애플리케이션**: 클라이언트가 보낸 이벤트를 Kafka로 발행(publish)하는 백엔드 서비스
    - **Kafka**: 이벤트를 실시간으로 전달하는 메시징 시스템. 여러 consumer가 구독해서 읽어감
    - **protovalidate**: Protobuf 메시지에 붙인 검증 규칙을 실제로 체크해주는 라이브러리
### 어디서 어떤 문제가 생겼나
> 지금까지 본 Delta Lake 예시는 데이터가 이미 테이블에 도착한 다음 단계에서 막는 방식이었다. 근데 실무에서는 그보다 더 앞단, 그러니까 **Kafka로 이벤트를 쏘기 직전**에 이미 걸러내고 싶은 경우가 많다.
> 
> 왜냐면 Kafka에 한 번 잘못된 이벤트가 들어가면, 그걸 구독하는 consumer가 여러 개(마케팅팀, 결제팀, 로그 저장 파이프라인 등)일 수 있는데, 그 잘못된 이벤트가 이 모든 consumer한테 다 퍼진 다음에야 문제를 발견하게 된다. 그럼 이미 늦다.
> 
> 그래서 백엔드 서비스가 Kafka에 이벤트를 발행하기 직전, **메시지 객체를 만드는 바로 그 코드 지점**에서 검증을 걸어버리는 게 protovalidate의 실무 위치다. "이상한 이벤트는 Kafka 토픽에 아예 발도 못 들이게 막는다."

```
[웹사이트에서 사용자가 페이지 방문]
        │
        ▼
[백엔드 서비스: Visit 메시지 객체 생성]
        │
        ▼
[protovalidate.validate(visit) 호출]  ← 여기가 지금 보는 예시 코드
        │
   ┌────┴────┐
   ▼          ▼
검증 통과    검증 실패
   │          │
   ▼          ▼
[Kafka로    [Kafka에 안 보냄,
 이벤트      에러 로그 남기거나
 발행]       재시도/알림]
```



### 1단계: 메시지 스키마에 검증 규칙 붙이기
- Protobuf 메시지 스키마 정의 (.proto 파일 — 이벤트의 "설계도")
```protobuf
message Visit {
  string visit_id = 1 [(buf.validate.field).string.min_len = 5];
  google.protobuf.Timestamp event_time = 2 [
    (buf.validate.field).timestamp.lt_now = true,
    (buf.validate.field).required = true];
  string user_id = 3 [(buf.validate.field).required = true];
  string page = 4 [(buf.validate.field).cel = {
    message: "Page cannot end with an html extension"
    expression: "this.endsWith('html') == false"
  }, (buf.validate.field).required = true];
}
```

- 이게 뭐야?
	- 이건 코드가 아니라 **스키마 정의 파일**이다. Kafka로 보낼 `Visit`이라는 이벤트가 어떤 필드를 갖고, 각 필드가 어떤 조건을 만족해야 하는지 미리 선언해둔 것.
	- `.proto` 파일을 컴파일하면, Python/Java 등 실제 프로그래밍 언어에서 쓸 수 있는 `Visit` 클래스가 자동 생성된다.
- 하는 일
    - Protobuf 메시지 정의에, 각 필드마다 검증 규칙을 어노테이션으로 붙임
- 핵심 문법
    - `(buf.validate.field).string.min_len = 5`: visit_id 최소 5글자 → value constraint
    - `(buf.validate.field).timestamp.lt_now = true`: event_time이 현재보다 이전 → value constraint. Delta Lake의 CHECK와 같은 의도
    - `(buf.validate.field).required = true`: nullability constraint에 해당. 필드 필수 지정
    - `(buf.validate.field).cel = {...}`: CEL(Common Expression Language)로 커스텀 조건 작성. expression이 검증 로직, message는 실패 시 에러 메시지
- Delta Lake와 차이
    - Delta Lake는 DB(테이블) 레벨, 저장 시점에 검증
    - Protobuf는 애플리케이션이 메시지를 만드는 시점(DB에 넣기도 전)에 검증

### 예시2: Protobuf — 2단계: 실제 검증 호출 시나리오
```python
# 사용자가 방문 이벤트를 발생시킴 → 백엔드가 Visit 객체 생성
visit = Visit(
    visit_id="v1",
    event_time=future_timestamp,  # 버그로 미래 시각이 들어갔다고 가정
    user_id="u900",
    page="home.html"
)

# Kafka로 보내기 직전, protovalidate로 먼저 검증
validate(visit)
```
- `Visit(...)`: 1단계에서 정의한 스키마로 자동 생성된 Python 클래스를 써서, 실제 방문 이벤트 객체를 만드는 코드. 이건 데이터베이스 코드가 아니라 **백엔드 애플리케이션 코드**다.
- `validate(visit)`: protovalidate 라이브러리 함수. 방금 만든 `visit` 객체가 1단계에서 정의한 규칙(min_len, lt_now, required, cel)을 다 만족하는지 체크.

**실행 결과**
```
ValidationError:
  - visit_id: value length must be at least 5 characters (got 2)
  - event_time: value must be less than now

>>> 검증 실패, visit 객체가 다음 단계(예: Kafka 발행)로 넘어가지 못함
```
- visit_id="v1"는 2글자라 min_len=5 위반, event_time은 미래 값이라 lt_now 위반 — 두 문제 한 번에 리포트
- Delta Lake의 all-or-nothing과 비슷하게, 검증 실패 시 메시지가 다음 단계로 못 넘어감

### 3단계: 검증 실패 시 실제 벌어지는 일
```python
try:
    validate(visit)
    kafka_producer.send('visits-topic', visit.SerializeToString())
except ValidationError as e:
    logger.error(f"잘못된 visit 이벤트, Kafka 발행 안 함: {e}")
    # Kafka에 아예 안 보내고 여기서 끝
```

**실행 결과**
```
ERROR: 잘못된 visit 이벤트, Kafka 발행 안 함:
  - visit_id: value length must be at least 5 characters (got 2)
  - event_time: value must be less than now

>>> kafka_producer.send()가 실행 안 됨
>>> 이 이벤트는 Kafka 토픽에 절대 안 들어감
>>> Kafka를 구독하는 어떤 consumer(마케팅팀, 결제팀 등)도 이 나쁜 데이터를 못 봄
```

### 최종 요약
- 한 줄 요약
    - Delta Lake는 테이블 스키마 레벨(CREATE TABLE + ALTER TABLE)에서
    - Protobuf는 메시지 스키마 레벨(어노테이션)에서
    - 각각 type/nullability/value constraint를 선언적으로 건다
- 실행 순서 (Delta Lake)
    - 1단계(타입+NULL 제약으로 테이블 생성) → 2단계(CHECK로 값 제약 추가) → 3단계(INSERT 시 위반하면 자동 거부)
- 실행 순서 (Protobuf)
    - 1단계(메시지 필드에 검증 어노테이션 작성) → 2단계(validate() 호출 시 위반하면 ValidationError)
- 핵심 라인
    - Delta Lake: `CHECK (event_time < NOW() + INTERVAL '1 SECOND')` — 코드 없이 SQL 한 줄로 비즈니스 로직 검증 자동화
    - Protobuf: `(buf.validate.field).required = true` — nullability constraint를 애플리케이션 레벨에서 강제

### Delta Lake 예시와 비교 — 막는 위치가 다르다
```
[Delta Lake 예시]
사용자 방문 → 배치 파이프라인 처리 → 테이블에 INSERT 시도 → 여기서 막힘 (이미 여러 단계 지난 뒤)

[Protobuf 예시]
사용자 방문 → 백엔드가 이벤트 객체 생성 → Kafka 발행 직전 → 여기서 막힘 (가장 앞단)
```

### 엔지니어 독백
> Delta Lake랑 Protobuf 둘 다 같은 Constraints Enforcer 패턴인데, 막는 위치가 파이프라인 최후방이냐 최전방이냐 차이다. 실무에서는 둘 다 걸어두는 게 이상적이다 — Protobuf로 소스에서부터 막고, 혹시 그걸 뚫고 들어온 게 있어도 Delta Lake CHECK가 최종 방어선으로 한 번 더 잡아준다.
> 
> 신입 때 헷갈리는 게 "Protobuf가 Kafka 전용 기술이냐"인데 아니다. Protobuf는 그냥 직렬화 포맷이고, Kafka는 그걸 실어 나르는 메시징 시스템 중 하나일 뿐이다. gRPC 통신에서도 Protobuf 쓰고, 파일로 저장할 때도 쓴다. 여기 예시에서 Kafka가 등장하는 건 "실시간 이벤트 스트리밍에 Protobuf+Kafka 조합이 실무에서 제일 흔하기 때문"이다.




## (5) 최신트렌드

### 1. dbt tests의 relationships / not_null
- 정체
    - SQL 변환 파이프라인 도구 dbt가 제공하는 선언적 데이터 검증 기능
- 이전 한계
    - FK(foreign key) 제약은 원래 INSERT마다 "이 값이 참조 테이블에 실제 존재하나"를 실시간으로 확인(lookup)함
    - 대용량 배치 DW(Redshift, BigQuery 등)는 하루 수백만~수억 건을 한 번에 적재 → 매 row마다 lookup하면 적재 속도가 크게 느려짐
    - 그래서 이런 DW들은 아예 FK 제약의 **실제 검증 자체를 안 함** (선언은 가능해도 강제 안 함)
    - 검증이 없으니 존재하지 않는 참조값(유령 값)이 조용히 테이블에 쌓여도 아무도 모름
- 요즘 쓰는 이유
    - DB가 INSERT 시점에 실시간으로 못 막아주는 상황이니, **파이프라인이 다 돌고 난 뒤에 배치로 한 번 검사해주는 방식**으로 우회
    - dbt test의 relationships가 이 역할 — "INSERT 성능은 안 건드리고, 적재 끝난 뒤 별도로 훑어서 유령 참조가 있었는지 찾아냄"
    - 실시간 강제만큼 즉각적이진 않지만, DW 적재 속도를 유지하면서도 매일 검증 리포트를 받을 수 있다는 게 체감 이득


### 2. Iceberg / Delta Lake의 Generated Columns
- 정체
    - 특정 컬럼 값을 다른 컬럼에서 자동 계산해서 채워주는 기능. CHECK처럼 값을 사후 검증하는 게 아니라, 애초에 정해진 공식으로만 값이 채워지게 강제
- 이전 한계
    - 예전엔 "이미 들어온 값이 조건을 만족하는가"만 체크 가능 (CHECK 방식)
    - 파생값(예: event_time에서 날짜만 뽑은 partition 컬럼) 계산은 파이프라인 코드에서 매번 직접 짜야 했음
    - 사람이 코드로 매번 계산하다 보니, 실수로 다른 공식을 쓰거나 값을 빠뜨릴 여지가 있었음
- 요즘 쓰는 이유
    - `GENERATED ALWAYS AS (CAST(event_time AS DATE))`처럼 스키마에 선언해두면 DB가 값을 자동으로 채워줌
    - 사람이 실수로 다른 값을 넣을 여지 자체가 없어짐 — 이것도 일종의 무결성 제약으로 작동


### 3. Kafka Schema Registry + protovalidate/Avro 검증의 결합
- 정체
    - 메시지가 Kafka에 발행되기 전, 스키마 구조 검증(Schema Registry)과 값 검증(protovalidate)을 같은 파이프라인에서 순서대로 적용하는 조합
- 이전 한계
    - 예전엔 스키마(타입/구조) 검증과 값(범위, 조건) 검증이 서로 다른 도구로 따로따로 돌았음
    - Schema Registry는 구조만 확인하고, 값 조건(미래 시각 금지 등)은 애플리케이션 코드가 별도로 짜야 했음
    - 두 검증이 분리돼 있으니, 개발자가 값 검증 코드를 깜빡하면 그 이벤트는 스키마만 맞고 값은 틀린 채로 Kafka에 들어감
- 요즘 쓰는 이유
    - Schema Registry(구조 확인) → protovalidate(값 확인)를 프로듀서 라이브러리 안에 표준으로 묶어둠
    - 신입 개발자가 값 검증 코드를 따로 안 짜도, 라이브러리가 자동으로 두 단계를 다 강제함
    - 사람이 매번 신경 쓰는 대신 시스템이 강제하는 구조로 옮겨간 게 체감 이점


### 실무 선호 정리
- DB 네이티브 제약(Delta CHECK, generated column): 저장소 레벨에서 강하게 막고 싶을 때, 파생 컬럼 자동화까지 원할 때
- dbt test(relationships): FK를 실시간으로 강제 못 하는 분석용 DW에서, 사후 배치 검증으로 대체하고 싶을 때
- Protobuf/Avro + Schema Registry: 스트리밍 이벤트가 소스 시점부터 오염 안 되게, 가장 앞단에서 이중으로 막고 싶을 때

### 엔지니어 독백
> 셋 다 결국 Constraints Enforcer의 핵심 아이디어 — "검증을 코드가 아니라 시스템에 위임한다" — 를 다른 레이어에 적용한 것뿐이다. DB는 저장 레이어에서, dbt는 배치 검증 레이어에서, Schema Registry/protovalidate는 스트리밍 소스 레이어에서.
> 
> 신입 때 이 셋 중 하나만 알면 "이거면 충분하지 않나" 싶은데, 실무에서 파이프라인이 batch면 batch대로, streaming이면 streaming대로 서로 다른 레이어에서 문제가 생긴다. 
> 
> 그래서 지금 이 세션에서 다룬 Denormalizer, AWAP, Constraints Enforcer가 다 서로 다른 방어선을 만드는 패턴이라는 걸 이해하고, 내 파이프라인 형태에 맞는 조합을 골라 쓰는 게 실무 감각이다.


<br><br><br><br>



# 패턴 #61: 스키마 호환성 적용자 (Schema Compatibility Enforcer)


## (1) 문제상황
**핵심 고통: 내 코드는 안 바꿨는데, 남이 스키마를 바꿔서 내 파이프라인이 계속 죽는다.**
- 등장 컴포넌트
    - **sessionization job**: 
	    - 방문 이벤트들을 시간 순서로 묶어서 "한 사용자의 한 방문 세션"으로 재구성하는 파이프라인. 
	    - 예를 들어 한 유저가 홈페이지 → 상품페이지 → 장바구니를 연속으로 봤으면, 이걸 하나의 세션으로 묶어주는 잡.
    - **input data를 생성하는 다른 팀**: 이 sessionization job이 읽는 원본 데이터를 만들어서 넘겨주는 producer 조직.

- 엔지니어 독백
> Stateful Sessionizer로 구현한 세션화 잡이 몇 달간 문제없이 잘 돌았다. 그러다 최근 한 달 사이 몇 번이나 잡이 실패했다.
> 
> 원인을 찾아보니, 입력 데이터를 만드는 다른 팀이 스키마를 바꿔놨다. 그 팀은 내가 쓰고 있는 필드들이 "이제 안 쓰는 거겠지" 생각하고 없애버렸다. 근데 내 sessionization 로직은 그 필드들을 그대로 참조하고 있었다.
> 
> 문제는, 그 팀이 스키마를 바꾸기 전에 나한테 물어보거나 확인할 방법이 아예 없었다는 거다. 그냥 어느 날 데이터가 그 필드 없이 넘어왔고, 내 잡은 그 필드를 찾다가 에러 내며 죽었다.
> 
> 동료들이랑 얘기해보고 요청한 게 이거다: **스키마를 깨뜨리는 변경 자체가 애초에 못 일어나게 막는 장치가 필요하다.**

- 타겟 
    - 앞서 배운 Constraints Enforcer는 "데이터 값"이 규칙을 지키는지 검증했다.
    - 검증대상 변화
	    - 값 → **스키마 자체의 변경** 여부
	    - "이 스키마 변경이 기존 consumer를 안 깨뜨리는가"를 미리 확인하는 게 필요.
- 한 줄 정리
    - Constraints Enforcer: 매 row의 "값"이 규칙을 어기지 않는지 검증
    - Schema Compatibility Enforcer: "스키마 변경 자체"가 기존 consumer와의 계약을 어기지 않는지 검증

<br> 

## (2) 솔루션

주요컨셉: 스키마 변경 시도를, 기존 consumer를 깨뜨리는지 미리 검증해서 거부할 수 있게 만든다.

### 스키마 변경 검증 3가지 방식 (분류 기준: 누가·언제 검증하는가)

- 1. 외부 서비스/라이브러리 방식
    - 검증 시점: producer가 데이터 보내기 직전
    - 검증 주체: producer/consumer와 분리된 별도 서버
    - 대표 사례: Kafka Schema Registry
        - producer/consumer와는 별개로 독립적으로 떠 있는 서버
        - producer → Schema Registry에 "이 스키마 등록해도 되나" 문의
        - 이전 버전과 비교해 통과/거부 판단
        - 거부 시 메시지 전송 자체가 실패
    - 라이브러리형: Apache Avro의 SchemaValidator
        - 검증만 함, 호환성 규칙 직접 설정은 불가

- 2. DB/파일 포맷의 암묵적(implicit) 방식
    - 검증 시점: 쓰기(write) 시도하는 순간
    - 검증 주체: 저장소 엔진 자체 (Delta Lake, 관계형 DB 등)
	    - 관계형 DB나 Delta Lake 같은 테이블 포맷에서 일어남
    - 별도 서버 없음 — 테이블이 이미 아는 기존 스키마와 새 데이터 스키마를 자동 비교
    - 다르면 그 자리에서 쓰기 거부
    - 테이블 생성 시 정의한 제약(타입, nullability)이 곧 암묵적 호환성 규칙
    - 외부 서비스처럼 "호환성 모드"를 명시적으로 선언하는 개념 없음

- 3. DDL 이벤트 기반 방식
    - 검증 시점: DROP COLUMN, RENAME COLUMN 같은 DDL 명령 실행 직전
    - 검증 주체: DB 이벤트 트리거 (PostgreSQL, SQL Server 등 일부 DB만 지원)
    - 트리거 안에 커스텀 검증 함수 삽입, 실패 시 DDL 자체를 롤백
    - 더 단순한 대안: 유저에게 ALTER TABLE 권한 자체를 안 줘서 원천 차단


### 엔지니어 독백 — 스키마 호환성 도입

> 여기까지 보면 신입 입장에선 이런 질문이 들 거다. "3가지 방식 다 '비교해서 판단한다', '자동으로 검증한다'고 하는데, 대체 뭘 기준으로 같다 다르다를 판단하는 거지?"
> 
> 맞는 질문이다. 지금까지는 검증이 어디서, 언제 일어나는지(서버냐, DB 엔진이냐, 트리거냐)만 정한 거고, 그 검증이 실제로 통과/거부를 가르는 기준은 아직 하나도 안 정했다.
> 
> 이 기준을 실무에서는 스키마 호환성(schema compatibility)이라고 부른다. 
> Schema Registry에 스키마를 등록할 때 "이 판단 기준으로 뭘 쓸지" 실제로 설정하는 옵션값이고, Delta Lake가 "다르면 거부"할 때도 내부적으로 이 기준과 똑같은 판단을 하고 있다. 
> 지금부터 이게 뭘 뜻하는지 제대로 본다



### 스키마 호환성 — 왜 이 개념이 필요한가
- 전제: 하나의 데이터셋을 여러 consumer가 읽음
- 전제: consumer들이 항상 같은 시점에 배포되지 않음
    - 예: sessionization job은 지난주 배포된 옛날 코드로 운영 중
    - 예: 다른 job은 오늘 새로 배포됨
    - 이 둘이 동시에 같은 데이터를 읽는 상황이 실무에서 상시 발생
- producer가 스키마를 바꾸면
    - 새로 배포된 consumer: 문제없음
    - 옛날 코드로 도는 consumer: 변경을 모른 채 읽다가 실패할 수 있음
- "스키마 호환된다"의 실제 의미
    - 스키마가 바뀌어도 옛날 버전 consumer, 새 버전 consumer 둘 다 안 죽고 계속 읽을 수 있음
- 검증 대상 정리
    - 값이 아니라 "이 스키마 변경이, 서로 다른 버전으로 동시에 도는 여러 consumer를 다 만족시키는가"
    - 이게 바로 앞서 본 3가지 방식이 각자 "비교해서 판단"하려던 그 판정 기준이다

### 엔지니어 독백 — 스키마 호환성 모드
> 근데 여기서 신입이 또 헷갈리는 지점이 있다. "consumer가 안 죽으면 되는 거 아니냐, 그럼 그냥 하나의 기준만 있으면 되지 왜 여러 모드로 쪼개져 있지?"
> 
> 이유는 "누가 먼저 업데이트되느냐"가 상황마다 다르기 때문이다. 어떨 땐 consumer가 producer보다 늦게 업데이트되고, 어떨 땐 반대로 consumer가 아직 옛날 버전에 머물러 있는 게 아니라 producer 쪽이 옛날 데이터를 계속 갖고 있어야 하는 상황도 있다.
> 
> 그래서 "누구 기준으로 호환을 맞추느냐"에 따라 규칙 자체가 달라진다. 이 기준을 정하는 게 지금부터 볼 backward, forward, full 3가지 모드다.


### 스키마 버전관리 
### 배경지식 1: 스키마도 "버전"이 있다
- 지금까지 스키마를 "테이블 하나의 고정된 구조"로만 생각했을 수 있는데, 실무에서 스키마는 **시간이 지나면서 계속 바뀌는 것**이다
- 스키마가 바뀔 때마다 그걸 v1, v2, v3처럼 버전으로 관리한다
- 코드에 버전(v1.0, v2.0)이 있듯이, 데이터의 "구조 정의"에도 버전이 있는 것
```
v1: {visit_id, page}
v2: {visit_id, page, device_type}   ← device_type 필드 추가되면서 버전 올라감
```

### 배경지식 2: 왜 여러 버전이 "동시에" 존재하나
- producer(데이터 만드는 쪽)와 consumer(데이터 읽는 쪽)는 **서로 다른 팀, 서로 다른 배포 일정**으로 움직인다
- producer가 스키마를 v2로 업그레이드해도, consumer 쪽 코드가 그 순간 바로 같이 업그레이드되는 게 아니다
```
[producer]  오늘 v2로 스키마 변경, 오늘부터 device_type 포함한 데이터 만듦
[consumer]  아직 v1 기준 코드로 운영 중 (다음 주에나 업데이트 예정)
```
- 즉 어느 한순간에 v1 데이터도 있고 v2 데이터도 있고, v1 코드로 도는 consumer도 있고 v2 코드로 도는 consumer도 있다 — **여러 버전이 동시에 뒤섞여 존재**하는 게 실무의 기본 상태

### 배경지식 3: 그래서 무슨 문제가 생기나
- 이렇게 버전이 뒤섞인 상태에서, "어떤 버전의 코드가 어떤 버전의 데이터를 읽어도 안전한가"가 문제가 된다
- 조합은 4가지
```
1) v1 코드 → v1 데이터: 당연히 문제없음
2) v2 코드 → v2 데이터: 당연히 문제없음
3) v2 코드 → v1 데이터: 새 코드가 옛날 데이터 읽음 ← 문제 생길 수 있음
4) v1 코드 → v2 데이터: 옛날 코드가 새 데이터 읽음 ← 문제 생길 수 있음
```
- 3번, 4번 조합에서 문제가 안 생기게 만드는 규칙이 필요하고, 그 규칙에 이름을 붙인 게 **호환성 모드**다
- 3번(새 코드가 옛날 데이터 읽기)을 보장하는 게 **Backward compatibility**
- 4번(옛날 코드가 새 데이터 읽기)을 보장하는 게 **Forward compatibility**



### 프레임워크별 스키마 관리 방법
### 1. Kafka (Schema Registry)

- 저장 위치: Schema Registry 서버 (내부적으로 Kafka의 `_schemas` 토픽에 저장)
- 등록 시점: producer가 스키마와 함께 메시지를 처음 보낼 때 자동 등록
- 버전 관리: subject(토픽별 이름)마다 버전 번호(1, 2, 3...)가 자동으로 매겨짐
- 호환성 모드: subject 단위로 명시적 설정 (BACKWARD, FORWARD, FULL 등)
```bash
curl -X PUT http://schema-registry:8081/config/visits-topic-value \
  -H "Content-Type: application/json" \
  -d '{"compatibility": "BACKWARD"}'
```


### 2. Delta Lake
- 저장 위치: 테이블 자체의 메타데이터 로그(`_delta_log/`)
- 등록 시점: `CREATE TABLE` 시점에 v1, 이후 `ALTER TABLE`이나 `mergeSchema` 옵션으로 새 버전 생성
- 버전 관리: 별도 버전 번호 개념보다는, 매 스키마 변경이 로그에 순차적으로 append됨
- 호환성 모드: 명시적 backward/forward 설정 없음 — 기본은 엄격 거부, `mergeSchema` 옵션으로 완화 가능
```python
new_data.write.format("delta").option("mergeSchema", "true").mode("append").save(table_path)
```


### 3. Protobuf — 어떤 프레임워크와 엮여서 쓰이나
- Protobuf 자체는 "데이터 직렬화 포맷"일 뿐, 실제로 누가 이 데이터를 주고받느냐는 별개의 프레임워크에 달림. 3갈래로 쓰임

**3-1. gRPC — Protobuf가 원래 태어난 용도**
```protobuf
service VisitService {
  rpc GetVisit (VisitRequest) returns (Visit);
}
```

- Google이 gRPC(원격 프로시저 호출 프레임워크)를 만들면서, 그 통신용 직렬화 포맷으로 Protobuf를 같이 설계함
- `.proto` 파일 하나로 메시지 구조뿐 아니라 API 함수 시그니처(서비스 정의)까지 같이 정의
- 마이크로서비스 간 내부 API 통신에 주로 씀 — REST/JSON보다 빠르고 타입 안전


**3-2. Kafka — Avro의 대안으로 추가된 옵션**
```
[producer 앱] → Visit을 Protobuf로 직렬화 → [Kafka 토픽] → [consumer 앱이 역직렬화]
```
- Confluent Schema Registry는 Avro뿐 아니라 Protobuf 스키마 등록·검증도 지원
- gRPC로 이미 마이크로서비스 통신을 Protobuf로 하던 조직이, Kafka 이벤트에도 같은 `.proto` 스키마를 재사용하는 경우가 실무에서 흔함 — 스키마를 두 번 안 만들어도 됨

**3-3. 파일 저장/배치 처리 — 별도 프레임워크 없이 단독 사용**
```python
with open("visits.pb", "wb") as f:
    f.write(visit.SerializeToString())
```
- Spark 같은 배치 엔진도 Protobuf 파일을 읽고 쓸 수 있음 (Parquet/Avro만큼 표준은 아님)
- 이 경우엔 각 언어의 protobuf 컴파일러가 생성한 클래스만 있으면 되고, 별도 프레임워크 불필요
- 저장 위치: `.proto` 파일이 Git 같은 코드 저장소에 존재
- 등록 시점: `.proto` 파일 작성 후 커밋하는 시점
- 버전 관리: Git 커밋 이력이 버전 이력. 필드마다 붙는 필드 번호(`= 1`, `= 2`)도 일종의 식별자
- 호환성 모드: 명시적 설정 개념은 약함, `buf breaking` 같은 도구로 커밋 전에 사전 검사


### 한눈에 비교

| 구분        | Kafka                       | Delta Lake                 | Protobuf                        |
| --------- | --------------------------- | -------------------------- | ------------------------------- |
| 저장 위치     | Schema Registry (별도 서버)     | 테이블 메타데이터 로그               | Git 저장소의 .proto 파일              |
| 등록 시점     | 메시지 첫 전송 시 자동               | CREATE TABLE 시 / 쓰기 시 옵션으로 | 파일 커밋 시                         |
| 버전 매기는 방식 | 자동 증가 번호                    | 로그 append (번호 개념 약함)       | Git 커밋 + 필드 번호                  |
| 호환성 모드 설정 | 명시적 (BACKWARD/FORWARD/FULL) | 옵션(mergeSchema)으로 암묵적 완화   | 도구(buf)로 사전 검사                  |
| 검증 시점     | 메시지 전송 직전 (API 호출)          | 쓰기 시도 순간 (엔진 자동 비교)        | 커밋/CI 시점 (buf breaking)         |
| 연결 프레임워크  | Kafka(메시징)                  | Spark/Delta 엔진(배치)         | gRPC(주 용도) / Kafka(보조) / 파일(단독) |
|           |                             |                            |                                 |

### 엔지니어 독백
> 이 표를 정리해놓고 보면, 결국 "검증을 언제 하느냐"가 셋의 제일 큰 차이다. Kafka는 런타임(메시지 보낼 때), Delta Lake도 런타임(쓸 때), Protobuf는 그보다 훨씬 이른 시점인 커밋/빌드 타임에 검증한다.
> 
> 그리고 Protobuf는 신입 때 "Kafka 전용 기술"로 오해하기 쉬운데 사실은 반대다. gRPC를 위해 태어났고, Kafka에 쓰이는 건 나중에 Avro 대안으로 추가된 옵션이다. 그래서 "이 회사는 Protobuf 쓴다"고 하면 보통 gRPC로 마이크로서비스끼리 통신하는 조직일 확률이 높고, 그런 조직이 Kafka도 같이 쓰면 이미 만들어둔 `.proto` 스키마를 그대로 재사용하는 경우가 많다.
> 
> 검증 시점이 이를수록 사고를 더 싸게 막는다는 것도 기억해둘 만하다. Protobuf처럼 커밋 시점에 걸리면 코드 리뷰에서 바로 잡히지만, Kafka나 Delta Lake처럼 런타임에 걸리면 이미 배포된 파이프라인이 돌다가 에러를 만나는 거라 대응 비용이 더 크다. 요즘 Kafka 쪽도 CI 단계에서 미리 스키마 검증을 하려는 트렌드가 있는 이유가 이거다.




### 스키마 호환성 모드 3가지 (기준: 누구를 기준으로 맞추는가)
- 1. Backward compatibility (하위 호환)
    - 기준: 새 버전 consumer가 옛날 데이터를 읽어야 함
    - 적용 상황: consumer 배포가 producer보다 늦음 / 과거 데이터를 나중에 다시 읽어야 함
    - 예시: 새 스키마에 optional 필드 추가 → 옛날 데이터엔 그 필드가 그냥 없는 걸로 처리 → 문제없음
    - 허용되는 변경: 필드 삭제, optional 필드 추가
    
- 2. Forward compatibility (상위 호환)
    - 기준: 옛날 버전 consumer가 새 데이터를 읽어야 함
    - 적용 상황: consumer 업데이트가 producer보다 느림 (예: 강제 업데이트 안 되는 모바일 앱)
    - 예시: 새 스키마에서 optional 필드 삭제 → 옛날 consumer는 그 필드가 비어있는 걸로 봄 → 원래 optional이라 계약상 문제없음
    - 허용되는 변경: 필드 추가, optional 필드 삭제

- 3. Full compatibility (완전 호환)
    - 기준: backward + forward 둘 다 만족
    - 적용 상황: 여러 팀이 동시에 쓰는 핵심 공용 이벤트 스키마
    - 허용되는 변경: optional 필드 추가/삭제만 (가장 좁음)


### Backward / Forward Compatibility — 플랫폼별 예시 

### 1. Kafka — Backward / Forward

**Backward (새 코드 v2 → 옛날 데이터 v1)**
```python
# v2 consumer 코드 — device_type을 참조
for message in consumer:
    visit = deserialize(message.value)
    device = visit.get('device_type', None)  # optional로 추가됐다면 None 처리 가능
```
- v1으로 저장된 옛날 메시지를 이 코드가 읽어도, `device_type`이 optional이면 `None`으로 처리되고 안 죽음

**Forward (옛날 코드 v1 → 새 데이터 v2)**
```python
# v1 consumer 코드 — device_type을 아예 모름
for message in consumer:
    visit = deserialize(message.value)
    print(visit.visit_id, visit.page)  # device_type 필드는 코드에 아예 없음
```

- v2로 만들어진 새 메시지에 `device_type`이 섞여 있어도, 이 코드는 그 필드를 아예 안 쳐다보니 무시하고 넘어감


### 2. Delta Lake — Backward / Forward

**Backward (새 코드가 옛날 파티션 읽기)**
```python
# v2 스키마 기준 코드 — ad_id 컬럼 있다고 가정하고 짬
df = spark.read.format("delta").load(table_path)
df.select("visit_id", "ad_id").show()
```
- 만약 과거에 쌓인 파티션에 `ad_id`가 없다면? Delta Lake는 스키마 진화(schema evolution)가 켜져 있으면 없는 컬럼은 NULL로 채워서 읽어줌 — optional 추가라면 문제없음
- 만약 `ad_id`가 NOT NULL로 강제돼 있었다면, 옛날 파티션엔 그 값이 없어서 읽기 자체가 실패할 수 있음


**Forward (옛날 코드가 새 파티션 읽기)**
```python
# v1 스키마 기준 코드 — ad_id를 아예 모름
df = spark.read.format("delta").load(table_path)
df.select("visit_id", "page").show()
```
- 새로 쌓인 파티션에 `ad_id` 컬럼이 추가돼 있어도, 이 코드는 `visit_id`, `page`만 select하니까 문제없이 동작


### 3. Protobuf — Backward / Forward

**Backward (새 코드 클래스가 옛날 직렬화 데이터 읽기)*
```python
# v2 .proto로 재생성된 Visit 클래스
visit = Visit()
visit.ParseFromString(old_v1_bytes)   # v1 시절에 저장된 바이트를 v2 클래스로 파싱
print(visit.device_type)  # proto3에서 optional 필드는 기본값(빈 문자열 등)으로 채워짐
```
- Protobuf는 proto3부터 필드가 기본적으로 optional 취급이라, 없는 필드는 자동으로 기본값(zero value)으로 채워짐 — 파싱 자체는 안 깨짐



**Forward (옛날 코드 클래스가 새 직렬화 데이터 읽기)**
```python
# v1 .proto로 생성된 Visit 클래스 (device_type 필드 자체가 클래스에 없음)
visit = Visit()
visit.ParseFromString(new_v2_bytes)   # v2가 만든, device_type 포함된 바이트를 v1 클래스로 파싱
```
- Protobuf는 모르는 필드를 자동으로 무시하고 넘어감(unknown field로 보존은 하되 에러 안 냄) — 안 죽음

### 정리 — 플랫폼마다 "안 죽는 방식"은 다르다

|플랫폼|Backward 안전장치|Forward 안전장치|
|---|---|---|
|Kafka (Avro)|optional 필드는 없으면 None/default 처리|옛날 코드가 새 필드를 코드에 안 넣었으면 그냥 무시|
|Delta Lake|schema evolution 켜져 있으면 없는 컬럼 NULL 처리|select 안 한 컬럼은 그냥 안 읽힘|
|Protobuf|optional 필드는 zero value로 자동 채워짐|모르는 필드는 unknown field로 무시|

### 엔지니어 독백
> 셋 다 "안 죽는다"는 결과는 같은데, 그 밑에서 실제로 일어나는 메커니즘은 플랫폼마다 다르다. Kafka/Avro는 Schema Registry가 사전에 등록을 막아주는 것과 별개로, 실제 파싱 레벨에서도 저런 처리가 있고, Protobuf는 애초에 프로토콜 설계 자체가 "모르는 필드는 무시한다"는 걸 기본 철학으로 깔고 간다. Delta Lake는 schema evolution 옵션을 안 켜두면 오히려 이런 유연함이 하나도 없이 그냥 다 막아버린다는 게 차이점이다.



### Transitive vs Nontransitive (기준: 몇 버전 전까지 보장하는가)
#### 엔지니어 독백 — 도입

> 지금까지 backward, forward는 다 "바로 옆 버전(v1↔v2)"끼리만 비교했다. 근데 실무에서 데이터를 오래 쌓아두는 팀은 v1이 아니라 6개월 전 v0 데이터를 지금 v5 코드로 다시 읽어야 하는 상황이 생긴다. 바로 옆 버전끼리만 안전하다고 해서, 멀리 떨어진 버전끼리도 안전하다는 보장은 없다. 이 문제를 다루는 게 transitive/nontransitive다.
#### 배경
- backward/forward는 기본적으로 인접한 두 버전(v, v+1)만 비교
- 실무에서 데이터를 오래 보관하고 나중에 재처리(reprocessing)하는 경우, 멀리 떨어진 버전 간 호환도 중요해짐
#### 정의
- Nontransitive: 인접한 두 버전끼리만 호환성 보장 (v1↔v2, v2↔v3는 보장하지만 v1↔v3는 보장 안 함)
- Transitive: 모든 과거·미래 버전 간 호환성 보장 (v1↔v2↔v3 전부 다 보장)

#### 다음 예시로 감을 잡는다
- order 데이터셋의 amount 필드가 v0(없음) → v1(optional, 기본값 0.0) → v2(required)로 바뀌는 사례
- v1→v2는 nontransitive 기준으로 통과하지만, v0→v2는 transitive 기준으로 위반됨 — 이걸 다음에 실제 값으로 확인한다


### 사례: Transitive vs Nontransitive 실제 차이

#### v1, v2가 변경사항
```
v0: {order_id}                                    ← amount 필드 자체가 없음
v1: {order_id, amount DOUBLE DEFAULT 0.0}         ← amount 있음, optional인데 기본값 0.0
v2: {order_id, amount DOUBLE REQUIRED}            ← amount 있음, required
```

#### 핵심: v1 데이터는 "optional"이어도 실제로는 amount 값이 항상 존재한다
- v1이 "optional"이라는 건 **스키마 선언**이 optional이라는 뜻이지, 실제로 저장된 데이터에 그 값이 없다는 뜻이 아니다
- v1은 default 값이 `0.0`으로 정해져 있어서, producer가 amount를 안 넣어도 시스템이 자동으로 `0.0`을 채워 넣는다
- 결과적으로 **v1으로 저장된 모든 레코드는 amount 필드에 실제 값이 다 들어있다** (사람이 넣었든, 기본값으로 채워졌든)

```
v1 데이터 실제 저장 형태:
{order_id: 501, amount: 29.99}   ← 사람이 직접 넣은 값
{order_id: 502, amount: 0.0}     ← 안 넣었지만 기본값으로 자동 채워짐

→ 어느 쪽이든 amount 필드에 "값이 없는" row는 존재하지 않음
```

#### 그래서 v2(required) 코드가 v1 데이터를 읽어도 문제없다
- v2 코드는 "amount가 반드시 있어야 한다"고 요구하는데, v1 데이터는 이미 (기본값이든 실값이든) amount가 다 채워져 있다
- 요구하는 것과 실제로 있는 것이 일치 → 안 깨짐

#### 반면 v0은 애초에 amount 필드가 없었다
```
v0 데이터 실제 저장 형태:
{order_id: 100}   ← amount라는 개념 자체가 없음, 채울 기본값도 없음
```
- v2 코드가 v0 데이터에서 amount를 찾으면 → 정말로 아무것도 없음, 채울 방법도 없음 → 깨짐

#### 이게 nontransitive/transitive와 연결되는 지점
- **Nontransitive**: v1→v2만 체크 → v1은 항상 값이 있었으니 통과
- **Transitive**: v0→v1→v2 전체를 체크 → v0까지 거슬러 올라가면 값이 없는 지점이 나와서 위반
- 즉 nontransitive는 "가장 가까운 과거"만 보니까 운 좋게 통과하지만, transitive는 "제일 먼 과거"까지 다 보니까 숨어있던 구멍(v0)을 찾아낸다

#### 엔지니어 독백
> 이 부분이 실무에서 제일 헷갈리는 지점이다. "required 추가 = 무조건 위험"이라고 단순하게 외우면 안 된다. 진짜 기준은 "이 필드가 이전 버전에서도 항상 값을 갖고 있었는가"다. optional이어도 default 값이 있으면 사실상 항상 값이 있는 거나 마찬가지라, required로 승격시켜도 안전할 수 있다.
> 
> 그래서 나는 필드를 optional로 처음 추가할 때 항상 default 값을 신중하게 정한다. 나중에 "이 필드를 필수로 바꾸고 싶다"는 요구가 오면, default가 이미 잘 설계돼 있으면 그냥 required로 바꿔도 nontransitive 기준으로는 안전하게 넘어갈 수 있으니까.


#### "과도기"로 이해하면 딱 맞다
```
v0: amount 없음
        │
        ▼  ← 과도기 시작: 일단 optional + 기본값으로 필드를 슬쩍 끼워넣음
v1: amount OPTIONAL (default 0.0)
        │
        ▼  ← 과도기 끝: 이제 다 채워져 있다는 확신이 생겼으니 required로 승격
v2: amount REQUIRED
```
- v1이라는 중간 단계가 바로 그 "과도기" 역할을 한다
- 이 단계에서 강제는 안 하지만, 기본값으로 실질적으로는 "항상 값이 있는" 상태를 미리 만들어둔다
- 그다음 v2에서 "이제 진짜로 필수다"라고 못을 박아도, 이미 v1부터 사실상 다 채워져 있었으니 아무도 안 깨진다

#### 왜 굳이 이렇게 두 단계로 나누나 — 한 번에 required로 못 가는 이유
- v0에서 바로 v1을 `amount REQUIRED`로 만들었다면?
    - v0 데이터엔 amount가 없어서, 그 즉시 backward compatibility 위반 → Schema Registry가 등록 자체를 거부
- 그래서 무조건 **optional로 먼저 끼워넣고, 데이터가 다 채워질 시간을 준 다음, required로 승격**하는 2단계 절차가 필요한 것

#### 이게 실무에서 일반적인 스키마 진화 패턴
```
1단계: 필드를 optional + 합리적인 default로 추가 (과도기 시작)
2단계: 일정 기간 대기 — 이 기간 동안 모든 producer가 실제로 값을 채우도록 유도
3단계: 데이터가 다 채워졌다고 확신되면, required로 승격 (과도기 종료)
```

#### 엔지니어 독백
> 이 패턴을 "expand-contract" 패턴이라고도 부른다. 먼저 넓게(expand) 열어서 과도기를 만들고, 나중에 좁게(contract) 조여서 확정한다. 스키마뿐 아니라 API 버전 관리, DB 마이그레이션에서도 똑같은 원리가 쓰인다.
> 
> 신입 때 "왜 한 번에 필수로 안 하고 이렇게 번거롭게 두 단계로 가지?" 싶을 텐데, 한 번에 가면 무조건 기존 데이터/consumer가 깨진다. 과도기 없이 스키마를 바꾸는 건 사실상 불가능하다고 보면 된다.



### 실무 선호 — 상황별
- Backward compatibility
    - 적합 상황: 데이터 레이크/웨어하우스처럼 과거 데이터를 계속 쌓고 재조회하는 환경
    - 가장 흔하게 쓰는 기본값
- Forward compatibility
    - 적합 상황: 모바일 앱처럼 클라이언트 업데이트 강제 불가능한 환경
- Full compatibility
    - 적합 상황: 여러 팀이 동시에 쓰는 핵심 공용 이벤트 스키마
    - 가장 안전, 스키마 설계 자유도는 가장 낮음
- Transitive
    - 적합 상황: 옛날 데이터를 자주 재처리하는 환경
- Nontransitive
    - 적합 상황: 스키마가 자주 바뀌고, 아주 오래된 데이터까지 호환될 필요는 없는 환경 — 유연성 우선

### 엔지니어 독백

> Full + Transitive로 제일 엄격하게 걸면 되지 않냐고 생각하기 쉽다. 실제로 해보면 숨 막힌다. 필드 이름 하나 잘못 지어서 고치고 싶어도 못 고친다. 새 필드 추가하고 옛날 필드는 deprecated 표시만 해야 한다.
> 
> 그래서 보통 Backward + Nontransitive로 시작한다. 완전히 안전한 것보다, 바로 다음 배포까지만 안 깨지면 그 사이 문제가 생겨도 빠르게 대응할 여유가 생긴다는 게 현실적인 절충점이다. rename이나 type 변경처럼 진짜 breaking change가 필요하면, 이 패턴 혼자로는 못 풀고 Schema Migrator 패턴으로 넘어가야 한다.


<br><br>


## (3) 결과

**결론: 스키마 파괴 사고는 막되, 관리 오버헤드와 진화 유연성을 대가로 치른다.**

### 단점 1: 상호작용 오버헤드 (Interaction overhead)

- 배경: 외부 서비스(Schema Registry) 방식은 쓰기마다 별도 서버와 통신 필요
- 문제
    - producer는 매번 "최신 버전과 호환되는가" 검증 요청을 보내야 함
    - 통신 자체가 지연(latency)과 인프라 부담으로 누적
- 대응: 검증 결과를 로컬 캐싱 → 매번 네트워크 요청 안 하도록 최적화

### 단점 2: 스키마 진화가 어려워짐

- 배경: 모든 변경이 정해둔 호환성 모드(backward/forward/full)를 통과해야 함
- 문제
    - 필드 이름 하나 바꾸는 것(rename)도 허용 안 됨
    - "삭제+추가"가 아니라 "그냥 이름만 바꾸기"는 호환성 규칙상 불가능
    - 결국 새 필드 추가 + 옛 필드 deprecated 처리라는 우회가 필요
- 대응: rename·타입 변경 같은 케이스는 별도 패턴 필요 → 다음에 배울 **Schema Migrator**

### 엔지니어 독백

> Schema Registry 처음 도입하면 "이제 스키마 사고는 끝났다" 안심하기 쉽다. 근데 실제로 운영해보면 새로운 불편함이 온다. 필드명 오타 하나 고치고 싶어도 backward compatibility에 걸려서 못 고친다. "이름만 바꾸는 건데 왜 이렇게 복잡하냐"는 불만을 자주 듣는다.
> 
> 그래도 이건 감수할 비용이다. 다른 팀이 몰래 필드 지워서 내 파이프라인이 죽는 사고를 막는 대가로, 스키마 유연성을 내주는 거다. 이 트레이드오프를 모르면 그냥 번거로운 규제로만 보인다.


<br><br>
## (4) 예시

엔지니어 독백

> Schema Compatibility Enforcer는 실무에서 두 군데서 제일 자주 마주친다. Kafka 스트리밍의 Schema Registry, 그리고 Delta Lake 배치 쓰기. 컨셉은 같은데 동작 방식이 다르다 — 하나는 외부 서비스에 물어보고, 하나는 테이블 엔진이 알아서 막는다.

### 예시1: Kafka Schema Registry — 1단계: 스키마 등록
```json
{"type": "record", "namespace": "com.waitingforcode.model", "name": "Visit",
 "fields": [
   {"name": "visit_id", "type": "string"},
   {"name": "event_time", "type": "int", "logicalType": "time"}
 ]}
```
- 하는 일: Avro 형식으로 Visit 스키마 정의, Schema Registry에 등록될 첫 버전(v1)
- 핵심 문법
    - `"type": "record"`: 여러 필드를 가진 복합 레코드 타입 선언
    - `"fields": [...]`: 필드 목록, 각각 `name`(필드명)과 `type`(타입) 지정
- 등록 시 호환성 모드(예: forward)를 함께 설정

### 예시1: Kafka Schema Registry — 2단계: 필드 누락된 메시지 전송 시도

```python
new_producer.send('visits_forward', value={'event_time': 1690000000})
```
- 하는 일: 새 producer가 `visit_id` 없이 메시지를 보내려 시도
- Kafka로 가기 전, 내부적으로 Schema Registry가 먼저 이 스키마 등록 가능 여부를 확인

### 예시1: Kafka Schema Registry — 3단계: 실행 결과
```
confluent_kafka.avro.error.ClientError: Incompatible Avro schema: 409
message: 'Schema being registered is incompatible with an earlier schema
for subject "visits_forward-value",
details: [{errorType: 'READER_FIELD_MISSING_DEFAULT_VALUE',
description: 'The field 'visit_id' at path '/fields/0' in
the old schema has no default value and is missing in the new schema'}]

>>> 메시지가 Kafka 토픽에 전송되지 않음
```
- `visit_id`는 원래 필수(default 없음) 필드였는데 새 스키마에서 통째로 빠짐
- Schema Registry가 "이 필드를 기대하는 기존 consumer가 있다"고 판단해 등록 거부
- backward compatibility 관점: required 필드 삭제는 옛날 스키마 형식을 못 지키는 변경이라 위반 처리됨




### 예시2: Delta Lake — 1단계: 기존 테이블 스키마
```
root
|-- visit_id: string (nullable = true)
|-- page: string (nullable = true)
|-- event_time: long (nullable = true)
```
- 현재 Delta 테이블이 가진 스키마 — 3개 컬럼으로 고정

### 예시2: Delta Lake — 2단계: 새 컬럼 포함 쓰기 시도

```python
new_data.write.format("delta").mode("append").save(table_path)
```
- 하는 일: `ad_id`라는 새 컬럼이 섞인 데이터를 기존 테이블에 append 시도
- 핵심 문법
    - `.mode("append")`: 기존 데이터는 안 건드리고 새 row만 추가하는 모드
- 쓰기 시도 순간, Delta 엔진이 들어오는 스키마와 테이블 스키마를 자동 비교 — 별도 서비스 호출 없음

### 예시2: Delta Lake — 3단계: 실행 결과
```
pyspark.errors.exceptions.captured.AnalysisException: A schema mismatch detected when
writing to the Delta table

Table schema:
root
-- visit_id: string (nullable = true)
-- page: string (nullable = true)

Data schema:
root
-- visit_id: string (nullable = true)
-- page: string (nullable = true)
-- ad_id: string (nullable = true)

>>> 쓰기 자체가 거부됨, 테이블에 반영 안 됨
```

- `ad_id`가 테이블 스키마엔 없어서 거부
- Kafka와의 차이: Kafka는 별도 서버가 판단, Delta Lake는 엔진 자체가 즉석 비교 — (2)솔루션의 "암묵적 방식" 실제 동작

### Delta Lake — Schema Evolution 켜고 끄는 법

### 기본 동작: 기본값은 "꺼짐" (엄격 모드)

- Delta Lake는 기본적으로 스키마가 다르면 무조건 쓰기를 거부한다 (앞서 본 `AnalysisException`)
- Schema evolution은 **쓰기 시점에 옵션을 명시적으로 켜야만** 작동하는 기능

### 켜는 방법 — 쓰기 코드에 옵션 추가
```python
# 옵션 없이 쓰면: 스키마 다르면 그냥 거부됨 (기본 동작)
new_data.write.format("delta").mode("append").save(table_path)

# mergeSchema 옵션을 켜면: 새 컬럼이 있어도 자동으로 테이블 스키마에 병합해서 허용
new_data.write.format("delta") \
    .option("mergeSchema", "true") \
    .mode("append") \
    .save(table_path)
```
- `mergeSchema=true`: 새로 들어오는 데이터에 있는 컬럼이 기존 테이블에 없으면, **테이블 스키마 자체를 업데이트**해서 그 컬럼을 추가해버림
- 기존 row들은 그 새 컬럼에 대해 자동으로 NULL이 채워짐 (이게 "옛날 데이터에 없는 컬럼을 optional 취급"하는 실제 동작)

### 세션 레벨로 항상 켜두는 방법
```python
spark.conf.set("spark.databricks.delta.schema.autoMerge.enabled", "true")
```
- 이 설정을 켜두면, 매번 `.option("mergeSchema", "true")`를 안 붙여도 모든 쓰기에 자동 적용됨
- 실무에서는 보통 이걸 잡 단위로 켜두기보단, 명시적으로 매 쓰기마다 옵션을 붙이는 걸 선호 — 의도치 않은 스키마 변경이 조용히 반영되는 걸 막기 위해

### 반대로 "덮어쓸 때" 스키마를 완전히 새로 정의하고 싶다면
```python
new_data.write.format("delta") \
    .option("overwriteSchema", "true") \
    .mode("overwrite") \
    .save(table_path)
```
- `overwriteSchema=true`: 기존 스키마를 아예 무시하고, 이번에 쓰는 데이터의 스키마로 테이블 스키마를 통째로 교체
- `mergeSchema`(추가)와 다르게 이건 기존 컬럼도 없앨 수 있는 훨씬 위험한 옵션 — 보통 완전히 스키마를 재설계할 때만 씀

### 정리

|옵션|동작|위험도|
|---|---|---|
|(옵션 없음, 기본값)|스키마 다르면 무조건 거부|안전 (제일 엄격)|
|`mergeSchema=true`|새 컬럼 있으면 기존 스키마에 병합(추가)|중간 (컬럼 늘리는 것만 허용)|
|`overwriteSchema=true`|스키마를 통째로 교체|위험 (컬럼 삭제도 가능)|

### 엔지니어 독백
> 실무에서 `mergeSchema`는 로그성 데이터(이벤트, 클릭스트림 등)처럼 필드가 계속 늘어나는 데이터셋에 자주 켜둔다. 근데 이걸 무분별하게 켜두면, producer 쪽에서 오타로 만든 컬럼(`devic_type` 같은)까지 그냥 테이블에 병합되어 버리는 사고가 난다. 그래서 `mergeSchema`를 켜더라도, 이 세션에서 배운 Constraints Enforcer(예상 컬럼 목록에 없는 건 거부)를 같이 걸어두는 게 안전하다.

### 최종 요약
```
한 줄 요약: Kafka는 외부 Schema Registry가 등록 시점에 호환성을 판단하고, Delta Lake는 테이블 엔진이 쓰기 시점에 스키마를 직접 비교한다.

실행 순서 (Kafka): 스키마+호환성 모드 등록 → 위반 스키마 전송 시도 → Registry가 거부, 전송 실패
실행 순서 (Delta Lake): 기존 스키마 확인 → 새 컬럼 포함 쓰기 시도 → 엔진이 불일치 감지, 쓰기 거부

핵심 라인: Kafka는 ClientError 발생 지점 — 메시지가 Kafka에 도달하기 전에 막힘. Delta Lake는 AnalysisException 발생 지점 — 쓰기가 테이블에 반영되기 전에 막힘. 둘 다 "나쁜 스키마가 저장소에 들어가기 전에 차단한다"는 원리는 같음.
```

---

## (5) 최신트렌드

**목적: 요즘 실무에서 스키마 호환성을 어떤 도구·워크플로로 관리하는지.**

### 1. Confluent Schema Registry의 CI/CD 통합
- 정체: 스키마 등록을 배포 파이프라인(CI/CD)에 넣어, 코드 배포 전에 호환성을 미리 검증하는 방식
- 이전 한계
    - 스키마 등록이 운영 코드가 실제 실행되는 시점(runtime)에 처음 일어남
    - 배포 후에야 "호환 안 된다"는 걸 알게 됨 — 롤백이 번거로움
- 요즘 쓰는 이유

> 예전엔 스키마 문제를 운영 중에 발견하는 게 흔했다. 요즘은 CI 파이프라인에 "사전 호환성 검증" 단계를 넣어서, 코드가 실제 배포되기 전에 Schema Registry한테 미리 물어본다. 배포 전에 알면 PR만 고치면 끝인데, 배포 후에 알면 롤백하고 재배포해야 한다. 이 차이가 크게 체감된다.

### 2. Buf (Protobuf 스키마 관리 도구)
- 정체: Protobuf 스키마의 버전 관리, 호환성 검사, 코드 생성을 통합한 CLI/서비스 도구
- 이전 한계
    - Protobuf는 원래 호환성을 자동 검사해주는 표준 도구가 마땅치 않음
    - 팀마다 직접 스크립트를 짜서 확인해야 했음
- 요즘 쓰는 이유

> `buf breaking` 명령 하나로 이 변경이 기존 스키마를 깨뜨리는지 바로 확인된다. 예전엔 필드 번호 재사용이나 필수 필드 삭제 같은 실수를 사람이 리뷰로 걸러야 했는데, 이제 도구가 자동으로 잡아준다. 사람이 놓치는 실수를 도구가 대신 잡아주니 리뷰 부담이 줄었다.

### 3. Iceberg의 Schema Evolution API
- 정체: 컬럼 추가·삭제·이름변경·타입변경을 메타데이터 레벨에서 다루는 Iceberg 자체 기능
- 이전 한계
    - Delta Lake의 암묵적 방식은 "다르면 무조건 거부"가 기본
    - 의도적인 변경(필드 추가 등)조차 매번 수동으로 허용 옵션(`mergeSchema` 등)을 켜야 함
- 요즘 쓰는 이유

> Iceberg는 스키마 변경을 명시적인 작업으로 다룬다. `ALTER TABLE ... RENAME COLUMN` 같은 명령으로 스키마를 안전하게 진화시키고, 기존 데이터 파일은 안 건드리고 메타데이터만 갱신한다. Delta Lake가 "일단 막고 보자"라면, Iceberg는 "허용된 변경은 유연하게 진화시키자"에 더 가깝다.

### 실무 선호 정리
- Confluent Schema Registry + CI 통합: Kafka 기반 스트리밍, 여러 팀이 각자 프로듀서를 배포하는 조직
- Buf: Protobuf를 쓰는 gRPC/마이크로서비스 환경
- Iceberg Schema Evolution: 배치/레이크하우스에서 스키마를 자주 유연하게 바꿔야 하는 팀

### 엔지니어 독백
> 결국 셋 다 하려는 건 같다 — 스키마가 깨지는 걸 최대한 빨리, 최대한 앞단에서 발견하는 것. 차이는 발견 시점을 배포 전(CI)으로 당기느냐, 도구가 자동 검사해주느냐, 저장소 자체가 유연하게 버전을 관리해주느냐다.
> 
> 신입 때 하나만 배우고 "이걸로 충분하다" 싶을 수 있는데, 실무에서는 스트리밍이냐 배치냐에 따라 쓰는 도구가 완전히 다르다. 오늘 본 Kafka Schema Registry와 Delta Lake 예시가 정확히 이 구분을 보여주고, 이 구분이 실무 감각의 핵심이다.

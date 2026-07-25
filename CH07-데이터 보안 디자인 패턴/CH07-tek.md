# 데이터 엔지니어링 디자인 패턴 - 데이터 보안 디자인 패턴

> 출처: Data Engineering Design Patterns (Bartosz Konieczny, O'Reilly 2025) Chapter 7 | 실무 데이터 엔지니어링 관점 정리

> **도식 기호 범례** — `⚠` 위험·함정(조심해야 할 동작) · `✗` 잘못된 결과(깨진 상태) · `✓` 올바른 결과 · `⇒` 그 결과 도출

---

## 목차

1. [데이터 제거 (Data Removal)](#1-데이터-제거-data-removal)
   - 패턴 #41: 수직 파티셔너 (Vertical Partitioner)
   - 패턴 #42: 제자리 덮어쓰기 (In-Place Overwriter)
2. [접근 제어 (Access Control)](#2-접근-제어-access-control)
   - 패턴 #43: 테이블 세분화 접근자 (Fine-Grained Accessor for Tables)
   - 패턴 #44: 리소스 세분화 접근자 (Fine-Grained Accessor for Resources)
3. [데이터 보호 (Data Protection)](#3-데이터-보호-data-protection)
   - 패턴 #45: 암호화기 (Encryptor)
   - 패턴 #46: 익명화기 (Anonymizer)
   - 패턴 #47: 의사 익명화기 (Pseudo-Anonymizer)
4. [연결성 (Connectivity)](#4-연결성-connectivity)
   - 패턴 #48: 비밀 포인터 (Secrets Pointer)
   - 패턴 #49: 비밀 없는 커넥터 (Secretless Connector)
5. [요약](#5-요약)

> 본 문서는 챕터 7 전체 — **#41·#42(7.1 Data Removal), #43·#44(7.2 Access Control), #45·#46·#47(7.3 Data Protection), #48·#49(7.4 Connectivity)** — 를 다룸.
> ※ 이름이 같은 챕터 8 의 **#51 Vertical Partitioner(스토리지 최적화용)** 와 혼동 주의 — 본 #41 은 **개인정보 삭제(잊힐 권리)** 를 겨냥한 보안 패턴.

---

## 책의 use case (챕터 도입)

> 데이터 가치·데이터 흐름 패턴으로 만든 **접근하기 쉽고 가치 있는 데이터셋** 은 중요한 비즈니스 자산이자,
> **악의적 행위자를 포함한** 다른 시장 참여자들의 시샘 대상이기도 함. 그래서 데이터 엔지니어링은 처리 잡을 짜는 데서 멈출 수 없음.

- **Compliance (규정 준수)** — 최근 몇 년간 큰 주목을 받은 영역. **GDPR(유럽)**·**CCPA(미국)** 같은 데이터 프라이버시 법이 **나(데이터 프로바이더)와 고객(데이터 컨슈머)** 사이의 경계를 정밀하게 규정.
- **Access control (접근 제어)** — 조직 안에서 데이터셋이 열려 있으면 **다른 팀이 실수로 덮어쓸** 수 있고, 그 여파가 나와 다운스트림 컨슈머 모두에게 큼.
- **Data protection (데이터 보호)** — 데이터셋 위치 접근권을 실수로 줘도, **암호화** 같은 보호 계층을 더해 뒀다면 컨슈머는 **복호화 키** 가 있어야 데이터를 읽을 수 있음.
- **Connectivity (연결성)** — 사람이든 애플리케이션이든 데이터 저장소에 연결해 읽고 씀.
  **자격 증명(credentials)을 Git 저장소에 두면** 유출 위험이 커짐 → 외부의 더 안전한 곳에 보관해야 함.

```
[챕터 7 데이터 보안 패턴 — 네 카테고리]
──────────────────────────────────────────────────────────────────────
 7.1 Data Removal     잊힐 권리 대응(개인정보 삭제)      #41 Vertical Partitioner / #42 In-Place Overwriter
 7.2 Access Control   세밀한 접근 제어                #43 Fine-Grained Accessor (Tables/Resources #44)
 7.3 Data Protection  암호화·익명화                   #45 Encryptor / #46 Anonymizer / #47 Pseudo-Anonymizer
 7.4 Connectivity     자격 증명 없이 안전하게 연결       #48 Secrets Pointer / #49 Secretless Connector
──────────────────────────────────────────────────────────────────────
 흐름 — 지울 건 지우고(7.1) → 접근을 좁히고(7.2) → 데이터 자체를 못 읽게 만들고(7.3) → 연결에서 자격 증명을 없앰(7.4).
```

### 패턴 흐름 — 챕터 6에서 챕터 7 으로

데이터를 잇고 나눠 **가치 있는 자산** 을 만들었으니(챕터 6), 이제 그 자산을 **어떻게 지키나** 가 데이터 보안 패턴의 출발점.
그 첫 카테고리가 **데이터 제거(Data Removal)** — 사용자가 삭제를 요청하면 그의 데이터를 지워야 하는 **잊힐 권리** 대응.

```
[패턴 흐름 — 챕터 6에서 챕터 7 으로]
──────────────────────────────────────────────────────────────────────
 챕터 6: 데이터 흐름으로 가치 있는 데이터셋을 만들고 팀 간 공유
      │ 새 과제: 이 자산을 규정에 맞게 "지켜야" 함 (GDPR·CCPA)
      ▼ "사용자가 '내 데이터 지워줘' 라고 하면 어떻게 지우지?"
 7.1 Data Removal (개인정보 삭제)
   #41 Vertical Partitioner — 불변 개인정보를 "딱 한 번만" 저장하도록 컬럼 단위로 분리 → 지울 대상 자체를 줄임
      │ (다음) 리팩터링할 시간·자원이 없는 레거시라면?
      ▼
   #42 In-Place Overwriter — 기존 저장소를 제자리에서 덮어써 삭제 (staging→승격, 블록 회수 필수)
──────────────────────────────────────────────────────────────────────
 핵심 — 데이터 제거는 "어떻게 지우나" 보다 "지울 데이터를 어떻게 적게 만드나(분리)" 가 먼저.
```

---

## 1. 데이터 제거 (Data Removal)

CCPA·GDPR 같은 프라이버시 규정은 여러 준수 요구사항을 정의하는데, 그중 하나가 **개인정보 삭제 요청(personal data removal request)**
— 사용자로부터 삭제 요청을 받으면 그 사용자의 데이터를 **지워야** 함. 이 절은 그 요구를 다루는 **두 구현 접근** 을 소개.

- **#41 Vertical Partitioner** — 데이터 조직을 **컬럼 단위로 분리** 해 삭제할 개인정보를 최소화 (아래 1-1 절).
- **#42 In-Place Overwriter** — 리팩터링 여유가 없을 때 **기존 저장소를 제자리에서 덮어써** 삭제 (아래 1-2 절).

---

### 1-1. 패턴 #41: 수직 파티셔너 (Vertical Partitioner)

> 첫 데이터 제거 패턴 — **똑똑한 데이터 조직(또는 워크플로 격리)** 이 까다로운 문제도 풀어준다는 원칙의 사례.
> 지울 데이터를 나중에 열심히 찾는 대신, **애초에 지울 대상이 적게** 데이터를 배치함.

#### 상황 (Problem)

**책의 use case** — 개인정보 삭제 파이프라인 설계 리뷰에서 지적된 스토리지 오버헤드:

- 새 **개인정보 삭제 파이프라인** 의 첫 설계 문서를 발표. 동료들 반응은 좋았으나 **중요한 스토리지 오버헤드** 를 지적받음.
- 데이터셋의 **여러 컬럼이 불변(immutable)** — 절대 바뀌지 않는데도 **매 레코드마다 반복 저장** 됨.
  예: **생일(birthday)**, **개인 식별 번호(personal ID number)**.
- 동료들의 피드백 — 각 불변 속성을 **딱 한 번만 저장** 하라. 그러면 삭제 요청이 왔을 때 **지울 데이터가 줄어듦**.
- **결정적 제약**: 잊힐 권리(개인정보 삭제)에 대응해야 하는데, 불변 개인정보가 **모든 레코드에 중복** 저장돼 저장·삭제 비용이 과다.

#### 해결 (Solution)

데이터셋을 **가변(mutable) 부분** 과 **불변(immutable) 부분** 두 갈래로 나눔 → **Vertical Partitioner**.

- **수평 vs 수직 파티셔닝** — 파티셔닝에는 두 방향이 있음.
  - **수평(horizontal)** — **관련 속성을 가진 레코드들을 함께** 둠. 대표 예가 배치 증분 처리에 흔한 **날짜 기반 파티션**. (챕터 8에서 다룸)
  - **수직(vertical)** — **각 row 를 쪼개** parts 를 서로 다른 곳에 씀. 이 **속성 기반 분할** 이 효율적인 개인정보 삭제 파이프라인을 가능케 함.
- **분할 절차** — 두 단계.
  - ① **쪼갤 컬럼** 과, 나중에 쪼갠 row 들을 **다시 합칠 때 쓸 속성**(예: `user_id`)을 식별.
  - ② 데이터 수집 잡에 **속성 기반 split 로직** 을 추가 — 한 row 를 두 부분으로 나눠 **각각 별도 저장소** 에 씀.
    매 레코드마다 값이 달라지는 **event 속성** 은 한 저장소로, 안 바뀌는 것·**개인정보 범위** 에 속하는 것은 다른 곳으로.
- **가장 쉬운 구현** — `SELECT` 문(또는 데이터 처리 프레임워크의 attribute projection). 
  **여러 쿼리** 를 날려 각각 **다른 컬럼 집합** 을 타깃으로 잡아 **다른 데이터스토어** 에 씀. 각 쿼리에 **중복 제거(dedup)** 같은 비즈니스 규칙을 붙일 수 있음.

```
[Figure 7-1 재현] Vertical Partitioner — PII·불변 속성을 전용 저장소로 분리
──────────────────────────────────────────────────────────────────────
                                 event_id: 1029384
                                 user_id: 10
 Input          Vertical         address: 12, Dummy Street
 location  ───► partitioner ───► city: Neverland
                                 action: click
                                 visited_page: home.html
                                       │  vertical partitioning
                     ┌─────────────────┴─────────────────┐
                     ▼                                   ▼
           event_id: 1029384                     user_id: 10
           user_id: 10                           address: 12, Dummy Street
           action: click                         city: Neverland
           visited_page: home.html
                     │                                   │
                     ▼                                   ▼
              [ Event data ]                       [ PII data ]
──────────────────────────────────────────────────────────────────────
 매 레코드마다 값이 바뀌는 event 속성 → Event data, 안 바뀌는 PII·불변 속성 → PII data.
 user_id 는 양쪽에 남겨, 컨슈머가 나중에 두 조각을 다시 join 하는 키로 씀.
```

**쉽게 풀어 보면 (방문 로그 예)** — 블로그 방문 이벤트 한 줄에 **매번 바뀌는 것** 과 **절대 안 바뀌는 개인정보** 가 섞여 있음.

- **한 줄에 섞인 두 종류** — `action`·`visited_page` 는 클릭마다 달라지지만, `address`·`city`(개인정보)는 그 사용자에 대해 항상 같음.
- **수직 분할** — 이 한 줄을 **Event data**(가변)와 **PII data**(불변) 두 저장소로 쪼개고, 양쪽에 `user_id` 만 남겨 나중에 이어붙일 키로 씀.
- **삭제가 쉬워지는 이유** — 사용자가 삭제를 요청하면, 매 레코드에 흩어진 개인정보를 다 뒤질 필요 없이
  **PII data 의 그 `user_id` 한 행만** 지우면 됨.

```
[왜 지울 게 줄어드나 — user_id=10, 방문 1,000건 예]
──────────────────────────────────────────────────────────────────────
 ✗ 비파티션 (한 테이블에 다)
   visit_id | user_id | address        | city      | action | visited_page
   1        | 10      | 12, Dummy St   | Neverland | click  | home.html
   2        | 10      | 12, Dummy St   | Neverland | scroll | post.html
   ...      | 10      | 12, Dummy St   | Neverland | ...    | ...        ← address·city 를 1,000벌 중복 저장
   ⇒ 삭제 요청 시 user_id=10 의 1,000행을 스캔·수정

 ✓ 수직 분할 (두 저장소)
   Event data (1,000행, 개인정보 없음)         PII data (user_id 당 1행)
   visit_id | user_id | action | page      user_id | address       | city
   1        | 10      | click  | home      10      | 12, Dummy St  | Neverland   ← 개인정보 1벌만
   2        | 10      | scroll | post
   ...                                     ⇒ 삭제 요청 = PII data 의 1행 delete
──────────────────────────────────────────────────────────────────────
 방문이 많은 사용자일수록 절감 폭이 큼 — 중복 저장되던 개인정보 벌수만큼 그대로 이득.
```

#### 고려사항 (Consequences)

삭제 use case 에는 좋은 성능 최적화지만, **컨슈머 (조회하는 쪽)** 에겐 몇 가지 단점이 따름.

- **Query performance (조회 성능)**
  - 수직 파티셔닝은 일종의 **데이터 정규화(normalization)** — 불변 속성이 가변 속성과 **따로 떨어져** 삶.
    **쓰기(write)** 는 볼륨을 줄여 최적화되지만, **읽기(read)** 는 저하됨 — reader 가 **쪼개진 row 들을 join** 해야 하기 때문.
  - 읽기 연산이 **네트워크 트래픽** 을 수반. 비파티션 버전이라면 같은 row 안에서 **로컬로** 끝났을 읽기가, 이제는 두 저장소를 오가야 함.
- **Querying complexity (조회 복잡도)**
  - 데이터 분리는 쿼리에 **추가 복잡도** 를 가져옴 — 컨슈머는 일부 속성이 **다른 곳에 있음** 을 알아야 함.
  - 완화가 어렵지 않음 — **단일 진입점(view 등)** 으로 노출하거나, **데이터 문서화**(예: data catalog)를 제공하거나,
    **데이터 리니지**(챕터 10 **Dataset Tracker 패턴**)로 분리 구조를 명확히 드러내면 됨.
- **Complexity in a polyglot world (폴리글랏 환경의 복잡도)**
  - 한 데이터셋이 **서로 다른 종류의 저장소**(예: NoSQL DB + 관계형 DB)에 **동시에** 사는 경우.
    **폴리글랏 지속성(polyglot persistence)** 은 reader 에게 좋음 — 각 컨슈머에 **적합한 기술** 로 레코드를 노출.
    예: 검색 기능은 **검색 최적화 DB**, 저지연 마이크로서비스는 **key-value store** 에서 같은 레코드를 원함.
  - 이 경우 **여러 저장소 시스템 전반** 에 수직 파티셔닝을 적용해야 할 수 있음(Figure 7-2).
- **Raw data (원본 데이터)**
  - **원본(분할 전) 데이터** 를 일정 기간 보관해야 하면, 삭제를 위한 **보완책** 이 따로 필요.
    Vertical Partitioner 는 **첫 변환 단계부터만** 적용되기 때문.
  - 쉬운 해법은 분할 전 데이터에 **짧은 보존 기간(retention)** 을 두는 것(삭제 요청 지연 한도를 지키는 선에서).
    단 이는 **backfill 용 데이터 가용성** 을 낮춤.

```
[Figure 7-2 재현] Polyglot persistence + vertical partitioning
──────────────────────────────────────────────────────────────────────
                          ┌─► Immutable personal data ─► RDBMS writer         ─► [ Mutable | Immutable ]
 Input raw ─► Vertical ───┤
      data    partitioner └─► Mutable data            ─► Search engine writer ─► [ Search engine ]
──────────────────────────────────────────────────────────────────────
 같은 레코드를 컨슈머 특성에 맞는 다른 저장소로 — 불변 PII 는 RDBMS, 가변 데이터는 검색엔진.
 잡이 각 row 를 쪼개면, 전용 컨슈머가 자기 저장소 계층에 맞게 처리.
```

| 파티셔닝 | 무엇을 나누나 | 대표 예 | 이 패턴에서의 쓰임 |
|---|---|---|---|
| **수평(horizontal)** | **행(row)** 을 그룹으로 | 날짜 기반 파티션(배치 증분) | 챕터 8 저장 패턴 |
| **수직(vertical)** | **열(column)** 을 저장소별로 | 가변 event ↔ 불변 PII 분리 | 삭제 대상(개인정보)을 한 곳에 모아 최소화 |

#### 구현 예시 (Examples)

**예시 1 — Spark `foreachBatch` 로 수직 분할 (Example 7-1)**

들어오는 레코드를 두 데이터셋으로 쪼개 **각각 다른 Kafka 토픽** 에 씀. 공통 입력이므로 `persist()` 로 한 번만 읽음:
```python
def split_visit_attributes(visits_to_save: DataFrame, batch_number: int):
    visits_to_save.persist()

    # 브랜치 1 — 방문(event) 데이터에서 user 컨텍스트를 제거
    visits_without_user_context = (visits_to_save
        .filter('user_id IS NOT NULL AND context.user.login IS NOT NULL')
        .withColumn('context', F.col('context').dropFields('user'))   # 개인정보 필드 제거
        .select(F.col('visit_id').alias('key'), F.to_json(F.struct('*')).alias('value')))
    # save to visits_without_user_context (Event data 토픽)

    # 브랜치 2 — user 속성만 뽑아 user_context 로 (불변 PII)
    user_context_to_save = (visits_to_save.selectExpr('context.user.*', 'user_id')
        .select(F.col('user_id').alias('key'), F.to_json(F.struct('*')).alias('value')))
    # save to user_context_to_save (PII 토픽)

    visits_to_save.unpersist()
```

> 단순한 **컬럼 기반 변환** 으로 방문 데이터에서 user 정보를 떼어내고, user 속성만 별도 토픽으로 보냄.
> `visit_id` 를 key 로 쓰는 방문 토픽과 달리, user 토픽은 `user_id` 를 key 로 써서 **사용자당 한 레코드** 로 정리됨.

**예시 2 — Delta Lake `MERGE` 로 dedup (Example 7-2)**

분할한 user_context 토픽을 Delta 테이블로 옮기며 `MERGE` 로 **user_id 당 최신 한 건만** 유지:
```python
def save_most_recent_user_context(context_to_save: DataFrame, batch_number: int):
    deduplicated_context = context_to_save.dropDuplicates(['user_id']).alias('new')
    current_table = DeltaTable.forPath(spark_session, get_delta_users_table_dir())
    (current_table.alias('current')
        .merge(deduplicated_context, 'current.user_id = new.user_id')
        .whenMatchedUpdateAll().whenNotMatchedInsertAll()
        .execute())
```

> 불변 속성을 **딱 한 번만** 저장하려는 목표를 MERGE 로 달성 — 같은 `user_id` 는 갱신, 없으면 삽입.

**예시 3 — Delta Lake 삭제 (Example 7-3)**

이렇게 분리해 두면 삭제는 **PII 테이블의 한 행** 을 지우는 것으로 끝:
```python
user_id_to_delete = '140665101097856_0316986e-9e7c-448f-9aac-5727dde96537'
users_table = DeltaTable.forPath(spark_session, get_delta_users_table_dir())
users_table.delete(f'user_id = "{user_id_to_delete}"')
```

> `delete` 만으로는 부족 — **`VACUUM`** 을 추가로 돌려 보존 기간을 넘긴 파일까지 물리 삭제해야 함.
> 안 그러면 **이전 버전(older version)을 읽어** 삭제된 사용자의 데이터를 여전히 복구할 수 있음(Delta 의 time travel).

**예시 4 — Kafka tombstone 메시지 (Example 7-4)**

Kafka 에서는 삭제를 **tombstone(묘비) 메시지** — 삭제할 레코드의 **key + `null` 값** — 로 표현.
`cleanup.policy=compact` 토픽에 보내면 background compaction 이 해당 tombstone 을 지움:
```bash
docker exec -ti ... kafka-console-producer.sh --bootstrap-server .... \
  --topic ... --property parse.key=true --property key.separator=, \
  --property null.marker=NULL

140665101097856_0316986e-9e7c-448f-9aac-5727dde96537,NULL
```

> compaction 후 그 `user_id` 는 토픽에서 사라짐. 단 이 방식은 **key 당 한 개** 인 `user_context` 토픽에서만 유효
> — `visits` 처럼 **여러 이벤트가 같은 visit id key 를 공유** 하는 토픽에 compaction 을 걸면 **마지막 방문만 남아** 데이터가 훼손됨.

```
[Examples 7-1 ~ 7-4 한 흐름 — 분리부터 삭제까지]
──────────────────────────────────────────────────────────────────────
 raw visits ─► split_visit_attributes (7-1, foreachBatch)
                  ├─► visits 토픽         (key=visit_id · 가변 event, 개인정보 제거)
                  └─► user_context 토픽   (key=user_id · 불변 PII)
                             │
                             ▼  MERGE + dropDuplicates(user_id)  (7-2)
                     [ Delta users 테이블 ]   ← user_id 당 1건으로 정리
                             │
        삭제 요청 ──────────►  │  delete(user_id = ...)  (7-3)  ─► + VACUUM (물리 삭제)
                             │
   (Kafka 저장이면)  tombstone  user_id,NULL  ─► compaction 이 제거  (7-4)
──────────────────────────────────────────────────────────────────────
 분리(7-1) → 중복 제거(7-2) → 논리 삭제(7-3) → 물리 정리(VACUUM 또는 compaction) 4단계.
 ⚠ VACUUM/compaction 을 빠뜨리면 이전 버전으로 복구 가능 → 삭제가 "완료" 되지 않음.
```

<details>
<summary><b>⚠ 트러블 로그</b> — Vertical Partitioner로 개인정보만 분리하고 원본(분할 전) 데이터의 보존·삭제를 잊으면 삭제 요청 사용자의 데이터가 원본 레이어에 남아 규정 위반.</summary>
<div markdown="1">

**예 —** `visits_raw` 원본을 backfill 용으로 90일 보관하면서 PII 테이블에서만 `user_id` 를 지우면,
삭제 요청 뒤에도 원본 JSON 에 `address`·`city` 가 남아 GDPR/CCPA 삭제 의무를 못 지킴.
또 Delta 에서 `delete` 만 하고 `VACUUM` 을 빼면 time travel 로 이전 버전을 읽어 복구까지 가능.

**권장 —** 수직 분할은 **첫 변환 단계부터만** 적용됨을 인지하고, 원본 데이터에는 **삭제 지연 한도 내의 짧은 retention** 을 걸 것.
Delta 는 `delete` 후 반드시 **`VACUUM`** 으로 물리 삭제, Kafka 는 **key 당 1건 토픽에만** compaction 기반 tombstone 을 쓸 것.

</div>
</details>

---

### 1-2. 패턴 #42: 제자리 덮어쓰기 (In-Place Overwriter)

> #41 Vertical Partitioner 는 **새 프로젝트를 시작하거나** 기존 워크로드를 마이그레이션할 **시간·컴퓨팅 자원이 충분** 할 때 좋음.
> 그 편안한 위치에 있지 못하면, 검증된 **덮어쓰기 전략** 에 기댈 수밖에 없음.

#### 상황 (Problem)

**책의 use case** — 개인정보 관리 전략이 없는 레거시 시스템을 물려받음:

- **테라바이트급 데이터** 가 **시간 기반 수평 파티션** 에 저장된 레거시 시스템을 인수. **개인정보 관리 전략이 정의돼 있지 않음**.
- 레거시임에도 조직에서 **널리 사용** 중. 그래서 정부의 새 프라이버시 규정을 준수해야 함 — 사용자 요청 시 개인정보를 즉시 제거.
- **결정적 제약**: 아키텍처를 **리팩터링할 자원이 없음**. #41 처럼 처음부터 컬럼을 나눠 설계할 수 없는 상황.

#### 해결 (Solution)

리팩터링 여유가 없으니 **In-Place Overwriter** — 기존 저장소를 **제자리에서 덮어써** 삭제. 구현은 스토리지 기술에 크게 의존.

- **네이티브 in-place 삭제 지원 시** — 제거 대상을 **WHERE 조건** 으로 겨냥해 `DELETE` 실행.
  - Iceberg·Delta Lake 같은 오픈 테이블 포맷은 잘못된 작업을 되돌리는 **time travel** 제공.
    단 개인정보를 삭제해도 **제거된 레코드를 담은 데이터 블록을 회수(reclaim)하지 않으면** 데이터가 그대로 남음(→ #41 의 `VACUUM` 과 같은 맥락).
- **네이티브 삭제 미지원 시**(JSON·CSV 등 원시 포맷) — 전체 데이터셋을 처리해 제거 사용자 레코드를 걸러내는 **시뮬레이션 잡** 실행(컴퓨팅 집약적).
  - **직접 교체 금지** — 잡이 재시도·실패하면 데이터 손실 가능. 대신 **스테이징 영역(staging area)** 에 결과를 쓰고,
    삭제가 성공한 뒤에만 기존 공개 데이터셋을 덮어쓰는 **데이터 승격(promotion) 잡** 실행. 승격 잡이 실패해도 스테이징에 이미 계산돼 있어 데이터 손실로 안 이어짐.
- **Compactable 저장소**(compaction 켠 Kafka 토픽 + key 기반 레코드) — ① 토픽 레코드 읽기 → ② 제거 대상이면 **record key + null payload** 전송.
  compaction 이 key 별 최신 항목만 남기므로, 개인정보가 든 이전 레코드가 제거되고 빈 delete marker 만 남음(#41 예시의 tombstone 과 동일).

```
[In-Place Overwriter — 저장소 기술별 삭제 구현]
──────────────────────────────────────────────────────────────────────
 삭제 요청(user_id)
   │
   ├─ 네이티브 삭제 지원(Delta·Iceberg)   DELETE WHERE user_id=... → 블록 회수(VACUUM) 필수
   │     └ 회수 안 하면 time travel 로 복구 가능 ⚠
   │
   ├─ 원시 파일(JSON·CSV)                전체 스캔·필터 → staging 기록 → 승격(promote)
   │     └ 최종 위치에 직접 덮어쓰기 금지(실패 시 손실) ⚠
   │
   └─ Compactable(Kafka, key 기반)       key + null(tombstone) → compaction 이 이전 레코드 제거
──────────────────────────────────────────────────────────────────────
 공통 대가 — 전체 데이터를 읽고 다시 씀 → I/O·비용이 #41 Vertical Partitioner 보다 큼.
```

> **참고 사항 — 삭제 벡터 (Deletion Vector)**
> 테이블 파일 포맷의 삭제 관리 방식은 둘. ① 제거된 행을 식별해 **작은 side file** 에 기록(쓰기 풋프린트 축소)하는 **deletion vector** — 컨슈머가 읽는 시점에 삭제 행을 직접 걸러냄. ② 반대로 제거 항목을 뺀 전체 데이터를 파일에 새로 씀 — 쓰기 집약적이지만 컨슈머는 바로 사용.

> **참고 사항 — 보존 기간 (Retention Period)**
> 데이터 보존 기간이 삭제 조치 지연 한도보다 짧으면, 그 자체를 삭제 전략으로 볼 수 있음(법정 기간 내 자동 제거). 단 구현 제안일 뿐, 최종 채택 전 **CDO·법무팀과 확인** 할 것.

```
[Figure 7-3 재현] 스테이징을 거친 데이터 제거
──────────────────────────────────────────────────────────────────────
 [ JSON public dataset ] ─► [ 데이터 제거 잡 ] ─► [ Staging area ]
                                                       │ 제거 성공 시에만
                                                       ▼
                               [ 데이터 승격 잡 ] ─► [ JSON public dataset ]
──────────────────────────────────────────────────────────────────────
 제거 잡은 결과를 staging 에만 씀 → 성공 후 승격 잡이 공개 데이터셋을 덮어씀.
 승격 잡이 실패해도 staging 에 이미 계산돼 있어 데이터 손실로 안 이어짐.
```

#### 고려사항 (Consequences)

읽기·쓰기 연산이 많은 패턴 — 둘 다 시스템에 부담.

- **I/O overhead(I/O 오버헤드)**
  - 파일을 읽고 덮어쓰기가 **심각한 I/O** 를 유발. 시간이 지나며 저장 공간이 거의 **2배** 로 커지고 처리량도 늘어남.
  - 완화 — 스토리지 계층이 필터 조건과 무관한 파일 읽기를 피할 수 있으면 작아짐. **Parquet·Delta·Iceberg** 는 데이터 블록별 통계(메타데이터)를 저장 → 쿼리 엔진이 제거 대상 행이 없는 블록을 **스킵**.
- **Cost(비용)**
  - 전체 데이터를 읽어야 하므로 **#41 Vertical Partitioner 보다 비쌈**. 예: 제거 엔티티 1건에 레코드 **2,000건** 이 있으면, Vertical Partitioner 는 **1건만** 읽고 drop, In-Place Overwriter 는 **2,000건** 이 영향받음.
  - 완화 — 삭제 요청을 **묶어서(batch)**, 요청마다 파이프라인 하나씩이 아니라 전체 요청에 대해 한 번 실행.

> **참고 사항 — 되돌릴 수 없는 롤백 (Impossible Rollback)**
> 스테이징 방식도 완벽치 않음. 삭제 잡에 버그가 있어 재실행해야 하는데, **원본이 이미 덮어써져** 원래 데이터셋을 못 씀. 완화 — Proxy 패턴에 기대거나 인프라 수준에서 **데이터 버저닝** 활성화(S3·Azure Storage·GCS 모두 버저닝 지원).

#### 구현 예시 (Examples)

**예시 1 — PySpark 로 플랫 파일에서 레코드 제거 (Example 7-5)**

Delta 예시는 #41 의 Example 7-3 과 동일 코드라 생략. **플랫 파일** 이 더 까다로움 — 전체를 필터링해 staging 에 새로 씀:
```python
input_raw_data = spark_session.read.text(get_input_table_dir())
df_w_user_column = input_raw_data.withColumn(
    'user', F.from_json('value', 'user_id STRING'))       # 필터용 user_id 만 추출
user_id = '139621130423168_029fba78-15dc-4944-9f65-00636566f75b'
to_save = df_w_user_column.filter(f"user.user_id != '{user_id}'").select('value')
to_save.write.mode('overwrite').format('text').save(get_staging_table_dir())  # 최종이 아닌 staging 에
```

> 데이터셋 변경을 최소화하려 **text API** 를 쓰고, 공간 절약을 위해 필터에 필요한 `user_id` 만 추출.
> Spark 는 **분산·비트랜잭션** 처리 계층이라 최종 위치에 바로 덮어쓰지 않고, staging 에 만든 뒤 **rename-like 명령** 으로 승격.

**예시 2 — AWS CLI 로 최종 위치 승격 (Example 7-6)**

```bash
aws s3 rm ${BUCKET}/output --recursive
aws s3 mv ${BUCKET}/staging ${BUCKET}/output --recursive
```

> 클라우드 오브젝트 스토어의 rename 은 로컬 파일시스템과 같은 트랜잭션 시맨틱이 아님
> — 흔히 **copy-and-remove** 로 구현돼, 실패 시 빈 유효 상태(empty valid state)를 남길 수 있음.

<details>
<summary><b>⚠ 트러블 로그</b> — In-Place Overwriter로 삭제하면서 staging을 건너뛰거나 블록 회수를 잊으면 데이터가 손실·잔존.</summary>
<div markdown="1">

**예 —** 시뮬레이션 잡이 최종 경로에 바로 `mode('overwrite')` 로 쓰다가 중간에 죽으면, 공개 데이터셋이 **반쯤 지워진 상태** 로 남아 다운스트림이 깨짐.
반대로 Delta 에서 `DELETE` 만 하고 블록 회수(`VACUUM`)를 빼면, time travel 로 이전 버전을 읽어 삭제한 사용자를 **복구** 할 수 있어 규정 위반.

**권장 —** 원시 포맷은 반드시 **staging → 승격** 2단계로 처리하고, 테이블 포맷은 `DELETE` 뒤 **블록 회수(VACUUM/expire)** 까지 돌릴 것. 원본은 인프라 **버저닝** 으로 롤백 경로를 확보.

</div>
</details>

---

## 2. 접근 제어 (Access Control)

효율적인 데이터 제거만으로는 기본 보안이 안 됨 — 가장 중요한 데이터 구획에는 **인가된 사용자만** 접근하게 해야 함.
개인정보를 비공개로 두는 것도 중요하지만, **데이터 그 자체가 최대 경쟁 자산** 이기도 함.

- **#43 Fine-Grained Accessor for Tables** — 테이블의 **컬럼/행 단위** 접근 제어(고전적 분석 세계). (아래 2-1)
- **#44 Fine-Grained Accessor for Resources** — **클라우드 리소스 단위** 접근 제어(최소 권한 원칙). (아래 2-2)

---

### 2-1. 패턴 #43: 테이블 세분화 접근자 (Fine-Grained Accessor for Tables)

> 사용자·그룹을 만들어 특정 테이블 권한을 주는 고전적 방식에 딱 맞음. 그런데 그보다 **더 세밀한** 제어가 가능.

#### 상황 (Problem)

**책의 use case** — HDFS/Hive 를 클라우드 DW 로 옮기며 마주친 저수준 인가 요구:

- 이전 **HDFS/Hive 워크로드를 클라우드 DW 로 마이그레이션** 한 뒤 보안 접근 정책을 구현해야 함.
- 첫 요구(사용자·그룹으로 **테이블** 접근 관리)는 쉬움 — 새 DW 가 고전적 사용자·그룹을 지원.
- 그러나 이해관계자의 추가 요구 — 테이블 접근 권한이 있어도 **모든 컬럼·행을 읽을 권한은 없을** 수 있음. 이 **저수준 리소스** 에도 인가 메커니즘이 필요.
- **결정적 제약**: 테이블 단위가 아니라 **컬럼·행 단위** 접근 제어가 필요.

#### 해결 (Solution)

**컬럼 기반 접근** 은 세 가지 구현.

- **① GRANT 연산자** — 인가 액션 스코프 안의 컬럼을 정의. Amazon Redshift·PostgreSQL 지원.
  (Example 7-7 — 두 컬럼만 읽기 허용: `GRANT SELECT(col_A, col_B) ON my_table TO some_user;`)
- **② data catalog + 정책 태그** — GCP BigQuery 방식. Data Catalog 에 **정책 태그** 를 만들어 보호 컬럼에 할당하고, 사용자에게 태그별 **Fine-Grained Reader** 역할을 부여.
- **③ data masking** — Databricks(Unity Catalog)·Snowflake 방식. 사용자가 보호 컬럼을 보되 접근권이 없으면 **내용이 숨겨짐**.
  (Example 7-8 — `engineers` 그룹만 `ip` 컬럼을 보는 마스킹 함수)
  ```sql
  CREATE FUNCTION ip_mask(ip STRING)
    RETURN CASE WHEN is_member('engineers') THEN ip ELSE '.' END;

  CREATE TABLE visits (
    visit_id STRING,
    ip STRING MASK ip_mask);          -- 컬럼에 마스킹 함수 부착
  ```

**행 기반 접근(row-level security)** 은 실행 쿼리에 **WHERE 조건을 동적으로** 추가. 이름은 제각각
— Databricks `ROW FILTER`, Amazon Redshift Row-Level Security, GCP BigQuery·Snowflake row access policies.
보호 테이블의 모든 select 에 동적 조건을 더하는 **별도 DB 객체** 를 정의.

- 네이티브 미지원 시 — **view + access guard 조건** 으로 시뮬레이션.
  (Example 7-9 — 쿼리 발행 사용자 소유 blog 만 반환)
  ```sql
  CREATE VIEW users_blogs AS
  SELECT ... FROM blogs WHERE table.blog_author = current_user
  ```
  조건이 사용자마다 달라져, 각 사용자가 **전용 view** 를 가진 것처럼 동작.

```
[Fine-Grained Accessor — 같은 테이블, 사용자마다 다른 뷰]
──────────────────────────────────────────────────────────────────────
 원본 visits     visit_id | city  | ip         | user_id
                 v1       | Seoul | 10.0.0.1   | u-10
                 v2       | Busan | 10.0.0.2   | u-20

 analyst (engineers 그룹 아님, 세션 user_id=u-10)
   컬럼: ip 마스킹(MASK)  +  행: RLS(user_id=current_user)
                 visit_id | city  | ip  | user_id
                 v1       | Seoul | .   | u-10        ← ip 가려짐 + u-20 행은 안 보임

 engineer (engineers 그룹)
                 visit_id | city  | ip         | user_id
                 v1       | Seoul | 10.0.0.1   | u-10   ← ip 원본
                 v2       | Busan | 10.0.0.2   | u-20   ← 모든 행
──────────────────────────────────────────────────────────────────────
 "어느 열(컬럼 마스킹) × 어느 행(row 필터)" 을 곱해 사용자별 뷰가 만들어짐.
```

#### 고려사항 (Consequences)

DB 네이티브 지원이라 앞선 패턴들보다 단점이 상대적으로 적음.

- **Row-level security limits(행 수준 보안의 한계)**
  - 대부분의 row-level 구현은 **연결 세션에서 직접 얻는 속성**(사용자명·그룹·IP)으로 적용 범위가 제한됨.
- **Data type(데이터 타입)**
  - 컬럼이 **nested structure** 같은 복합 타입이면 단순 컬럼 기반 전략을 못 씀. 먼저 **unnest** 해 다른 테이블로 노출하거나(Dataset Materializer 패턴), fine-grained 권한을 지원하면 그걸 사용.
- **Query overhead(쿼리 오버헤드)**
  - row/column 보호가 쿼리에 **동적으로 추가되는 SQL 함수** 로 표현됨 → 예기치 않은 지연 유발 시, 허용된 데이터만 담은 **전용 테이블·view**(Dataset Materializer)로 완화. 단 데이터 중복·거버넌스 부담이 따름.

#### 구현 예시 (Examples)

**예시 — PostgreSQL(컬럼·행) + DynamoDB (Example 7-10 ~ 7-12)**

컬럼 접근 — 나열한 컬럼만 허용, `SELECT *` 는 거부(Example 7-10):
```sql
GRANT SELECT(id, login, registered_datetime) ON dedp.users TO user_a;
-- user_a 가 미포함 컬럼을 조회하면: ERROR: permission denied for table users
```

행 접근 — 정책으로 `login = current_user` 조건을 모든 조회에 주입(Example 7-11):
```sql
ALTER TABLE dedp.users ENABLE ROW LEVEL SECURITY;
CREATE POLICY user_row_access ON dedp.users USING (login = current_user);
```

NoSQL 도 지원 — DynamoDB 는 IAM 정책의 `dynamodb:LeadingKeys` 로 **자기 user_id 로 시작하는 행만** 읽게 함(Example 7-12):
```json
{
 "Statement":[{
   "Sid": "...",
   "Effect":"Allow",
   "Action":["..."],
   "Resource":["arn:aws:dynamodb:us-west-1:123456789012:table/users"],
   "Condition":{
     "ForAllValues:StringEquals":{
       "dynamodb:LeadingKeys":["${www.amazon.com:user_id}"]
     }
   }
 }]
}
```

<details>
<summary><b>⚠ 트러블 로그</b> — 컬럼 마스킹만 걸고 row-level을 빠뜨리면 다른 사용자의 행이 통째로 노출됨.</summary>
<div markdown="1">

**예 —** `ip` 컬럼에 마스킹 함수를 걸어 안심했지만 `ENABLE ROW LEVEL SECURITY` 를 안 걸면,
분석가가 `SELECT visit_id, city FROM visits` 로 **모든 사용자의 방문 행** 을 그대로 조회 — 컬럼 하나만 가려졌을 뿐 행 필터는 없음.
또 `context` 같은 **nested 컬럼** 엔 컬럼 GRANT 가 안 먹어, 통째로 열리거나 통째로 막힘.

**권장 —** 컬럼 보호와 행 보호는 **별개** 임을 기억하고 둘을 함께 설계할 것. 복합 타입 컬럼은 먼저 unnest 해 평탄화한 뒤 권한을 적용.

</div>
</details>

---

### 2-2. 패턴 #44: 리소스 세분화 접근자 (Fine-Grained Accessor for Resources)

> 접근 기반 패턴은 **테이블** 데이터셋에 좋음. 그런데 DB 만 쓰는 건 아님 — 클라우드 제공자가 관리하는 다른 데이터 스토어(오브젝트 스토어·큐 등)에도 적용.

#### 상황 (Problem)

**책의 use case** — 보안 감사가 지적한 과도한 권한:

- 보안 감사가 클라우드 계정의 **과도하게 넓은 권한** 을 탐지. 위험 중 하나 — 한 데이터 처리 잡이 계정의 **모든 데이터셋을 덮어쓸** 가능성.
- 감사관이 **최소 권한(at-least privilege)** 모범 사례를 제시 — 각 컴포넌트에 **필요한 최소 권한만** 부여해, 데이터 처리 잡이 **실제 작업하는 데이터셋만** 다루게.
- **결정적 제약**: 클라우드 제공자에서 최소 권한을 **기술적으로 구현** 해야 함.

#### 해결 (Solution)

모든 주요 클라우드(AWS·Azure·GCP)가 최소 권한 구현을 제공 = 이 패턴의 backbone. **두 전략**.

- **① resource 기반** — 접근 스코프를 **리소스 수준에서 직접** 정의.
  (Example 7-13 — GCS 버킷에 IAM 정책 부착, Terraform)
  ```hcl
  data "google_iam_policy" "admin_access" {
    binding {
      role    = "roles/storage.admin"
      members = ["user:admingcs@waitingforcode.com",]
    }
  }
  resource "google_storage_bucket_iam_policy" "policy" {
    bucket      = google_storage_bucket.default.name
    policy_data = data.google_iam_policy.admin_access.policy_data
  }
  ```
- **② identity 기반** — 접근 권한을 **identity 수준**(사람 또는 애플리케이션 사용자)에서 정의. AWS 는 서비스가 assume 하는 IAM role 로 지원.
  (Example 7-14 — Spark EMR 잡이 Kinesis Data Streams 를 읽고 쓰게)
  ```hcl
  data "aws_iam_policy_document" "emr_assume_role" {
    statement {
      effect     = "Allow"
      principals { type = "Service"; identifiers = ["elasticmapreduce.amazonaws.com"] }
      actions    = ["sts:AssumeRole"]
    }
  }
  resource "aws_iam_role" "job_role" {
    name               = "visits-processor-role"
    assume_role_policy = data.aws_iam_policy_document.emr_assume_role.json
  }
  resource "aws_iam_policy" "visits_read_writer_policy" {
    name   = "visits_rw"
    policy = jsonencode({
      Version   = "2012-10-17"
      Statement = [{
        Action   = ["kinesis:Get*", "kinesis:Describe*", "kinesis:List*", "kinesis:Put*"]
        Effect   = "Allow"
        Resource = ["arn:aws:kinesis:us-east-1:1234567890:streams/visits"]}]
    })
  }
  resource "aws_iam_role_policy_attachment" "policy_attachment" {
    role       = aws_iam_role.job_role.name
    policy_arn = aws_iam_policy.visits_read_writer_policy.arn
  }
  ```

fine-grained 권한은 유연 — 특정 리소스, 같은 **prefix 로 시작하는 리소스 집합**, 심지어 **runtime 조건 기반** 리소스도 타깃 가능.
tag 기반도 가능(Example 7-15 — S3 에서 `aws:TagKeys` 로 `user_id` 태그가 붙은 리소스만 PutObject):
```json
"Statement": [{
 "Effect": "Allow", "Action": "s3:PutObject",
 "Resource": "*", "Condition": {
  "ForAllValues:StringEquals": {"aws:TagKeys": ["${www.amazon.com:user_id}"]}}
}]
```

```
[Figure 7-4 재현] resource 기반 vs identity 기반 접근 제어
──────────────────────────────────────────────────────────────────────
 resource 기반   User A ──► [ Object store ]    ← 접근 정책이 "리소스"에 붙음
                            (admin: user A / reader: group B·user C / writer: user B)

 identity 기반   User A ──► [ Database A ]       ← 접근 정책이 "사용자"에 붙음
                 (User A 정책: admin: database A / reader: message queue B / writer: object store C)
──────────────────────────────────────────────────────────────────────
 resource 기반 = 정책을 각 리소스에 부착 / identity 기반 = 정책을 각 사용자에 부착.
```

#### 고려사항 (Consequences)

IaC 나 커스텀 스크립트로 정의하니 기술적으론 어렵지 않음. 하지만 트레이드오프가 있음.

- **Security by the book trade-off(원칙과 현실의 트레이드오프)**
  - 최소 권한 원칙은 훌륭하나, **많은 작은 접근 정책** 을 낳아 복잡한 환경에서 유지보수가 어려움.
  - 완화 — **wildcard/prefix 기반**(예: `visits*`)으로 리소스 수를 관리 가능하게. 단 이는 최소 권한 위반 소지 — **미래에 생길** visits-prefixed 리소스 접근까지 허용될 수 있음. 보안팀과 논의 필요.
- **Complexity(복잡도)**
  - resource 기반과 identity 기반을 한 프로젝트에 **둘 다** 쓰면 복잡도 증가. 가능하면 use case 를 더 많이 커버하는 **한 방식** 을 선호.
- **Quotas(쿼터)**
  - 접근 정책에도 한도가 있음. 예: AWS IAM 기본 커스텀 정책 **1,500개**, GCP IAM 프로젝트당 커스텀 role **300개**. 일부는 유연해 제공자에 상향 요청 가능.

#### 구현 예시 (Examples)

**예시 — 권한 없는 접근의 예외와 두 부여 방식 (Example 7-16 ~ 7-18)**

권한 없이 S3 버킷을 읽으면 예외(Example 7-16):
```bash
$ aws s3 ls s3://dedp-visits-301JQN/
# An error occurred (AccessDenied) when calling the
# ListObjectsV2 operation: Access Denied
```

identity 기반 — 역할에 S3 읽기 액션을 스코프(Example 7-17):
```json
{"Version": "2012-10-17", "Statement": [{"Sid": "VisitsS3Reader",
 "Effect": "Allow", "Action": ["s3:Get*", "s3:List*"],
 "Resource": ["arn:aws:s3:::dedp-visits-301JQN/*",
   "arn:aws:s3:::dedp-visits-301JQN"]
}]}
```

resource 기반 대안 — 버킷 정책으로 특정 user 를 인가(Example 7-18):
```bash
$ aws s3api put-bucket-policy --bucket dedp-visits-301JQN --policy file://policy.json
# policy.json
{"Statement": [{"Effect": "Allow",
  "Principal": {"AWS": "arn:aws:iam::123456789012:user/visits-s3-reader"},
  "Action": ["s3:Get*", "s3:List*"],
  "Resource": "arn:aws:s3:::dedp-visits-301JQN/*"
}]}
```

<details>
<summary><b>⚠ 트러블 로그</b> — 유지보수 편하려고 wildcard 권한을 넓게 주면 최소 권한이 무너짐.</summary>
<div markdown="1">

**예 —** 정책 개수를 줄이려 `arn:aws:s3:::visits*` 처럼 prefix 로 묶으면, 나중에 만든 `visits-raw-pii` 버킷까지 **자동으로 읽기 가능** 해져 감사에서 지적받음.
반대로 resource 기반과 identity 기반을 한 계정에 섞어 쓰면, 같은 버킷에 정책이 두 군데 붙어 **어느 쪽이 거부인지** 추적이 어려움.

**권장 —** wildcard 는 미래 리소스까지 포함됨을 전제로 **보안팀 승인 후** 최소 범위로만 쓰고, resource·identity 중 **한 방식** 으로 통일할 것. 정책 쿼터(AWS 1,500 / GCP 300)도 사전에 확인.

</div>
</details>

---

## 3. 데이터 보호 (Data Protection)

데이터 접근을 **논리 수준**(DB·클라우드 서비스 스코프)에서 통제하는 것만으로는 완전히 안전한 시스템이 못 됨.
빠진 조각은 **데이터 그 자체를 보호** 해 예상치 못한 사용에 대비하는 것.

- **#45 Encryptor** — 저장(at rest)·전송(in transit) 데이터를 **암호화**. 접근 통제가 뚫려도 **복호화 키** 없이는 못 읽게 함. (아래 3-1)
- **#46 Anonymizer** — 민감 데이터를 **제거·변형** 해 재식별 불가능하게(제거·교란·합성 대체). 최강 보호지만 분석 가치를 훼손. (아래 3-2)
- **#47 Pseudo-Anonymizer** — 민감 값을 **쓸모 있는 가짜 값** 으로 대체(마스킹·토큰화·해싱·암호화). 분석은 가능하나 조합 재식별 위험. (아래 3-3)

```
[7.3 데이터 보호 — 세 패턴의 위치]
──────────────────────────────────────────────────────────────────────
 #45 Encryptor           원본 그대로 저장하되 "키 없으면 못 읽음"      ─► 조직 안에서 데이터를 지킴
 #46 Anonymizer          민감 값을 지우거나 딴 값으로 → 재식별 불가     ─► 밖으로 내보낼 때(데이터 공유)
 #47 Pseudo-Anonymizer   민감 값을 "쓸 수 있는 가짜 값" 으로            ─► 밖으로 내보내되 분석은 가능하게
──────────────────────────────────────────────────────────────────────
 #45 는 키를 가진 사람에겐 원본이 그대로 보이지만, #46·#47 은 원본 자체를 바꿔 내보냄.
 보호 강도 #46 > #47 > #45(공유 맥락) / 분석 활용도 #45 > #47 > #46 — 정확히 반대 방향.
```

---

### 3-1. 패턴 #45: 암호화기 (Encryptor)

> 클라우드에서 잡을 돌려도 데이터는 **물리적으로 어딘가 저장** 되고, 인가되지 않은 사람이 읽으려 할 수 있음.
> 접근 정책(#43·#44)에 더해, **접근 통제가 뚫려도 데이터를 못 쓰게** 만드는 계층이 암호화.

#### 상황 (Problem)

**책의 use case** — 저장·전송 데이터 보안을 강제하라는 과제:

- 테이블·클라우드 리소스에 fine-grained 접근 정책(#43·#44)을 구현한 뒤, **저장(at rest)·전송(in transit) 데이터 보안** 을 강제하는 과제를 받음.
- 이해관계자 우려 — 인가되지 않은 사람이 **스트리밍 브로커와 잡 사이 전송 데이터** 를 가로채거나, **서버에서 데이터를 물리적으로** 훔칠 수 있음.
- **결정적 제약**: 접근 통제가 뚫려도 데이터가 읽히면 안 됨 → 데이터 자체를 암호화.

#### 해결 (Solution)

**Encryptor** — 두 보호 수준이 필요하니 **두 구현**.

- **① 저장 데이터(at rest)** — **client-side** 또는 **server-side** 암호화. 차이는 **암호화 키 관리** 주체.
  - **client-side** — 데이터 프로듀서가 저장 전 암호화하고 **키 관리도 책임**.
  - **server-side** — 모든 암/복호화를 서버가 수행. 컨슈머·프로듀서는 요청만 보내고 **키 관리 포함 전부** 서버가 처리.
    공개 클라우드가 널리 지원 — **AWS·GCP는 KMS(Key Management Service), Azure는 Key Vault**.
- **② 전송 데이터(in transit)** — 클라이언트↔저장소가 네트워크로 데이터를 교환하는 계층.
  클라우드 구현은 비교적 쉬움 — 클라이언트 **SDK 수준에서 보안 통신** 활성화 + 서비스에 필요한 **프로토콜 버전(TLS)** 설정.

```
[Figure 7-5 재현] 저장 데이터 server-side 암호화 워크플로
──────────────────────────────────────────────────────────────────────
                          ┌─ Encryption key store ─┐
                     ③ 키 │  (KMS / Key Vault)      │
                    ┌────►└──────────▲──────────────┘
                    │            ② 키 요청│
        ① 요청       │                  │
 [ Client ] ─────────┴──► [ Encryptable data store ]
      ▲                              │
      └──────── ④ 복호화된 데이터 ─────┘
──────────────────────────────────────────────────────────────────────
 ① 요청이 암호화 저장소에 도달 → ② 저장소가 키 저장소에 복호화 키 요청(인가 안 되면 실패)
 → ③ 키 획득 → ④ 복호화한 레코드를 클라이언트에 반환. 클라우드에선 이 교환이 완전히 추상화됨.
```

#### 고려사항 (Consequences)

물리 저장 수준의 보안이라 **공짜는 아님**.

- **Encryption/decryption overhead(암/복호화 오버헤드)**
  - 데이터가 평문이 아니라 변형된 형태로 저장 → 암/복호화 없이는 못 씀. 매 읽기·쓰기가 **CPU 에 추가 부담**.
- **Data loss risk(데이터 손실 위험)**
  - 저장 데이터를 보호하지만, 부작용으로 **인가 사용자 접근도 차단** 될 수 있음 — **암호화 키를 잃거나** 키 접근을 잃으면.
  - 완화 — 클라우드는 암호화 저장소에 **soft delete** 를 구현. 삭제 요청이 즉시 반영되지 않고 **유예 기간** 동안 실수를 복원 가능.
- **Protocol updates(프로토콜 갱신)**
  - 전송 암호화는 설정이 쉽지만 **최신 유지가 필요** — TLS 는 1.0·1.1 이 deprecated(보안 이슈)라 그 버전 쓰는 서비스는 업그레이드 필요. 클라우드에선 **서비스 수준에서 버전 업그레이드** 로 단순화됨.

#### 구현 예시 (Examples)

**예시 — AWS KMS 암호화 + S3 연결 + Azure TLS 강제 (Example 7-19 ~ 7-21)**

KMS 암호화 키 정의 + 다른 서비스(Lambda IAM role)에 암/복호화 권한 부여(Example 7-19):
```hcl
module "kms" {
  source                  = "terraform-aws-modules/kms/aws"
  key_usage               = "ENCRYPT_DECRYPT"
  deletion_window_in_days = 14                       # data loss 완화용 키 복원 창(soft delete)
  aliases                 = ["visits-bucket-encryption-key"]
  grants = {
    lambda_doc_convert = {
      grantee_principal = aws_iam_role.iam_key_reader.arn
      operations        = ["Encrypt", "Decrypt", "GenerateDataKey"]
    }
  }
}
```

정의한 키를 S3 버킷에 연결 — 기본 server-side 암호화(Example 7-20):
```hcl
resource "aws_s3_bucket_server_side_encryption_configuration" "visits" {
  bucket = aws_s3_bucket.visits.id
  rule {
    apply_server_side_encryption_by_default {
      kms_master_key_id = module.kms.key_arn
      sse_algorithm     = "aws:kms"
    }
  }
}
```

전송 암호화 — Azure Event Hubs 에 **최소 TLS 버전** 강제(Example 7-21):
```hcl
resource "azurerm_eventhub_namespace" "visits" {
  name                = "visits-namespace"
  location            = azurerm_resource_group.dedp.location
  resource_group_name = azurerm_resource_group.dedp.name
  sku                 = "Standard"
  capacity            = 2
  minimum_tls_version = "1.2"                         # 구버전 TLS 클라이언트 차단
}
```

> 요약하면 암호화는 **키를 암호화 대상 리소스·인가 identity 와 연결** 하는 일. 저장은 KMS 키를 버킷에 붙이고, 전송은 서비스에 최소 TLS 버전을 설정.

<details>
<summary><b>⚠ 트러블 로그</b> — 암호화 키를 성급히 삭제하거나 구버전 TLS를 방치하면 데이터를 못 읽거나 전송이 가로채짐.</summary>
<div markdown="1">

**예 —** 정리하겠다고 KMS 키를 즉시 삭제하면, 그 키로 암호화된 S3 객체 전체가 **영구히 복호화 불가**
— soft delete 유예 없이 지우면 인가 사용자도 못 읽음.
반대로 Event Hubs 의 `minimum_tls_version` 을 안 걸어 TLS 1.0 클라이언트를 허용하면, 스트리밍 구간에서 데이터가 평문에 가깝게 **가로채질** 수 있음.

**권장 —** 키에는 반드시 **삭제 유예 창(`deletion_window_in_days`)** 을 두고, 전송에는 **`minimum_tls_version = "1.2"` 이상** 을 강제할 것. 키 접근권을 잃으면 데이터도 잃는다는 점을 운영 문서에 명시.

</div>
</details>

---

### 3-2. 패턴 #46: 익명화기 (Anonymizer)

> 챕터 6에서 봤듯 데이터셋은 **다른 파이프라인과 공유** 할 때 가치가 올라감. 그런데 늘 그렇게 간단하지는 않음.
> 데이터셋에 **PII 속성** 이 있고 사용자가 그 정보를 파트너와 공유하는 데 동의하지 않았다면, 공유 전에 **특별한 준비 단계** 가 필요.

#### 상황 (Problem)

**책의 use case** — 외부 분석 업체와의 데이터 공유 계약:

- 조직이 **외부 데이터 분석 회사** 와 계약 — 고객 행동을 분석해 커뮤니케이션 전략을 최적화하기로 함.
- 그런데 데이터셋에 **PII 속성이 많고**, 일부 사용자는 그 정보를 **제3자와 공유하는 데 동의하지 않음**.
- 데이터 엔지니어링 팀에 떨어진 과제 — 공유할 데이터셋을 **프라이버시 규정에 맞게** 만드는 파이프라인 작성.
- **결정적 제약**: 데이터셋의 **일부는 공유될 수 없음** ⇒ 그 부분을 **제거하거나 변형** 해야 함.

> **참고 사항 — PII 만은 아니다 (Not Only PII)**
> 예시가 PII 를 다루는 건 가장 흔히 논의되는 use case 이기 때문이지, 보호가 필요한 데이터 타입이 그것뿐이라서가 아님.
> **PHI**(보호 대상 건강 정보)·**IP**(지식 재산) 데이터도 같은 대상. 이후 설명은 단순화를 위해 PII 로 통일.

#### 해결 (Solution)

**Anonymizer** — 데이터셋에서 **민감 데이터를 제거해 각 행을 익명 정보로** 바꿈.
이 과정을 거치면 컨슈머 (분석 업체)는 **사용자를 식별할 수 없음**. 구현은 세 가지.

- **① Data removal (데이터 제거)** — 선택한 컬럼을 입력 데이터셋에서 **빼냄**. 구현이 가장 쉬움.
- **② Data perturbation (데이터 교란)** — 입력값에 **노이즈를 더해** 값의 의미를 바꿈. 더 복잡.
  예: IP 컬럼에서 임의 위치에 자릿수를 추가 — `123.456.789.012` → `1823.456.7809.012`.
- **③ Synthetic data replacement (합성 데이터 대체)** — 원본 값을 **합성 데이터 생성기** 가 만든 값으로 치환.
  입력 컬럼의 **타입을 해석** 해 대응하는 대체값을 만들어 내는 **똑똑한 모델**(대개 머신러닝 모델)이 필요.
  대체값은 원본 속성과 **같아 보이지만 값은 다름**. 예: country 컬럼에서 `Portugal` → `Croatia`.

구현 난이도는 **①·② 가 가장 쉬움** — **매핑 함수** 나 **컬럼 변환** 으로 값을 지우거나 교란하면 됨.
**③ 합성 데이터** 는 분석·생성 모델을 만들어야 해서 **데이터 사이언스 팀과의 협업** 이 필요할 수 있음.
덜 이상적인 버전으로는, 컬럼마다 **랜덤 값을 만드는 대체 함수** 를 직접 구현하는 방법도 있음.

```
[Anonymizer — 세 구현의 결과 비교 (같은 입력 행)]
──────────────────────────────────────────────────────────────────────
 원본             user_id | country  | ip              | birthday
                 1       | Portugal | 123.456.789.012 | 1990-03-11

 ① 제거           user_id | country  | ip              | (birthday 컬럼 자체가 없음)
                 1       | Portugal | 123.456.789.012

 ② 교란           user_id | country  | ip               | birthday
                 1       | Portugal | 1823.456.7809.012| 1990-03-11   ← 값에 노이즈 삽입

 ③ 합성 대체       user_id | country  | ip              | birthday
                 1       | Croatia  | 123.456.789.012 | 1990-03-11   ← 같은 "타입" 의 다른 값
──────────────────────────────────────────────────────────────────────
 ①은 컬럼이 사라지고, ②·③은 컬럼은 남되 값이 원본과 무관해짐 ⇒ 어느 쪽이든 재식별 불가.
 ③만 "그럴듯한 값" 을 유지해 스키마·타입 호환성이 깨지지 않음.
```

#### 고려사항 (Consequences)

민감 데이터를 확실히 보호하지만, **데이터 사용성(usability)에 큰 타격** 을 줌.

- **Information loss (정보 손실)**
  - 정보를 제거하거나 대체하면 데이터셋은 **다른 무언가가 됨**. 최종 사용자 — **데이터 분석가·데이터 사이언티스트 같은 기술 사용자 포함** — 는 그 민감 컬럼에 더 이상 의존할 수 없음.
  - 결과적으로 **잘못된 데이터 예측 모델**, **틀린 데이터 인사이트** 같은 문제로 이어질 수 있음.

#### 구현 예시 (Examples)

**예시 — birthday 컬럼 제거 + email 합성 대체 (Example 7-22)**

`birthday` 는 **제거**, `email` 은 **Faker 라이브러리** 로 만든 저품질 합성 데이터로 **대체**.
컬럼 제거는 두 가지 — `SELECT` 목록에서 **빼는 방식**(읽을 컬럼이 적을 때)과 `drop` 으로 **지우는 방식**(반대 상황).
값 대체는 컬럼 기반 함수(`withColumn`) 또는 행 기반 함수(`mapInPandas`) — 여기선 컬럼 하나만 바꾸므로 전자를 사용:

```python
@pandas_udf(StringType())
def replace_email(emails: pandas.Series) -> pandas.Series:
  faker_generator = Faker()                                    # 합성 값 생성기
  return emails.apply(lambda email: faker_generator.email())   # 원본 email 은 아예 안 씀

users.drop('birthday').withColumn('email', replace_email(users.email))
```

> 결과 데이터셋에는 `birthday` 컬럼이 없고, `email` 값은 **무작위 생성 주소** 로 바뀜.
> 대체 함수가 원본 `email` 을 **인자로 받되 사용하지 않는다** 는 점이 핵심 — 원본과의 연결 고리가 남지 않음.

<details>
<summary><b>⚠ 트러블 로그</b> — 익명화를 "컬럼 제거" 로만 처리하면 다운스트림 모델·리포트가 조용히 망가짐.</summary>
<div markdown="1">

**예 —** 공유 데이터셋에서 `birthday` 를 drop 했더니, 그 컬럼으로 연령대 세그먼트를 만들던 분석 업체의 리포트가
전부 `null` 세그먼트로 떨어짐. 에러 없이 **숫자만 조용히 틀어져** 두 달 뒤 캠페인 성과 리뷰에서야 발견됨.

**반대 함정 —** 급한 대로 `email` 을 Faker 로 대체했는데, 배치마다 **같은 사용자에게 다른 가짜 email** 이 붙어
업체 쪽에서 사용자 단위 집계(고유 사용자 수)가 배치 수만큼 부풀려짐. 익명화는 **원본과의 연결 고리를 끊는 것** 이 목적이라
사용자 단위 집계가 필요하면 애초에 #47 Pseudo-Anonymizer 를 써야 하는 상황.

**권장 —** 익명화 대상 컬럼을 정하기 전에 **컨슈머 (분석 업체)가 그 컬럼으로 무엇을 계산하는지** 를 먼저 확인하고,
사용자 단위 집계가 필요하면 #46 이 아니라 **#47** 을 선택할 것.

</div>
</details>

---

### 3-3. 패턴 #47: 의사 익명화기 (Pseudo-Anonymizer)

> #46 Anonymizer 는 강력한 보호를 제공하지만, **값이 사라지거나 바뀌어** 데이터 사이언스·분석 파이프라인에 매우 나쁜 영향을 줄 수 있음.
> Pseudo-Anonymizer 는 그 충격을 **줄이는** 패턴.

#### 상황 (Problem)

**책의 use case** — 익명화된 데이터셋이 쓸모없다는 항의:

- 외부 분석 업체와 공유한 **익명화 데이터셋에 컬럼이 다 들어 있지 않음**. 가장 단순한 익명화 전략이라 **제거** 를 택했기 때문.
- 그 탓에 업체 팀이 **대부분의 비즈니스 쿼리에 답하지 못함**.
- 팀의 요청 — **실제 PII 값은 계속 숨기되**, **더 쓸모 있는 형태의 값으로 대체한** 새 데이터셋을 달라.
- **결정적 제약**: 데이터 공유 + PII + **비즈니스 의미는 어느 정도 남겨야 함** — 이 세 조건이 겹치는 자리.

#### 해결 (Solution)

**Pseudo-Anonymizer** — 원본 값을 **원본과 다소간 관련 있는 다른 값** 으로 대체. 네 가지 구현.

- **① Data masking (마스킹)** — 민감 데이터를 **의미 없는 문자** 나 더 현실적인 대체값으로 교체.
  예: SSN `999-55-1040` → `XXX-XX-1040` 또는 `9XX-5X-1XXX`.
  이 전략을 쓰면 **서로 다른 여러 사용자가 같은 마스킹 값을 공유** 할 수 있음.
- **② Data tokenization (토큰화)** — 초기 값을 **가공의 값** 으로 치환하되, **원본↔대체값 매핑을 token vault store 에 저장**.
  핵심은 **vault 접근을 안전하게 지키는 것** — vault 보안이 뚫리면 인가되지 않은 사람이 **토큰화 값을 역변환** 할 수 있음.
- **③ Hashing (해싱)** — 민감 값을 **완전히·비가역적으로** 대체.
  예: `contact@waitingforcode.com` → SHA-256 해시 + Base64 인코딩 → `gD0B+pUpXYVZ9nqhgLRuban0CilZRKVp4dcmvmocsYE=`.
- **④ Encryption (암호화)** — **컬럼 또는 행에 암호화 키** 를 적용. #45 Encryptor 의 데이터셋 키와 비슷한 방식.
  차이는 **암호화 키에 접근할 수 있는 사용자는 원본 값을 복원할 수 있다** 는 점.

데이터 타입에 맞는 방법을 고른 뒤 익명화 함수를 구현. **마스킹** 처럼 단순한 **컬럼 변환 함수** 로 표현되는 것도 있고,
**토큰화** 처럼 **별도 매핑 테이블** 이 필요해 추가 구현 노력이 드는 것도 있음.

```
[네 구현의 되돌릴 수 있음(reversibility) 축]
──────────────────────────────────────────────────────────────────────
 ③ Hashing        비가역            되돌릴 방법 없음                 ─► 조인 키로는 쓸 수 있음(같은 입력 = 같은 해시)
 ① Masking        비가역(손실)       원본 복원 불가 + 값 충돌 발생      ─► 여러 사용자가 같은 값을 가질 수 있음
 ② Tokenization   가역(vault 필요)  vault 를 가진 쪽만 복원           ─► vault 가 뚫리면 전부 노출 ⚠
 ④ Encryption     가역(키 필요)      키를 가진 쪽만 복원               ─► 키 관리는 #45 Encryptor 와 동일 문제
──────────────────────────────────────────────────────────────────────
 "복원 가능성" 이 곧 "관리해야 할 비밀(vault·키)이 하나 더 생김" 을 뜻함 — ②·④는 그 비밀이 새 공격면.
```

> **참고 사항 — 익명화 vs 의사 익명화 (Anonymization Versus Pseudo-Anonymization)**
> 이 패턴의 의사 익명화 기법들은 익명화 과정의 일부로 소개되기도 하지만, 둘 사이엔 **중요한 차이** 가 있음.
> 의사 익명화된 PII 데이터는 **다른 데이터셋과 결합하면 식별 가능해질 수 있음**.
> 익명화된 데이터셋은 그렇지 않음 — **결합하더라도 식별 불가능한 상태로 남음**.

#### 고려사항 (Consequences)

개인정보를 보호하긴 하지만, **#46 Anonymizer 만큼 강한 보호 보장을 주지는 못함**.

- **False sense of security (거짓 안전감)**
  - 의사 익명화는 개인정보를 흐릴 뿐 **#46 보다 약한 보안 보장**. 가장 큰 문제는 **데이터셋 결합** — 의사 익명화된 컬럼이 **개인을 식별하는 PII 컬럼으로 되살아날 수 있음**.
  - 완전 익명화였다면 식별이 불가능했을 것 — Country·Role 컬럼이 아예 제거되거나 `Europe`·`Software engineer` 처럼 **일반화** 됐을 테니, 두 표를 합쳐도 이 소프트웨어 엔지니어가 누구인지 알 수 없음.
- **Information loss (정보 손실)**
  - **마스킹** 이 대표 사례 — SSN 일부 숫자가 보존되므로 **서로 다른 두 SSN 이 같은 사용자를 가리키게** 됨.
    예: `999-55-1040` 과 `999-13-1040` 이 둘 다 `XXX-XX-1040` 으로 마스킹됨.
  - 여기에 더해 **데이터 타입 손실** 도 있음. **일반화(generalization)** 가 좋은 예 — **숫자 값** 이 **텍스트 타입으로 표현된 숫자 범위** 로 바뀜.

```
[Table 7-1 / 7-2 재현] 결합으로 무너지는 의사 익명화
──────────────────────────────────────────────────────────────────────
 [Table 7-1] 음식 선호 테이블 — 그 자체로는 신원이 완벽히 보호됨
 User ID | Liked foods              | Disliked foods
 1000    | carrot, broccoli, potato | chips, chocolate bar

 [Table 7-2] 사용자 등록 테이블 — 마스킹돼 있음
 User ID | Country   | Role
 1000    | S*n M****o | C******h P*******r i******r
──────────────────────────────────────────────────────────────────────
 ⇒ 두 표를 user_id 로 결합하면:
   "S*n M****o 에 사는 C******h P*******r i******r 가 carrot·broccoli·potato 를 좋아함"
   ✗ 이름이 저 마스킹 패턴에 맞는 나라는 거의 없음 ⇒ San Marino 로 특정
     ⇒ "inventor" 부분까지 풀려 user_id 1000 = John Doe(Cheetach Processor 개발자) 로 재식별
   ✓ 완전 익명화였다면 Country/Role 이 제거되거나 "Europe"·"Software engineer" 로 일반화돼 결합해도 식별 불가
```

#### 구현 예시 (Examples)

**예시 — PySpark `mapInPandas` + 컬럼 변환 (Example 7-23 ~ 7-26)**

의사 익명화 구현은 **매핑 함수** 에 의존. 대상 테이블은 다음과 같음(Example 7-23):

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

`user_id` 를 제외한 나머지 컬럼을 의사 익명화 — **두 가지 방법** 을 함께 씀.
먼저 `mapInPandas` 로 country 를 **지리 권역으로 일반화**, ssn 을 **마스킹**(Example 7-24):

```python
def pseudo_anonymize_users(input_pandas: pandas.DataFrame) -> pandas.DataFrame:
  def pseudo_anonymize_country(country: str) -> str:
    countries_area_mapping = {
    'Poland': 'eu', 'France': 'eu', 'Spain': 'eu', 'the USA': 'na'   # 일반화 매핑
    }
    return countries_area_mapping[country]

  def pseudo_anonymize_ssn(ssn: str) -> str:
    return f'{ssn[0]}***-{ssn[5]}***-{ssn[10]}***'                   # 각 블록 첫 자리만 보존

  for rows in input_pandas:
    rows['country'] = rows['country'].apply(lambda c: pseudo_anonymize_country(c))
    rows['ssn'] = rows['ssn'].apply(lambda ssn: pseudo_anonymize_ssn(ssn))
    yield rows
```

`salary` 는 `mapInPandas` 매핑에 넣지 않음 — **타입이 바뀌기 때문**.
입력은 integer 인데 범위는 string 이라 **타입 비호환 변환** 이므로, 단순한 **컬럼 기반 매핑** 이 따로 필요(Example 7-25):

```python
pseud_anonymized_users = (users.mapInPandas(pseudo_anonymize_users, users.schema)
  .withColumn('salary', functions.expr('''
    CASE WHEN salary BETWEEN 0 AND 50000 THEN "0-50000"
         WHEN salary BETWEEN 50000 AND 60000 THEN "50000-60000"
         ELSE "60000+" END''')))                     # int → string 범위로 일반화
```

두 변환을 모두 적용한 결과(Example 7-26):

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

> `user_id` 1·2 의 마스킹된 ssn 이 **`0***-0***-1***` 로 완전히 같아짐** — 위 "Information loss" 가 실제로 드러난 자리.
> `mapInPandas` 는 **스키마를 그대로 유지** 해야 하므로(`users.schema` 전달) 타입이 바뀌는 변환은 `withColumn` 으로 분리해야 함.

```
[Examples 7-23 ~ 7-26 한 흐름 — 컬럼마다 다른 변환 경로]
──────────────────────────────────────────────────────────────────────
 users --> mapInPandas(pseudo_anonymize_users, users.schema)  <- ① country · ssn 처리 (타입 유지)
       --> withColumn('salary', CASE WHEN ... END)            <- ② salary 처리 (타입 변경)
       --> Example 7-26 result                                <- 최종 결과

 column  in      via          out               type
 ------------------------------------------------------
 user_id STRING  (none)       1                 STRING    <- 변환 없음 — 조인 키로 남김
 country STRING  mapInPandas  eu                STRING    <- 일반화 (타입 유지 O)
 ssn     STRING  mapInPandas  0***-0***-1***    STRING    <- 마스킹 (타입 유지 O)
 salary  INT     withColumn   "0-50000"         STRING    <- 일반화 + 타입 변경 INT -> STRING
──────────────────────────────────────────────────────────────────────
 갈림길은 "타입이 바뀌는가" 딱 하나 —
 mapInPandas 는 users.schema 를 그대로 넘겨받아 출력 스키마를 유지해야 하므로 INT → STRING 을 못 함
 ⇒ salary 만 mapInPandas 밖으로 빠져나와 withColumn 으로 처리.
 ⚠ salary 를 pseudo_anonymize_users 안에 넣으면 스키마 불일치로 런타임에 실패.
```

<details>
<summary><b>⚠ 트러블 로그</b> — 의사 익명화한 컬럼을 여러 테이블에 흩어 두면 조인 한 번으로 재식별됨.</summary>
<div markdown="1">

**예 —** `users` 테이블의 `country` 를 `eu`/`na` 로 일반화해 안심했지만, 같은 데이터마트의 `user_profile` 테이블엔
`city` 컬럼이 마스킹 없이 남아 있었음. 두 테이블을 `user_id` 로 조인하면 city 하나로 국가가 그대로 복원돼
일반화가 무의미해짐 — 책의 San Marino 예시가 실무에서 나타나는 형태.

**반대 함정 —** 해싱만 믿고 `email` 을 SHA-256 으로 바꿨는데, **salt 없이** 해싱하면 흔한 도메인의 주소는
레인보우 테이블로 역추적 가능. 해싱은 "비가역" 이지만 **입력 공간이 좁으면 사실상 가역**.

**권장 —** 의사 익명화는 **컬럼 단위가 아니라 데이터셋 조합 단위** 로 재식별 위험을 평가할 것.
공유 범위 안의 **모든 테이블을 함께 놓고** 조인 가능한 준식별자(city·zip·생년·직함)를 같이 처리하고, 해싱에는 반드시 salt 를 적용.

</div>
</details>

---

## 4. 연결성 (Connectivity)

여기까지가 **데이터를 보호하는 법**. 데이터 보안의 핵심이지만 이것만으로는 부족할 수 있음.
챕터 6의 데이터 흐름 패턴을 떠올려 보면 — 데이터는 **같은 시스템 안에서, 또 시스템 사이를 계속 흐르고**, 그 데이터에 **접근** 해야 함.
이 절은 그 **안전한 접근 전략** 을 다룸.

- **#48 Secrets Pointer** — 자격 증명을 코드/Git 에 두지 않고 **시크릿 매니저의 이름(참조)** 으로 가리킴. (아래 4-1)
- **#49 Secretless Connector** — 아예 **관리할 자격 증명 없이** IAM·인증서 기반으로 연결. (아래 4-2)

```
[연결성 — 자격 증명을 어디에 두나]
──────────────────────────────────────────────────────────────────────────────────────
 ✗ 나쁜 기본값           코드/Git 에 login·password 직접 기재   ─► 유출 시 그대로 사용 가능
 #48 Secrets Pointer  코드엔 "이름" 만, 값은 시크릿 매니저       ─► 런타임에 조회, 관리할 비밀은 남음
 #49 Secretless       비밀 자체가 없음(IAM 신원·인증서)         ─► 관리할 비밀 0, 대신 설정·로테이션 부담
──────────────────────────────────────────────────────────────────────────────────────
 #48 → #49 로 갈수록 "관리해야 할 비밀의 개수" 가 줄어듦. 다만 #49 도 "일이 없다" 는 뜻은 아님.
```

---

### 4-1. 패턴 #48: 비밀 포인터 (Secrets Pointer)

> **login/password 인증** 은 여전히 DB 접근에 가장 흔히 쓰이는 방식. 간단하지만 **주의 없이 쓰면 위험** 하고,
> 이 패턴이 그 주의책 중 하나.

#### 상황 (Problem)

**책의 use case** — 과거 자격 증명 유출로 청구액이 튄 경험:

- visits 실시간 처리 파이프라인이 **외부 API** 로 각 이벤트를 **지오로케이션 정보** 로 보강. 외부 회사가 제공하며 **유일한 인증 수단이 login/password 쌍**.
- 과거에 팀이 **다른 API 의 login/password 를 실수로 공유** 한 적이 있음. 그 API 는 **요청 기반 과금** 이라 유출이 곧 **청구액 증가** 로 이어짐.
- **결정적 제약**: 지금 당장 이 위험을 줄여야 하며, 새 데이터 보강 API 와 상호작용하는 **코드에 login/password 를 저장하지 않아야** 함.

#### 해결 (Solution)

자격 증명은 민감한 파라미터. 지키는 가장 좋은 방법 중 하나는 **어디에도 저장하지 않는 것**
— 대신 **참조(reference, 곧 포인터)** 를 씀. 그것이 **Secrets Pointer**.

- **시크릿 매니저 서비스 활용** — Google Cloud Secret Manager·AWS Secrets Manager 등에 로그인·패스워드·API 키 같은 민감 값을 **전부 저장**.
  - **중앙 집중** — 민감 데이터를 한 곳에서 관리하므로 **접근 모니터링이 쉬움**.
  - **관리 용이** — **모든 컨슈머를 갱신하지 않고도** 새 자격 증명 세트를 설정할 수 있음.
- **컨슈머 (연결하는 쪽) 의 동작** — 더 이상 민감 파라미터의 **값** 을 참조하지 않고, 시크릿 매니저의 **이름** 을 참조.
  - 각 컨슈머는 **런타임에 서비스로 쿼리를 날려** 시크릿 값을 가져옴.
  - 통신 비용 절약을 위해 자격 증명을 **로컬 캐시에 일정 시간 보관** 할 수 있음.
- **두 수준의 보호** — 접근이 두 겹으로 막힘.
  - **1단계** — 컨슈머가 **시크릿 매니저 접근권** 을 가져야 함. 없으면 키 자체를 못 받아옴.
    이 단계는 앞의 **fine-grained access 패턴(#43·#44)** 으로 지킬 수 있음.
  - **2단계** — 자격 증명 자체가 네이티브하게 보장. 유효하지 않으면 컨슈머는 하위 API·DB 에 접근하지 못함.

```
[Secrets Pointer — 코드에는 이름만, 값은 런타임에]
──────────────────────────────────────────────────────────────────────
 [ 코드 / Git ]   properties = {user: SecretId('user'), password: SecretId('pwd')}
                                        │  ① 이름으로 조회 (런타임)
                                        ▼
                               [ Secrets manager ]  ← 1단계 보호: 접근권 없으면 여기서 막힘 (#43·#44)
                                        │  ② 값 반환 (선택적으로 로컬 캐시)
                                        ▼
                            [ PostgreSQL / 외부 API ]  ← 2단계 보호: 값이 유효하지 않으면 여기서 막힘
──────────────────────────────────────────────────────────────────────
 코드는 값을 "모른 채" 동작 — 정확히는 시크릿 매니저 덕분에만 알게 됨.
 ⇒ 자격 증명이 코드베이스에 묶이지 않고 별도 자산으로 관리됨.
```

#### 고려사항 (Consequences)

Phil Karlton 의 유명한 말 — *"컴퓨터 과학에 어려운 것은 둘뿐이다: 캐시 무효화와 이름 짓기."*
그중 **캐시** 부분이 Secrets Pointer 에도 그대로 적용됨.

- **Cache invalidation and streaming jobs (캐시 무효화와 스트리밍 잡)**
  - 자격 증명을 캐시하면 **지금 쓰는 게 최신인지 알 수 없어** 연결 문제로 이어질 수 있음.
    반대로 조회 요청이 비싸다면 **요청을 많이 줄여 실행 시간을 최적화** 할 수 있음 — 캐시를 구현한다면 **갱신 수단도 함께** 갖춰야 함.
  - **가장 단순한 접근** — 갱신 요청을 보내고 싶지 않다면 **그냥 잡을 실패시키는 것**.
    재시작하면 새 실행 버전이 시크릿 매니저에서 자격 증명을 다시 로드함.
    단 자격 증명이 **자주 바뀌면 실패 횟수가 늘어** 최적이 아님. 또 재시도에도 데이터가 정확하게 유지되도록 **멱등성 패턴(챕터 4)** 을 함께 쓸 것.
  - **비동기 갱신 프로세스** 도 시도할 수 있음. 다만 **출력 데이터 스토어로 데이터를 보내기 시작한 뒤, 연결 파라미터를 갱신하기 전에** 자격 증명이 바뀌면 쓰기 문제가 생길 수 있음.

```
[캐시 무효화 — 로테이션 시점을 시간축에 놓고 보기]
──────────────────────────────────────────────────────────────────────
 t0         t10         t30         t31         t40     <- 시간축 (t30 에 로테이션 발생)
 pwd_A                  pwd_B                           <- 시크릿 값 (pwd_A -> pwd_B)
                        |
 [run][end] [run][end]  |  [run][end] [run][end]        <- 배치 잡 — 매 실행 시작마다 재조회
                        |                                  ✓ t30 이후 실행이 pwd_B 를 자연히 로드
                        |
 [run ==================+==========================>    <- 스트리밍 잡 — t0 에 한 번 조회 후 캐시
                        |                                  ✗ 캐시가 갱신될 계기가 없음
                        +--> 다음 연결에서 인증 실패
──────────────────────────────────────────────────────────────────────
 배치는 "재시작" 이 자연히 일어나 캐시가 저절로 무효화되지만, 스트리밍은 그 계기가 없음
 ⇒ 같은 코드가 배치에선 멀쩡하고 스트리밍에서만 터지는 이유.

 [대응 세 갈래 — 무엇을 대가로 내는가]
   ① 그냥 실패시키기 — 재시작 시 새 실행이 재조회. 구현 비용 0 / 자격 증명이 자주 바뀌면 실패 폭증
      ⚠ 재시도로 데이터가 깨지지 않게 멱등성 패턴(챕터 4) 병행 필수
   ② 캐시 TTL 을 짧게 — 로테이션 주기보다 짧게 잡음. 무중단 / 조회 요청·비용 증가
   ③ 비동기 갱신 — 백그라운드로 재조회. 무중단 / 쓰기 도중 교체되면 실패 가능
      ⚠ 위험 구간 — 출력 데이터 스토어로 전송을 시작한 뒤, 연결 파라미터를 갱신하기 전 사이
```

<details>
<summary><b>▸ 더 보기</b> — 캐시 TTL 이 무엇이고, 대응 세 갈래를 각각 언제 고르나.</summary>
<div markdown="1">

**캐시 TTL (Time To Live)** — 캐시에 담아 둔 값에 붙이는 **유효 기간**.
"이 값을 몇 초 동안 재사용해도 좋다" 는 선언으로, 그 시간이 지나면 캐시가 **만료(stale)** 로 표시돼
다음 사용 시 원본(여기서는 시크릿 매니저)에서 다시 읽어옴.

- **동작** — TTL 5분이면 5분 안에는 몇 번을 쓰든 조회 0회, 5분이 지난 뒤 첫 사용에서 1회 조회하고 다시 5분이 시작.
- **TTL = 최대 노출 창** — 로테이션이 t30 에 일어나고 TTL 이 5분이면 늦어도 t35 에는 새 값을 잡음.
  즉 **옛 값을 쓰는 최악의 시간이 TTL 로 상한** 이 잡힘.
- **기본 규칙은 `TTL < 로테이션 주기`** — 단 TTL 을 줄일수록 안전해지는 대신 조회 횟수가 늘어
  캐시를 둔 이유(비용·지연 절감)가 사라짐. 그 균형점을 찾는 것이 ②.

**이 선택지가 생기는 이유** — 시크릿 매니저 조회는 공짜가 아님. 네트워크 왕복이고 관리형 서비스는
**호출당 과금 + 레이트 리밋** 이 있음. 그래서 캐시를 두는데, 캐시하는 순간
**지금 든 값이 최신인지 알 수 없어짐** — 세 갈래는 이 간극을 메우는 방법.

**① 그냥 실패시키기** — 갱신 로직을 만들지 않고, 인증이 깨지면 잡을 죽게 둠.
재시작한 새 실행이 시크릿 매니저에서 다시 읽어오므로 자연히 해결.

- **멱등성이 왜 붙나** — 이 전략은 **실패를 정상 경로로 삼는 것** 이라 실패가 안전해야 성립.
  잡이 죽으면 마지막 체크포인트부터 재처리하며 **이미 쓴 레코드를 다시 씀** ⇒
  멱등 쓰기(`MERGE`·`UPSERT`·결정적 키)가 없으면 **인증 실패 한 번이 데이터 중복으로 번짐**.

**② 캐시 TTL 을 짧게** — 위 정의대로 노출 창을 TTL 로 상한.

- **완전한 무중단은 아님** — 로테이션 직후부터 TTL 만료까지는 여전히 옛 값을 씀.
  서버가 옛 자격 증명을 즉시 폐기하면 그 사이에 실패 ⇒
  **#49 의 겹침 구간(overlap)** 이 이 창을 덮어 줘야 안전.

**③ 비동기 갱신** — 백그라운드 스레드가 주기적으로 다시 읽어 커넥션 파라미터를 교체.

- **위험 구간이 벌어지는 순서** — 싱크에 커넥션을 열고 쓰기 시작(**옛 자격 증명으로 이미 인증된** 커넥션)
  → 백그라운드가 새 값 수신 → 진행 중인 쓰기는 여전히 옛 커넥션 위
  → 서버가 옛 자격 증명을 무효화하면 **쓰기 도중 연결이 끊겨 부분 쓰기**.
- 즉 갱신 자체는 성공했는데 **in-flight 쓰기가 끼여 깨지는** 형태.
  완화 — 파라미터 교체를 아무 때나 하지 말고 **마이크로배치 경계에서만**, 진행 중인 쓰기를 drain 한 뒤에 수행.

**어떻게 고르나** — 셋은 배타적이지 않음. 실무에서 가장 흔한 조합은 **② + ①**
— TTL 로 평상시를 덮고, 그래도 어긋나면 실패시켜 재시작에 맡김.
③은 재시작 비용이 정말 큰 장기 실행 스트리밍에서만 값을 함.

| 상황 | 선택 |
|---|---|
| 배치 잡 | 고민 불필요 — 매 실행이 재조회라 ① 이 저절로 성립 |
| 스트리밍 + 로테이션 주기가 긺 | ① 실패-재시작 (멱등성 필수) |
| 스트리밍 + 로테이션이 잦음 | ② TTL 을 로테이션 주기보다 짧게 |
| 재시작 비용이 매우 큼 | ③ 비동기 갱신 (배치 경계에서만 교체) |

**한 발 물러서면** — 이 세 갈래 전체가 **관리할 비밀이 존재하기 때문에** 생기는 문제.
**#49 Secretless Connector** 로 가면 IAM 이 임시 자격 증명을 발급하고 SDK 가 만료 전에 알아서 갱신하므로
**캐시 무효화라는 고민 자체가 사라짐** — 책이 #48 다음에 곧바로 #49 를 배치한 이유.

</div>
</details>

- **Logs (로그)**
  - 이 패턴은 **자격 증명이 유출되지 않는다는 거짓 안전감** 을 줌. 값이 안전한 곳에 있고 인가된 주체만 접근할 수 있는 건 맞지만,
    그 부분이 뚫리지 않아도 **로그에 무심코 포함하면 그대로 유출됨**.
- **A secret remains secret (비밀은 여전히 비밀)**
  - 컨슈머가 자격 증명을 직접 다루지 않는다고 해서 **자격 증명이 없는 건 아님**.
    컨슈머가 값 대신 참조를 쓰게 하려면, **시크릿 값을 안전하게 생성해 시크릿 저장소에 넣는 secrets producer** 가 있어야 함.
  - 실무에서 그 주체는 둘 — 값을 저장소에 넣는 **사람 관리자**, 또는 DB 를 만들면서 **랜덤 시크릿 값을 정의하는 IaC 스택**.

#### 구현 예시 (Examples)

**예시 — Spark 에서 PostgreSQL 을 평문 자격 증명 없이 연결 (Example 7-27)**

PostgreSQL 테이블을 읽어 JSON 파일로 변환하는 Spark 잡. 원래는 login/password 를 포함한 파라미터가 필요하지만,
평문 값 대신 **참조** 를 정의:

```python
secretsmanager_client = boto3.client('secretsmanager')
db_user = secretsmanager_client.get_secret_value(SecretId='user')['SecretString']    # 이름으로 조회
db_password = secretsmanager_client.get_secret_value(SecretId='pwd')['SecretString']
spark_session.read.option('driver', 'org.postgresql.Driver').jdbc(
  url='jdbc:postgresql:dedp', table='dedp.devices',
  properties={'user': db_user, 'password': db_password})    # 값은 런타임에만 존재
```

> 연결 설정은 여전히 user·password 를 참조하지만 **코드는 그 값을 모름**
> — 엄밀히는 알지만, **시크릿 매니저 데이터 스토어 덕분에만** 앎. 자격 증명이 **코드베이스가 아니라 별도 자산** 으로 관리됨.

또 하나의 이점 — **여러 환경(multi-environment) 작업이 단순해짐**.
Terraform 같은 IaC 로 DB 와 연결 속성을 만든다면 **모든 환경에서 같은 시크릿 이름** 을 유지하고 생성은 IaC 에 맡기면 됨.
⇒ 환경마다 다른 연결 파라미터를 담은 **환경별 설정 파일** 을 코드가 다룰 필요가 없어짐.

<details>
<summary><b>⚠ 트러블 로그</b> — 시크릿을 캐시한 스트리밍 잡이 자격 증명 로테이션 뒤에도 옛 값을 계속 쓰거나, 조회한 값이 로그로 새어 나감.</summary>
<div markdown="1">

**예 —** 30일 주기로 로테이션하는 DB 패스워드를 구동 시 한 번만 읽어 캐시한 Structured Streaming 잡이,
로테이션 당일 새벽 `FATAL: password authentication failed for user "dedp"` 로 죽음.
잡은 무기한 실행이라 **재시작이 없어 캐시가 영원히 갱신되지 않았던 것** — 배치 잡에서는 안 드러나던 문제.

**반대 함정 —** 디버깅하겠다고 `logger.info(f'connecting with {properties}')` 를 넣으면
조회해 온 **평문 패스워드가 그대로 로그 저장소** 에 남음. 시크릿 매니저로 옮긴 의미가 사라지고, 로그는 대개 접근 통제가 더 느슨함.

**권장 —** 장기 실행 스트리밍 잡은 **캐시 TTL 을 로테이션 주기보다 짧게** 두거나, 갱신 수단이 없다면
**실패 후 재시작으로 다시 로드되게** 설계할 것(멱등성 패턴 병행). 연결 파라미터 객체는 **통째로 로깅하지 말 것**.

</div>
</details>

---

### 4-2. 패턴 #49: 비밀 없는 커넥터 (Secretless Connector)

> #48 Secrets Pointer 는 자격 증명을 **지키는** 법을 보여줌. 그런데 **관리할 자격 증명이 아예 없다면 더 낫지 않을까?**
> 이 패턴이 그걸 가능하게 함.

#### 상황 (Problem)

**책의 use case** — API 키 관리를 피하고 싶은 소규모 팀:

- 조직의 한 팀이 **새 데이터 처리 서비스** 통합을 시작. 찾은 코드 예제는 전부 **API 키** 로 클라우드 관리 리소스와 상호작용.
- 팀 규모가 작아 **이 API 키들을 관리하고 싶지 않음**.
- 두 번째로 문의가 들어옴 — **코드에서 참조할 어떤 종류의 자격 증명도 없이** 리소스 접근을 보장하는 대안이 있는가.
- **결정적 제약**: 클라우드 서비스를 쓰면서 **자격 증명 자체를 관리하지 않아야** 함.

#### 해결 (Solution)

**Secretless Connector** — 두 가지 주요 접근.

- **① IAM 기반** — 클라우드 제공자의 **IAM 서비스** 활용. 사용자·관리자가 **문서 접근 정책(document access policy)** 으로 각 사용자·그룹·역할에 읽기·쓰기 액션을 할당.
  - 이 IAM 기반 정책 방식은 **애플리케이션 사용자**(데이터 처리 잡 등)에도 적용됨.
    이들은 사람 사용자와 **같은 클라우드 리소스** 를 쓰지만 상호작용이 **자동화** 돼 있음 — 예를 들어 잡이 하루 중 특정 시각에 실행되도록 스케줄됨.
  - 따라서 이들은 **로그인하지 않지만** 어떻게든 클라우드 리소스 접근을 **인가받아야** 함.
- **② 인증서 기반(certificate-based authentication)** — 워크플로는 ①과 유사하나, IAM 서비스 자리에 **CA(certificate authority)** 컴포넌트가 들어감.
  - 이 authority 가 **연결 과정에서 쓰인 인증서를 검증** 한 뒤에야 워크플로 진행을 인가.

```
[Figure 7-6 재현] 클라우드 서비스의 credentialless 접근 워크플로
──────────────────────────────────────────────────────────────────────
 [ Data processing job ]        애플리케이션 사용자 — 로그인하지 않음
        │  ▲
        1  4                    ① 오브젝트 하나를 읽겠다는 요청
        ▼  │                    ④ 응답 — 권한 충분 ⇒ 데이터 / 부족 ⇒ 에러
 [ An object store service ]    객체를 바로 반환하지 않음
        │  ▲
        2  3                    ② "이 사용자에게 필요한 권한이 다 있나?" 검증 요청
        ▼  │                    ③ 권한 스코프 목록 응답
 [ IAM service ]                Access policy: can read object store
                                               can read a message queue
──────────────────────────────────────────────────────────────────────
 잡은 로그인도 자격 증명도 없이, 오직 부여받은 "신원(identity)" 만으로 ②에서 인가됨.
 ※ 인증서 기반이면 IAM 자리에 CA(certificate authority)가 들어가 연결에 쓰인 인증서를 검증.
```

#### 고려사항 (Consequences)

이름의 **-less** 접미사가 "할 일이 없다" 는 인상을 주지만, **이 패턴도 일이 필요함**.

- **Workless impression (일이 없다는 인상)**
  - 자격 증명이 관여하지 않아도 **여전히 할 일이 있음** — 엔티티가 credentialless 접근을 쓰도록 **설정** 해야 함.
  - 예: AWS 에서는 **assume role 권한** 을 설정해, 엔티티가 **STS(Security Token Service)** 가 반환하는 **임시 자격 증명** 으로 다른 서비스와 상호작용하게 만들어야 함.
- **Rotation (로테이션)**
  - 이 항목은 본질적으로 **인증서 기반 인증** 에 해당. 접근 키를 **정기적으로 로테이션** 하는 것은 유출 위험을 줄이는 보안 모범 사례로 여겨짐.
    다만 인증서·로그인·패스워드는 **추가 관리 오버헤드** 를 낳음.
  - 컨슈머에 영향 없이 로테이션하려면 **3단계** — ① 새 자격 증명을 생성해 컨슈머에게 공유 → ② 그동안 **구/신 양쪽을 모두 지원**(동시에 마이그레이션하지 못하는 컨슈머도 계속 쓸 수 있게) → ③ 컨슈머가 신규 사용을 확인한 뒤에야 **구 자격 증명 폐기**.

```
[무중단 로테이션 — 겹침 구간(overlap)이 있어야 하는 이유]
──────────────────────────────────────────────────────────────────────
        (1)                               (3)       <- ① 새 자격 증명 생성·공유 / ③ 구 자격 증명 폐기
         |                                 |
 old  #####################################X        <- 구 자격 증명 — ③에서 끊김
 new     ########################################>  <- 신 자격 증명 — ①부터 유효
         |<----------- overlap ----------->|        <- ② 구·신 동시 지원 구간
 A         +--> new                                 <- 컨슈머 A: 즉시 전환
 B                +--> new                          <- 컨슈머 B: 며칠 뒤 전환
 C                           +--> new               <- 컨슈머 C: 마지막으로 전환
                                    |
                                    +-- 모두 전환 확인 ⇒ 그 뒤에야 ③ 실행
──────────────────────────────────────────────────────────────────────
 ✓ 겹침 구간이 있으면 — 컨슈머마다 전환 시점이 달라도 아무도 끊기지 않음.
 ✗ 겹침 없이 ①에서 바로 갈아끼우면 — 아직 전환 못 한 B·C 가 동시에 인증 실패.
 ⇒ ③의 방아쇠는 "날짜" 가 아니라 "모든 컨슈머의 신규 사용 확인" — 확인 전에 폐기하면 무중단이 깨짐.
```

<details>
<summary><b>▸ 더 보기</b> — 겹침 구간을 얼마로 잡나, ③을 언제 누르나, 그리고 인증서 만료라는 "자동 ③".</summary>
<div markdown="1">

**적용 범위 — 인증서 기반에만 해당.** IAM 기반은 이 고민이 없음.
AWS STS 가 발급하는 임시 자격 증명은 수명이 짧고 **SDK 가 만료 전에 알아서 재발급** 하므로
로테이션이 런타임에 **사람 개입 없이** 일어남. 인증서는 사람이 만들고·배포하고·폐기해야 함
— `-less` 가 "할 일이 없다" 는 뜻이 아니라는 지적이 가장 구체적으로 드러나는 자리.

**로테이션이 막는 것 —** 흔한 오해가 "로테이션하면 유출을 막는다" 인데, 그렇지 않음.
**로테이션은 유출을 막지 못하고, 유출의 유효 기간을 자름.**

- 자격 증명은 시간이 갈수록 노출 확률이 **누적** 됨 — 디버깅 로그, DB 백업, 퇴사자 노트북,
  슬랙에 붙인 스크린샷, 티켓 첨부파일. 어느 하나가 언제 샜는지는 대개 모름.
- 90일 로테이션 = **"샜더라도 최대 90일이면 무효" 라는 상한**.
  #48 의 TTL 이 *옛 값을 쓰는 시간* 의 상한이라면, 로테이션 주기는 *유출된 값이 살아 있는 시간* 의 상한.

**왜 순간 전환이 불가능한가 —** **제공자는 하나, 컨슈머는 N개** 라는 비대칭 때문.
제공자는 "언제 바꿀지" 를 정할 수 있지만 **컨슈머가 "언제 받아쓸지" 는 통제하지 못함**.

- 주간 릴리스 팀 ↔ 분기 릴리스 팀
- 재시작이 없는 스트리밍 잡 (#48 의 캐시 문제와 같은 뿌리)
- 조직 밖 파트너 — 일정을 강제할 수단이 아예 없음
- 휴가·연말 코드 프리즈

**①에 숨은 순서 —** 도식엔 한 점으로 그려져 있지만 실제로는 둘로 나뉘고, 뒤집으면 사고가 남.

> **서버에 신 자격 증명을 먼저 등록 → 그다음 컨슈머에 배포**

반대로 하면 컨슈머가 신으로 바꿨는데 서버가 아직 모르는 상태가 되어,
**가장 성실하게 빨리 옮긴 컨슈머가 제일 먼저 죽음**. 도식에서 `new` 막대가 ①에서 시작하는 것은
"①시점에 서버가 이미 신을 받아들인다" 는 뜻.

**겹침 구간을 얼마로 잡나 —** 가장 느린 쪽에 맞춤.

> **겹침 ≥ max(가장 느린 컨슈머의 배포 주기, 최대 캐시 TTL, 스트리밍 재시작 주기) + 버퍼**

가장 느린 컨슈머가 분기 배포면 겹침도 최소 한 분기. "2주면 되겠지" 는 가장 느린 쪽을 안 본 것.

**③의 방아쇠가 "확인" 인 이유 —** 날짜로 정하는 것은 **가정에 기대는 것**.
"2주 지났으니 다 옮겼겠지" 는 검증이 아니라 희망. 확인할 **수단** 이 있어야 함.

- 접속 로그에서 **구 인증서 지문(fingerprint)·시리얼로 붙는 세션이 0** 인지
- 자격 증명별 **마지막 사용 시각(last used)** — AWS IAM 액세스 키가 제공하는 그 지표
- 인증 성공 카운터를 **자격 증명 단위로 분리** 해 집계

⚠ **확인 수단이 없으면 겹침을 아무리 길게 잡아도 안전하지 않음.** 그냥 "아마 됐을 것" 으로 끝나고,
폐기 버튼을 누르는 순간 결과를 알게 됨. 로테이션 설계는 **"어떻게 확인할 것인가" 를 먼저 정하는 일**
— "며칠 줄 것인가" 를 정하는 일이 아님.

**겹침이 없으면 —** 단순히 "몇몇이 실패" 보다 나쁨.

- **동시에** 터짐 — 폐기는 한 번의 동작이라 아직 못 옮긴 모든 컨슈머가 **같은 초에** 인증 실패.
  하나씩 순차로 터지면 알아채고 멈출 수 있는데 그 기회가 없음 ⇒ 장애 반경 최대.
- **복구가 느림** — 컨슈머 쪽 설정을 바꿔 배포해야 하는데, 그게 애초에 못 옮긴 이유.
  배포가 느려서 못 옮긴 팀은 복구도 느림.
- **롤백도 깔끔하지 않음** — 폐기가 CA 차원의 취소(revocation)였다면 되돌리기 어려움.

**겹침의 대가 —** 도식이 `✓` 만 말하는 자리의 보완.
**겹침 동안 공격면은 2배** — 구 자격 증명이 여전히 유효하니, 이미 유출된 상태라면 그 기간만큼 계속 열려 있음.

| 종류 | 겹침 | 근거 |
|---|---|---|
| **정기 로테이션** (예방) | 넉넉하게 | 유출 증거가 없으므로 무중단이 우선 |
| **유출 대응 로테이션** (사고) | 최소 또는 0 | 노출 차단이 우선 — 컨슈머 장애를 **의도적으로 감수** |

유출 사고에서 "무중단이니까 겹침을 두자" 는 판단은 틀림 — 그 구간이 곧 공격자에게 주는 시간.
반대로 겹침을 습관적으로 연장하면 구 자격 증명이 반영구적으로 살아남아 **로테이션 자체가 무의미** 해짐.

**인증서 특유의 함정 — ③은 안 눌러도 눌림.** 도식의 ③은 "**내가** 폐기한다" 는 전제인데, 인증서엔 **만료일** 이 있음.

> **만료는 시간이 대신 눌러 주는 ③** — 내 의지와 무관하게, 준비가 됐든 안 됐든 실행됨.

- 만료일은 **파일 안에 박혀 있어** 캘린더·모니터링에 올리지 않으면 아무도 모름 (아래 트러블 로그의 형태).
- 그래서 역산이 필요 — **로테이션 시작 데드라인 = 만료일 − 겹침 구간**.
  만료가 12월 31일이고 가장 느린 컨슈머가 분기 배포면 **9월에는 ①을 시작** 해야 함.
- **CA 체인 자체의 만료** 도 있음 — 서버 인증서만 갱신하고 컨슈머 쪽 CA 번들(`sslrootcert` 로 넘기는 그 파일)을 안 바꾸면 검증이 깨짐.

**#48 과 이어지는 지점 —** 두 도식은 한 문제의 양면.
#48 Secrets Pointer 를 함께 쓰면 ①의 **"컨슈머에게 공유"** 단계가 거의 사라짐
— 컨슈머는 값이 아니라 **이름** 을 참조하므로 시크릿 매니저의 값만 바꾸면 코드 배포 없이 전파됨.
그런데 **그래도 겹침은 필요** 함. 컨슈머가 값을 **캐시** 하고 있기 때문
⇒ 위의 `겹침 ≥ 최대 캐시 TTL` 규칙이 여기서 나옴.

**체크리스트**

1. 확인 **수단** 부터 만든다(자격 증명별 last-used·세션 로그). 없으면 로테이션 설계가 성립하지 않음
2. 서버에 신 등록 → 그다음 컨슈머 배포 (순서 뒤집지 말 것)
3. 겹침 = max(가장 느린 배포 주기, 최대 캐시 TTL, 스트리밍 재시작 주기) + 버퍼
4. 폐기는 **구 자격 증명 사용 0 확인 후** 에만
5. 인증서는 `만료일 − 겹침` 을 캘린더에 등록. **CA 번들 갱신** 도 함께
6. 유출 사고 시엔 위 원칙을 **의도적으로 깨고** 즉시 폐기

</div>
</details>

#### 구현 예시 (Examples)

**예시 1 — 인증서 기반 PostgreSQL 연결 (Example 7-28)**

Spark 잡의 연결 속성에 **패스워드가 없음**. 대신 서버와 공유한 **인증서** 에 의존:

```python
input_data = spark.read.option('driver', 'org.postgresql.Driver').jdbc(
  url='jdbc:postgresql:dedp', table='dedp.devices',
  properties={'ssl': 'true', 'sslmode': 'verify-full',       # 서버 호스트명까지 검증
   'user': 'dedp_test', 'sslrootcert': 'dataset/certs/ssl-cert-snakeoil.pem',
})
```

> `verify-full` SSL 모드는 **서버 호스트명이 서버 인증서에 저장된 이름과 일치하는지** 확인.
> (`verify-ca` 는 인증서 유효성만 보고 호스트명은 안 봄 — 중간자 공격 여지가 남음.)

**예시 2 — GCP Dataflow 잡이 GCS 를 읽게 (Example 7-29 ~ 7-30)**

먼저 **Service Account**(GCP 의 애플리케이션 사용자) 생성 — 이름과 적용 프로젝트만 있으면 됨:

```hcl
resource "google_service_account" "visits_job_sa" {
  account_id = "dedp"
  display_name = "Dataflow SA for processing visits from GCS"
}
```

다음으로 이 Service Account 를 **처리 대상 GCS 버킷** 과 **잡 자체** 에 연결. `role` 에 읽기 전용 권한:

```hcl
resource "google_storage_bucket_iam_binding" "visits_access" {
  bucket = "visits"
  role   = "roles/storage.objectViewer"       # 읽기 전용 — #44 최소 권한과 같은 결
  members = [
    "serviceAccount:${google_service_account.visits_job_sa.email}",
  ]
}

resource "google_dataflow_job" "visits_aggregator" {# ...
  service_account_email = google_service_account.visits_job_sa.email   # 잡에 "신원" 을 부여
}
```

> 이렇게 하면 `visits_aggregator` 잡이 **신원(identity)** 을 갖게 되고, 결과적으로 visits 버킷을 읽는 데
> **런타임에 제공할 자격 증명이 전혀 필요 없음**.

<details>
<summary><b>⚠ 트러블 로그</b> — "비밀이 없다" 는 말을 "관리할 게 없다" 로 읽으면 인증서 만료일에 파이프라인이 통째로 멈춤.</summary>
<div markdown="1">

**예 —** 인증서 기반으로 PostgreSQL 에 붙는 배치 잡 12개가 같은 `ssl-cert-snakeoil.pem` 을 공유하다가,
1년 만기 인증서가 만료되며 **같은 날 전부** `SSLHandshakeException: PKIX path validation failed` 로 실패.
패스워드와 달리 만료일이 **파일 안에만** 적혀 있어 아무도 캘린더에 넣어 두지 않았던 게 원인.

**반대 함정 —** IAM 기반으로 바꾸면 만료는 사라지지만, Service Account 에 `roles/storage.admin` 처럼
**넓은 role** 을 붙여 두면 자격 증명이 없다는 이유로 오히려 **점검 대상에서 빠짐** — #44 의 최소 권한이 무너지는 지점.

**권장 —** 인증서 기반은 **만료일을 모니터링 대상으로 등록** 하고, 로테이션은 **구/신 동시 지원 구간** 을 두고 진행할 것.
IAM 기반은 자격 증명이 없어도 **role 스코프를 #44 기준으로 정기 감사** 할 것.

</div>
</details>

---

## 5. 요약

챕터 7의 데이터 보안은 **"만든 데이터 자산을 어떻게 지키나"** 를 네 카테고리로 다룸 — **데이터 제거 · 접근 제어 · 데이터 보호 · 연결성**.

- **데이터 제거** — 잊힐 권리 대응. #41 **Vertical Partitioner** 는 **지울 대상을 애초에 줄이고**(컬럼 분리), #42 **In-Place Overwriter** 는 리팩터링 여유가 없을 때 **기존 저장소를 제자리에서 덮어써** 삭제(staging→승격, 블록 회수 필수).
  #42 는 더 비싸지만 사전 데이터 조직 전략이 없는 저장소에도 통해 **#41 보다 범용적**.
- **접근 제어** — 인가된 사용자만 접근. #43 **Fine-Grained Accessor for Tables** 는 **컬럼/행 단위**(GRANT·마스킹·row-level)로 DW·레이크하우스에, #44 **for Resources** 는 **클라우드 리소스 단위**(최소 권한, IAM resource/identity 기반)로 서버리스 NoSQL·오브젝트 스토어에 적용.
- **데이터 보호** — 데이터 자체를 보호. #45 **Encryptor** 는 저장·전송 데이터를 **암호화** 해 접근 통제가 뚫려도 **키 없이는 못 읽게** 하고,
  #46 **Anonymizer**·#47 **Pseudo-Anonymizer** 는 **데이터 공유 시나리오** 에서 데이터셋을 안전하게 만드는 방법.
- **연결성** — 저장소에 안전하게 연결. #48 **Secrets Pointer** 는 패스워드·인가 키를 **Git 저장소에 두지 않고** 쓰게 하고,
  가장 좋은 전략은 **관리할 자격 증명 자체가 없는 것** 이므로 로그인·패스워드 없는 상호작용에는 #49 **Secretless Connector** 를 씀.

| 패턴 | 카테고리 | 한 줄 요약 | 핵심 트레이드오프 |
|---|---|---|---|
| #41 Vertical Partitioner | Data Removal | 가변 event ↔ 불변 PII 를 컬럼 단위로 분리해 삭제 대상 최소화 | 쓰기·삭제↓ / <br>읽기 시 join·복잡도↑ |
| #42 In-Place Overwriter | Data Removal | 기존 저장소를 제자리에서 덮어써 삭제(staging→승격) | 리팩터링 불필요·범용 / <br>I/O·비용↑(전체 스캔) |
| #43 Fine-Grained Accessor (Tables) | Access Control | 테이블 컬럼/행 단위 접근 제어(GRANT·마스킹·RLS) | DB 네이티브 / <br>복합 타입·쿼리 오버헤드 |
| #44 Fine-Grained Accessor (Resources) | Access Control | 클라우드 리소스 단위 최소 권한(IAM) | 유출 반경↓ / <br>정책 폭증·쿼터·복잡도 |
| #45 Encryptor | Data Protection | 저장·전송 데이터를 암호화, 키 없으면 못 읽음 | 접근 뚫려도 보호 / <br>CPU 오버헤드·키 분실 위험 |
| #46 Anonymizer | Data Protection | 민감 값을 제거·교란·합성 대체해 재식별 불가로 | 결합해도 안전 / <br>정보 손실(분석 가치 훼손) |
| #47 Pseudo-Anonymizer | Data Protection | 민감 값을 쓸모 있는 가짜 값으로(마스킹·토큰화·해싱·암호화) | 분석 가능 / <br>결합 재식별·정보 손실 |
| #48 Secrets Pointer | Connectivity | 코드엔 시크릿 "이름" 만, 값은 시크릿 매니저에서 런타임 조회 | 코드베이스 분리·중앙 관리 / <br>캐시 무효화·로그 유출 |
| #49 Secretless Connector | Connectivity | IAM 신원·인증서로 자격 증명 없이 연결 | 관리할 비밀 0 / <br>설정 필요·인증서 로테이션 |

```
[챕터 7 전체 선택 가이드 — #41~#49]
──────────────────────────────────────────────────────────────────────
 삭제 대상을 지울 때
   새 설계·마이그레이션 여유 있음        ─► #41 Vertical Partitioner (컬럼 분리)
   레거시·자원 부족                     ─► #42 In-Place Overwriter (제자리 덮어쓰기)

 접근을 좁힐 때
   테이블의 컬럼/행 단위                 ─► #43 Fine-Grained Accessor for Tables
   클라우드 리소스(버킷·스트림) 단위      ─► #44 Fine-Grained Accessor for Resources

 데이터 자체를 지킬 때
   조직 안에서 저장·전송 보호            ─► #45 Encryptor (KMS/Key Vault·TLS)
   밖으로 공유, 분석 가치 포기 가능       ─► #46 Anonymizer (제거·교란·합성)
   밖으로 공유, 분석은 되게              ─► #47 Pseudo-Anonymizer (마스킹·토큰화·해싱·암호화)

 저장소에 연결할 때
   자격 증명을 코드 밖으로               ─► #48 Secrets Pointer (시크릿 매니저 참조)
   자격 증명을 아예 없애기               ─► #49 Secretless Connector (IAM·인증서)
──────────────────────────────────────────────────────────────────────
 ⚠ 삭제는 논리 삭제 뒤 물리 회수(VACUUM/블록 reclaim)까지, 암호화는 키 삭제에 유예 창을,
   의사 익명화는 데이터셋 "조합" 기준으로 재식별 위험을 평가할 것.
```

**정리** — 챕터 7은 **"지울 데이터를 줄이거나 확실히 지우고(#41·#42) → 접근을 컬럼·행·리소스 단위로 좁히고(#43·#44) →
뚫려도 못 읽게 암호화하고(#45), 밖으로 내보낼 땐 비식별화하고(#46·#47) → 연결에서 자격 증명을 숨기거나 없앤다(#48·#49)"** 로 이어지는 다층 방어선.

한 줄로 관통하는 원칙은 **"보호를 강하게 할수록 쓸모가 줄어든다"** —
#46 이 #47 보다 안전하지만 분석 가치는 반대이고, #49 가 #48 보다 비밀이 적지만 설정·로테이션 부담은 남음.
어느 층을 얼마나 조일지는 **컨슈머 (다운스트림)가 그 데이터로 무엇을 하는지** 를 보고 정할 일.

> 데이터 보안 패턴으로 여정의 끝이 가까워졌지만 **세 가지 주제가 남음**.
> 그중 첫째가 **데이터 스토리지** — 삭제 요청 대응을 돕는 동시에 **데이터 접근을 최적화** 하는 영역으로, 다음 챕터(챕터 8)의 주제.

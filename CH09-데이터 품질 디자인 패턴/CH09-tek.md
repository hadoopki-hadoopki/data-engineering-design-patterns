# 데이터 엔지니어링 디자인 패턴 - 데이터 품질 디자인 패턴

> 출처: Data Engineering Design Patterns (Bartosz Konieczny, O'Reilly 2025) Chapter 9 | 실무 데이터 엔지니어링 관점 정리

> **도식 기호 범례** — `⚠` 위험·함정(조심해야 할 동작) · `✗` 잘못된 결과(깨진 상태) · `✓` 올바른 결과 · `⇒` 그 결과 도출

---

## 목차

1. [품질 확보 (Quality Enforcement)](#1-품질-확보-quality-enforcement)
   - 패턴 #59: 감사－쓰기－감사－배포 (Audit-Write-Audit-Publish, AWAP)
   - 패턴 #60: 제약 조건 적용자 (Constraints Enforcer)
2. [스키마 일관성 (Schema Consistency)](#2-스키마-일관성-schema-consistency)
   - 패턴 #61: 스키마 호환성 적용자 (Schema Compatibility Enforcer)
   - 패턴 #62: 스키마 마이그레이터 (Schema Migrator)
3. [품질 관찰 (Quality Observation)](#3-품질-관찰-quality-observation)
   - 패턴 #63: 오프라인 옵서버 (Offline Observer)
   - 패턴 #64: 온라인 옵서버 (Online Observer)
4. [요약](#4-요약)

> 본 문서는 **챕터 9 전체** — **#59~#64** — 를 다룸.
> **#65 Flow Interruption Detector** 부터는 **챕터 10 데이터 관찰 가능성** 문서에서 정리.

---

## 책의 use case (챕터 도입)

> **신뢰(trust)** 는 데이터셋의 중요한 가치. 데이터 교환은 **상호 거래** 와 같아서,
> 서비스(데이터셋)를 **제공** 하거나 **소비** 하는 쪽에 서게 됨.
> 최종 목표는 **프로듀서와 컨슈머 (받아 쓰는 쪽) 모두가 이 교환에 만족** 하는 것.
> 유감스럽게도 **신뢰할 수 없는 데이터셋** 으로 일하는 것은 즐거울 수 없음 —
> 거기서 뽑아낸 인사이트가 **언제든 틀린 것이 될 수 있기** 때문.

신뢰를 잃는 원인 중 하나가 **낮은 데이터셋 품질** — 즉 **불완전성(incompleteness) · 부정확성(inaccuracy) ·
불일치(inconsistency)**. 다행히 이 문제들은 새로운 것이 아니고, 완화할 디자인 패턴이 존재.

```
[챕터 9 데이터 품질 패턴 — 세 카테고리]
──────────────────────────────────────────────────────────────────────
 9.1 Quality Enforcement   품질을 강제해 저품질 데이터를        #59 AWAP
                           컨슈머 (다운스트림)에게 노출하지 않음  #60 Constraints Enforcer
 9.2 Schema Consistency    스키마 수준의 품질 이슈를 해결        #61 Schema Compatibility Enforcer
                           (프로듀서의 스키마 변경이 원인)        #62 Schema Migrator
 9.3 Quality Observation   오늘의 규칙이 내일의 데이터에도       #63 Offline Observer
                           유효한지 계속 관찰                    #64 Online Observer
──────────────────────────────────────────────────────────────────────
 흐름 — 값을 막고(9.1) → 스키마를 막고(9.2) → 막는 규칙 자체가 낡지 않게 지켜봄(9.3).
 ※ 본 문서는 세 카테고리 전체 — 즉 #59~#64 를 다룸.
```

**세 카테고리가 나뉜 이유**

- **① 품질 확보** — 저품질 데이터를 **컨슈머 (다운스트림)에게 노출하지 않는 것** 이 첫 번째 방어선.
- **② 스키마 일관성** — 프로듀서는 대개 문제없이 데이터를 생성하다가, **어느 날 스키마를 바꿈**.
  진화 유형에 따라 이것이 **파이프라인의 치명적 실패** 와 **데이터 제공자에 대한 신뢰 상실** 로 이어짐.
- **③ 품질 관찰** — **오늘 정한 강제 규칙이 내일의 데이터에도 유효한가** 를 보장하는 일.
  데이터와 스키마를 통제하는 것에 더해, **컨슈머 (조회하는 쪽)보다 먼저 새 이슈를 발견** 하려면
  데이터셋을 관찰해야 함. 관찰 기법이 **처리 중인 데이터셋의 가장 최신 개요** 를 제공해
  강제 규칙을 최신 상태로 유지하게 해 줌.

### 패턴 흐름 — 챕터 8에서 챕터 10 으로

```
[패턴 흐름 — 챕터 8에서 챕터 10 으로]
──────────────────────────────────────────────────────────────────────
 챕터 8: 데이터를 나누고·묶고·정렬해 "빠르고 싸게 읽히게" 만듦
      │ 남은 과제: 아무리 잘 최적화해도 "남이 쓰게 만들기엔" 부족 — 신뢰가 없으면 안 씀
      ▼ "이 숫자, 믿어도 되나?"
 9.1 Quality Enforcement (값을 막기)
   #59 AWAP — 파이프라인 안에 감사 단계를 넣어 입력·출력 둘 다 검증
      │ (한계) 검증 로직을 내가 다 짜야 함 = 구현 부담이 내 쪽
      ▼ "DB 가 대신 막아 줄 수는 없나?"
   #60 Constraints Enforcer — 검증을 DB·스토리지 포맷에 위임 (선언적)
      │ (한계) 값은 막지만 스키마 변경 자체는 못 막음
      ▼ "필드를 통째로 지워 버리면?"
 9.2 Schema Consistency (스키마를 막기)
   #61 Schema Compatibility Enforcer — 호환되지 않는 스키마 변경을 거부
      │ (한계) 변경의 "종류" 만 통제. 의도적인 파괴적 변경은 어떻게?
      ▼ "필드 이름을 바로잡고 싶은데, 컨슈머를 깨지 않으려면?"
   #62 Schema Migrator — 유예 기간 동안 옛 필드·새 필드를 함께 실어 보냄
      │ (한계) 값도 스키마도 통제했지만, 통제 규칙 자체가 낡는 문제는 그대로
      ▼ "내가 정한 규칙이 내일의 데이터도 덮는가?"
 9.3 Quality Observation (규칙이 낡지 않게 관찰)
   #63 Offline Observer — 관찰을 별도 파이프라인으로 분리 (non-blocking)
      │ (한계) 스케줄이 느슨하면 컨슈머 (다운스트림)가 먼저 이슈를 발견
      ▼ "주 1회로는 늦다"
   #64 Online Observer — 관찰을 생성 파이프라인 안으로 넣음
      │ (한계) 데이터는 봤지만, 파이프라인이 "아예 안 돌았을 때" 는 못 봄
      ▼ "AWAP 이 완벽해도 AWAP 잡이 안 돌면 소용없다"
 챕터 10 Data Observability — #65 Flow Interruption Detector 로 이어짐 (별도 문서)
──────────────────────────────────────────────────────────────────────
```

---

## 1. 품질 확보 (Quality Enforcement)

데이터셋의 품질을 보장한다는 것은 **불완전하거나 불일치하거나 부정확한 데이터셋을 공유하지 않겠다** 는 뜻.
품질 확보는 **신뢰할 만한 데이터를 공유** 한다는 목표로 파이프라인에 적용하는 **첫 번째 카테고리**.

```
[품질 확보 — 두 패턴의 책임 위치]
──────────────────────────────────────────────────────────────────────
 #59 AWAP                  #60 Constraints Enforcer
 ─────────────────────     ────────────────────────────
 검증이 사는 곳:            검증이 사는 곳:
   내 파이프라인 코드         데이터베이스 · 테이블 포맷 · 직렬화 스키마
 방식: 명령적(imperative)   방식: 선언적(declarative)
 표현력: 프로그래밍 언어      표현력: DB 가 지원하는 제약 종류까지
         이 허용하는 전부
 실패 단위: 내가 정함         실패 단위: 대개 트랜잭션 전체 (all-or-nothing)
 ⇒ 둘은 대체재가 아니라 보완재 — DB 제약으로 막을 수 있는 건 DB 에 맡기고,
   나머지(볼륨 급감·분포 변화 등)를 AWAP 감사 단계가 맡음.
──────────────────────────────────────────────────────────────────────
```

---

### 1-1. 패턴 #59: 감사－쓰기－감사－배포 (Audit-Write-Audit-Publish, AWAP)

> 좋은 데이터셋 품질을 보장하는 첫 번째 방법은 **데이터 흐름에 제어 장치를 추가** 하는 것.
> 이는 **단위 테스트의 어서션(assertion)** 과 비슷 —
> 코드가 기대 입력에 대해 올바르게 동작하는지 확인하는 그 장치를 **데이터 흐름으로 옮겨 놓은 것**.
> 그 결과 **데이터셋이 기대에 못 미치면 실행 전체를 멈추는 품질 가드** 가 파이프라인에 생김.

#### 상황 (Problem)

**책의 use case** — 고유 방문자 50% 급감이 사실은 집계 버그였음:

- 일일 배치 ETL 잡이 **Figure 1-1**(1장 케이스 스터디)의 **사용자 방문 통계** 를 생성 중.
- **지난 한 주간 결과가 좋지 않았음.** 실제로 **고유 방문자 수가 50% 감소** 했고,
  프로덕트 팀은 이를 이슈로 판단.
- 그 결과 프로덕트 팀은 **방문자를 웹사이트로 끌어오기 위한 새 마케팅 캠페인을 시작**.
- 그런데 오늘 이 잡의 새 기능을 작업하다가 **고유 방문자 집계가 올바르게 계산되지 않고 있음** 을 발견.
  프로덕트 팀에 알렸고, 팀은 **캠페인을 중단** 했지만 **앞으로 유사한 이슈가 없도록 보장** 해 달라고 요청.
- **결정적 제약**: 잘못된 숫자가 **파이프라인을 통과했다는 사실 자체를 아무도 몰랐음**.
  잡은 성공했고, 실패한 것은 **데이터** 뿐.

```
[사고의 구조] 잡은 성공했는데 데이터는 틀렸음
──────────────────────────────────────────────────────────────────────
 D-7  집계 잡 ✓ 성공 ─► unique_visitors = 52,000 (전주 대비 -50%)  ⚠ 아무도 못 막음
 D-6  집계 잡 ✓ 성공 ─► unique_visitors = 51,300                   ⚠
  …
 D-1  집계 잡 ✓ 성공 ─► unique_visitors = 50,900                   ⚠
      프로덕트 팀 ─► "방문자가 줄었다" ─► 마케팅 캠페인 집행 (예산 소진)
 D+0  엔지니어가 코드 수정 중 우연히 발견 ─► 집계 로직 버그 ⇒ 숫자가 애초에 틀렸음
──────────────────────────────────────────────────────────────────────
 ⇒ 잡 실패(exit code)만 감시하는 오케스트레이터는 이 사고를 절대 못 잡음.
 ⇒ 필요한 것은 "데이터 자체에 대한 어서션".
```

#### 해결 (Solution)

**이전 날짜들 대비 50% 급감한 데이터 볼륨** 은 **AWAP 패턴이 빛나는 완벽한 use case**.

> **참고 사항 — WAP 의 진화 (Write-Audit-Publish Evolution)**
> AWAP 패턴은 **Michelle Ufford** 가 **2017 DataWorks Summit** 에서 공유한
> **WAP(Write-Audit-Publish) 패턴의 진화형**.
> WAP 을 세상에 소개한 원본 발표는 유튜브에서 볼 수 있음
> — "Whoops, the Numbers Are Wrong! Scaling Data Quality @ Netflix".
> WAP 과 달리 **AWAP 은 입력 데이터 검증까지 포함** 해,
> 보통 **가벼운 검증을 입력 데이터셋에 대해 먼저 수행**.

AWAP 의 아이디어는 **입력·출력 데이터셋 모두가 정의된 비즈니스·기술 요구사항
(완전성·정확성 등)을 충족하는지 확인하는 제어 장치(= 감사 단계, audit step)를 추가** 하는 것.

```
[Figure 9-1 재현] 파이프라인에 적용한 AWAP
────────────────────────────────────────────────────────────────────────────────────────
  ┌─────────┐
  │ Source  │
  └────┬────┘
       ▼
  ╔═════════╗   ┌─────────┐   ┌───────────┐   ┌─────────┐   ╔═════════╗   ┌──────┐
  ║  Audit  ║──►│ Extract │──►│ Transform │──►│ Staging │──►║  Audit  ║──►│ Load │
  ╚═════════╝   └─────────┘   └───────────┘   └─────────┘   ╚═════════╝   └──┬───┘
                                                                             ▼
                                                                        ┌─────────┐
                                                                        │ Target  │
                                                                        └─────────┘
────────────────────────────────────────────────────────────────────────────────────────
 ╔═╗ 로 표시한 두 단계가 감사(audit) 태스크. 나머지는 평범한 ETL.
 ① Audit  — 변환 시작 전, 입력 데이터 소스를 검사
 ② Audit  — 변환이 끝난 뒤, 스테이징에 있는 결과를 검사 (통과해야 Load 로 진행)
 ⇒ Load 가 마지막에 오는 것이 핵심 — 감사에 실패하면 최종 저장소는 손도 대지 않은 상태로 남음.
```

**두 감사 태스크의 진짜 차이는 "감사 대상 데이터 스토어"**

- **① 첫 번째 감사 잡 — 입력 데이터 소스**
  - 데이터셋 변환을 **시작하기 전** 에 입력 소스를 분석하는 책임.
  - 아주 흔히 검증을 **빠른 연산으로 제한** — **입력 파일 포맷 검증 · 파일/테이블 크기 제어 · 스키마 체크**.
  - 예 — 보통 `a`·`b`·`c` 세 컬럼을 가진 테이블에 새 CSV 파일을 적재한다고 하면,
    첫 번째 감사 단계는 **파일의 첫 줄만 분석** 해 세 필드가 존재하고 올바르게 정의됐는지 검증할 수 있음.
  - ⚠ **전체 데이터셋 검증도 기술적으로 가능하지만**, 그러면 **데이터셋을 두 번 읽을 위험**
    (감사 단계에서 한 번, 변환 단계에서 또 한 번)이 있음을 유념할 것.
- **② 두 번째 감사 잡 — 변환된 데이터**
  - **로컬 단위 테스트의 확장** 으로 볼 수 있음 — **실제 데이터셋 위에서 돌아가는** 버전.
  - 따라서 제어 함수는 **데이터 자체에 더 초점**.
  - 예 — 위의 `a`·`b`·`c` 를 변환하는데 그 결과가 **절대 `NULL` 이면 안 된다면**,
    여기에 검증 함수를 추가.

**"중복 검증" 판단에 아주 조심할 것**

같은 `NULL` 검증이라도 **의도(intent)와 범위(scope)가 다름**.

```
[같은 NULL 검증, 다른 의미]
──────────────────────────────────────────────────────────────────────
 입력 데이터셋에 붙인 NULL 검증
   ⇒ "데이터 제공자가 만든 데이터셋이 내 기대를 충족하는가?"   (남의 책임을 확인)
 출력 데이터셋에 붙인 NULL 검증
   ⇒ "내 변환 로직이 결측값을 만들어 내지 않는가?"            (내 책임을 확인)
──────────────────────────────────────────────────────────────────────
 겉보기엔 둘 다 NULL 을 본다. 하지만 실패했을 때 전화를 걸 상대가 다름.
```

**같은 검증 함수를 두 곳에 두면 전체 데이터셋을 여러 번 처리하는 위험** 이 있음.
데이터 볼륨 때문에 그것이 걱정된다면, 검증 동작을 **가장 포괄적인(exhaustive) 위치에 두는 것** 을 항상 고려할 것.
`NULL` 검증의 경우 그곳은 **두 번째 감사 단계** — 거기서는 **입력 데이터셋에서 온 `NULL` 과
내 변환 로직이 만든 `NULL` 을 모두 잡을 수 있음**.

> **참고 사항 — 단위 테스트와 AWAP (Unit Tests and AWAP)**
> 단위 테스트는 소프트웨어에 의존하는 어떤 시스템에서든 중요하고, 데이터 엔지니어링 파이프라인도 예외가 아님.
> 다만 **데이터에 관해서는 단위 테스트가 정적(static)** — 어느 한 시점에 작성한 것이라
> 지금의 현실은 반영해도 **앞으로 벌어질 일을 대변하지는 못함**.
> 그래서 AWAP 의 감사 단계가 **실세계 데이터 위에서 단위 테스트를 확장** 한다고 말한 것.
> 오해는 말 것 — **(로컬에서 도는) 단위 테스트는 잘못된 비즈니스 로직 구현이 유발하는
> 데이터 품질 이슈에 대한 언제나 첫 번째 방어선**.

**검증의 두 수준**

- **레코드 수준(records level)** — 특정 레코드의 속성을 검증.
- **데이터셋 수준(dataset level)** — 전체 속성을 검증. **데이터 볼륨 · 특정 컬럼의 고유성(distinctiveness) ·
  어떤 컬럼의 `NULL` 비율** 등.

**감사 실패의 결과가 항상 "파이프라인 실패" 는 아님**

- **데이터 디스패칭 (data dispatching)**
  - 감사한 출력 데이터셋의 **일부만 유효하지 않다면**, **유효한 부분은 그대로 컨슈머 (다운스트림)에게 승격**
    하고 **유효하지 않은 레코드는 별도 스토리지에 보관** 할 수 있음.
  - **Dead-Letter 패턴처럼 들리지만 다름** — 여기엔 **예기치 못한 런타임 오류가 없음**.
    dead-lettering 로직이 **내가 명시적으로 만든 데이터 제어 메커니즘의 결과** 로 발생.
- **비차단 감사 (nonblocking audit)**
  - 처리된 데이터셋에 **약간의 결함이 있어도** 최종 저장소로 승격시키고 싶을 수 있음.
  - 그 경우 **이슈가 있다고 주석(annotate)** 을 달아, 리더가 **신뢰도를 스스로 평가** 하고
    기대에 못 미치면 처리하지 않도록 하는 것이 좋음.
  - 예 — 어떤 컬럼의 `NULL` 이 예상외로 늘었지만 데이터셋 전반은 멀쩡하다면,
    **그 컬럼을 안 쓰는 컨슈머 (다운스트림)는 그대로 사용** 가능.
    그 컬럼을 쓰는 컨슈머도 **결측 비율이 자신의 허용 임계값 이하라면** 처리하기로 결정할 수 있음.
  - 주석은 **테이블이나 파일에 data summary 엔트리를 만들어** 가능한 품질 이슈를 나열하는 식으로 구현.

```
[감사 실패 시 세 갈래]
──────────────────────────────────────────────────────────────────────
 감사 결과 ─┬─ ① fail       ─► 파이프라인 중단. 최종 저장소는 이전 상태 그대로.       ✓ 가장 안전
            │                    ⚠ 배치가 통째로 밀림 — SLA 와 충돌할 수 있음
            ├─ ② dispatch   ─► 유효 레코드만 승격 + 무효 레코드는 격리 스토리지로
            │                    ⚠ Dead-Letter 와 달리 "런타임 오류" 가 아니라 "내 규칙" 의 결과
            └─ ③ nonblocking ─► 전부 승격 + 데이터 요약(annotation)에 이슈 기록
                                 ⚠ 컨슈머 (조회하는 쪽)가 주석을 읽는다는 전제가 성립해야 의미가 있음
──────────────────────────────────────────────────────────────────────
```

**스트리밍에서의 AWAP**

지금까지의 설명은 AWAP 이 배치 전용처럼 들리지만 사실이 아님. **스트림 워크로드도 두 가지 방식으로 사용** 가능.

```
[Figure 9-2 재현] 스트리밍에 적용한 AWAP
──────────────────────────────────────────────────────────────────────────────────────────
 ① window based (윈도 기반)
   ┌───────┐   ┌───────────────┐   ┌────────────────┐   ╔════════════════╗   ┌────────┐
   │ Input │──►│ Streaming job │──►│ Data window    │──►║ Audit function ║──►│ Output │
   └───────┘   └───────────────┘   └────────────────┘   ╚════════════════╝   └────────┘
                                     처리 시간 윈도를           스트리밍 잡
                                     잡 안에서 직접 생성         "안에서" 감사

 ② staging based (스테이징 기반)
   ┌───────┐   ┌───────────────┐   ┌────────────────┐   ╔════════════════╗   ┌────────┐
   │ Input │──►│ Streaming job │──►│ Staging layer  │──►║ Audit job      ║──►│ Output │
   └───────┘   └───────────────┘   └────────────────┘   ╚════════════════╝   └────────┘
                                     출력만 스테이징으로        별도 잡이
                                     바꾸고 로직은 그대로        "밖에서" 감사
──────────────────────────────────────────────────────────────────────────────────────────
 ① 윈도가 닫히면 잡이 감사 단계를 실행하고, 버퍼된 레코드에 fail/dispatch/ignore 중 하나를 적용.
 ② 데이터 처리 로직은 손대지 않음. 출력을 스테이징 계층에 쓰고, 감사 잡이 검증한 뒤 최종 위치로 승격.
 ⇒ 두 방식 모두 첫 번째 감사 단계가 없음 = 스트리밍의 AWAP 은 고전적 WAP 에 가까움.
   데이터가 연속적으로 흘러 들어오므로, 대개 변환 뒤에 검증하는 편이 더 간단하기 때문.
   앞서 말한 "가장 포괄적인 위치에 검증을 둔다" 는 규칙의 실제 사례.
```

#### 고려사항 (Consequences)

AWAP 은 **추가 안전성** 을 주지만 **추가 비용** 을 치름.

- **Compute cost (컴퓨팅 비용)**
  - 감사 단계의 성격에 따라 **추가 컴퓨팅 비용** 이 발생할 수 있음.
  - **메타데이터 기반 연산**(파일 포맷 검증 등)은 저렴하지만,
    **데이터를 실제로 훑는 연산**(row 기반 검증 등)은 더 비쌈.
  - 다만 그것이 **생성한 데이터의 품질을 보장하기 위해 치르는 값**.
- **Rules coverage (규칙 커버리지)**
  - row 검증을 예로 들면, 들어오는 각 row 의 값을 검증하기 위해 **비즈니스 규칙 집합** 을 정의하게 됨.
  - 유감스럽게도 **데이터셋은 시간에 따라 진화** 하므로 **오늘의 규칙이 내일의 데이터셋을 온전히 덮지 못할 수 있음**.
  - 그래서 **AWAP 으로 통제되는 파이프라인을 100% 신뢰할 수 있다고 여기지 않는 편이 나음**.
    잊혔거나 낡은 검증의 위험은 남아 있고, 그것은 **9.3 품질 관찰(#63·#64)** 패턴으로 잡아야 함.
- **Streaming latency (스트리밍 지연)**
  - 스트리밍 맥락의 AWAP 은 **추가 지연** 을 만들 수 있음.
  - 예 — **처리 시간 윈도 안에서 `NULL` 분포를 확인** 하고 싶다면,
    **데이터 전달이 윈도 누적 기간만큼 늦어짐**.
- **An issue may not be an issue (이슈가 이슈가 아닐 수 있음)**
  - 감사 단계가 잡아낸 이슈가 **실제 이슈가 아닐 수 있음** 을 유념할 것.
    놀랍게 들리겠지만 **데이터는 동적** 이고, 틀려 보이는 것이 사실은 맞는 것으로 판명될 수 있음.
  - 예 — 블로깅 플랫폼의 **데이터 볼륨을 검증하는 감사 단계**.
    **소셜 미디어에 인용되는 것 같은 예상치 못한 성공** 을 만나면,
    방문 수가 예상외로 높아지고 처리할 데이터 볼륨도 훨씬 커지는 것이 **정상**.
    프로듀서 쪽에 뭔가 잘못됐다는 뜻이 아님.
  - ⇒ **모든 감사 실패를 치명적 이슈로 볼 필요는 없음.** 때로는 **알림만 발생시키고
    추가 조사를 요구** 하는 것으로 충분.

```
[감사 실패 = 데이터 이슈? 아님]
──────────────────────────────────────────────────────────────────────
 규칙: "일일 방문 수가 전일 대비 ±30% 를 벗어나면 실패"
   케이스 A  방문 50,000 → 25,000 (-50%)   ⇒ 집계 버그 · 수집 누락      ✗ 진짜 이슈
   케이스 B  방문 50,000 → 210,000 (+320%) ⇒ 글이 소셜에 퍼짐           ✓ 정상
──────────────────────────────────────────────────────────────────────
 ⇒ 같은 규칙에 걸려도 A 는 파이프라인 중단, B 는 알림 후 진행이 맞음.
   감사 결과를 "실패/성공" 이 아니라 "차단/경고" 두 등급으로 나눠 둘 것.
```

#### 구현 예시 (Examples)

**예시 1 — Apache Airflow + PostgreSQL 배치 파이프라인 (Example 9-1)**

파이프라인이 **입력 데이터셋 감사로 시작** 하고, 검증에 성공하면 변환을 시작하며,
그 결과가 **다시 검증된 뒤에야 최종 데이터 스토어에 쓰임**.

```python
audit_file_to_load = PythonOperator(
    task_id='audit_file_to_load',
    python_callable=local_validate_the_file_before_processing   # ① 입력 감사
)
transform_file = PythonOperator(
    task_id='transform_file',
    python_callable=flatten_input_visits_to_csv
)
def local_validate_flatten_visits():
  validate_flatten_visits(get_current_context())

audit_transformed_file = PythonOperator(
    task_id='audit_transformed_file',
    python_callable=local_validate_flatten_visits               # ② 출력 감사
)
load_flattened_visits_to_final_table = PostgresOperator(
    task_id='load_flattened_visits_to_final_table',
    sql='/sql/load_file_to_visits_table.sql'                    # 감사를 통과해야 도달
)

(next_partition_sensor >> audit_file_to_load >> transform_file
  >> audit_transformed_file >> load_flattened_visits_to_final_table)
```

**예시 2 — 입력 데이터셋 검증 (Example 9-2)**

입력 감사는 **JSON 라인의 정확성과 전체 파일 크기** 를 어서션.

```python
if f_size < min_size:
  validation_errors.append(
    f'File is to small. Expected at least {min_size} bytes but got {f_size}')
if lines < min_lines:
  validation_errors.append(
    f'File is too short. Expected at least {min_lines} lines but got {lines}')
if invalid_json_line:
  validation_errors.append(
    f'File contains some invalid JSON lines. The first error found was
    {invalid_json_line}, line {invalid_json_line_number}')

# 첫 오류에서 멈추지 않고 errors 를 모아 두었다가 마지막에 한 번에 던지는 것이 요점
if validation_errors:
  raise Exception('Audit failed for the file:\n-'+"\n-".join(validation_errors))
```

> 보다시피 **최종 오류 메시지가 입력 파일이 가진 이슈를 전부 담음**.
> `#60 Constraints Enforcer` 의 **all-or-nothing + 첫 오류에서 중단** 과 대비되는 지점.

**예시 3 — 처리된 데이터셋 검증 (Example 9-3)**

같은 로직이 처리된 데이터셋에도 적용되며, 여기서는 **pandas** 로 `NULL` 을 찾음.
이 잡은 **제약이 없는 CSV 포맷** 에서 동작하므로 이런 별도 `NULL` 체크가 필요.
그렇지 않다면 **`#60 Constraints Enforcer` 에 기댈 수 있었음**.

```python
required_columns = ['visit_id', 'event_time', 'user_id', 'page', 'ip', 'login',
 'browser', 'browser_version', 'network_type', 'device_type', 'device_version']
cols_w_nulls = []
visits = pandas.read_csv(partition_file(context, 'csv'), sep=';', header=0)
for validated_column in required_columns:
 if visits[validated_column].isnull().any():
  cols_w_nulls.append(validated_column)

if columns_with_nulls:   # ※ 원서 표기 그대로. 위에서 만든 변수는 cols_w_nulls
 raise Exception('Found nulls in not nullable columns:'+','.join(cols_w_nulls))
```

**예시 4 — Spark Structured Streaming: 스테이징 테이블에 쓰기 (Example 9-4)**

스트리밍 잡을 돌리는 방법 중 하나가 **트리거**(데이터 처리 로직의 실행 주기를 정의하는 시간 기반 표현식).
**처리 윈도의 훌륭한 대안** — **상태가 없어(stateless)** 상태 관리 오버헤드를 자연스럽게 피함.

```python
visits = (spark_session.readStream
  .option('kafka.bootstrap.servers', 'localhost:9094').option('subscribe', 'visits')
  .option('startingOffsets', 'EARLIEST').option('maxOffsetsPerTrigger', '50')
  .format('kafka').load()
  .selectExpr('CAST(value AS STRING)')
   .select(F.from_json("value", get_visit_event_schema()).alias("visit"), "value")
  .selectExpr('visit.*')
)
# ...
write_query = (visits.writeStream
  .trigger(processingTime='15 seconds')      # 윈도 대신 트리거 = stateless
  .option('checkpointLocation', checkpoint_dir)
  .foreachBatch(write_dataset_to_staging_table).start())   # 최종이 아니라 스테이징
```

**예시 5 — 스테이징 테이블 위의 감사 잡 (Example 9-5)**

두 번째 잡이 **스테이징 테이블을 스트리밍** 하며 데이터 품질 제어를 수행.
평가 결과에 따라 **이슈가 없으면 최종 목적지로, 있으면 오류 목적지로** 씀.

```python
visits = (spark_session.readStream.format('delta')
  .option('maxBytesPerTrigger', 20000000)
  .table(get_staging_visits_table())
  .withColumn('is_valid', row_validation_expression)   # row 마다 유효성 플래그
)
# ...
write_query = (visits.writeStream
  .trigger(processingTime='30 seconds')
  .option('checkpointLocation', checkpoint_dir)
  .foreachBatch(audit_dataset_and_write_to_output_table)   # 여기서 fail/dispatch 분기
  .start())
```

<details>
<summary><b>⚠ 트러블 로그</b> — 입력 감사에 무거운 전수 검증을 넣으면 데이터셋을 두 번 읽어 배치 시간이 두 배가 됨.</summary>
<div markdown="1">

**예 —** "입력부터 철저히 보자" 며 1차 감사에서 **원본 JSON 8TB 를 전부 파싱해 NULL 비율까지** 검사함.
변환 단계가 같은 8TB 를 다시 읽으면서 **일일 배치가 40분 → 1시간 25분** 으로 늘고,
S3 GET 요청 비용도 두 배가 됨. 정작 잡아낸 이슈는 **2차 감사에서도 전부 잡히는 것들** 이었음.

**권장 —** 1차 감사는 **첫 줄 스키마 확인 · 파일 크기 · 줄 수 · 포맷 유효성** 같은
**메타데이터 중심의 빠른 검사** 로 제한하고, 레코드 값 검증은 **가장 포괄적인 위치인 2차 감사** 에 몰 것.

</div>
</details>

<details>
<summary><b>⚠ 트러블 로그</b> — 감사를 최종 테이블에 쓴 "뒤" 에 두면 AWAP 이 아니라 사후 부검이 됨.</summary>
<div markdown="1">

**예 —** Airflow DAG 를 `transform >> load >> audit` 순으로 짜 둠.
감사가 실패해도 **이미 `visits_daily` 최종 테이블에 저장된 상태** 라,
BI 대시보드는 새로고침 때 **틀린 숫자를 그대로 보여 줌**. 롤백하려면
`DELETE FROM visits_daily WHERE dt='2026-08-21'` 를 수동으로 돌려야 하고,
그 사이 캡처된 스크린샷은 회수할 방법이 없음.

**권장 —** **감사 → 스테이징 → 감사 → 배포** 순서를 지킬 것.
`load` 는 반드시 **두 번째 감사 뒤** 에 오고, 스테이징은 **컨슈머 (조회하는 쪽)에게 노출되지 않는 위치** 여야 함.

</div>
</details>

---

### 1-2. 패턴 #60: 제약 조건 적용자 (Constraints Enforcer)

> AWAP 패턴은 **데이터 처리 파이프라인 안에서 직접** 데이터를 검증. 달리 말해 **구현 부담이 내 쪽**.
> 그런데 신뢰할 만한 데이터셋을 만드는 **더 쉬운 길** 이 있음 —
> 품질 제어를 **데이터베이스나 스토리지 포맷에 위임** 해 **선언적(declarative) 접근** 에 기대는 것.

#### 상황 (Problem)

**책의 use case** — 필수 필드에 무작위 `NULL` 이 섞여 들어옴:

- 배치 파이프라인이 **Figure 1-1**(1장)의 방문을 처리해 결과를 테이블에 다시 씀.
- **몇 달 동안 아무 이슈 없이 돌았는데**, 이제 **여러 필수 필드에서 무작위 `NULL` 값** 이 나오고 있음.
- **데이터 처리 잡은 이미 복잡** 해서 **거기에 데이터 검증 복잡도를 더하고 싶지 않음**.
- **결정적 제약**: 필수 필드 누락 같은 **데이터 품질 오류가 있으면 적재 프로세스 자체를 실패시키는**
  대안적 접근이 필요.

#### 해결 (Solution)

**검증 책임을 데이터베이스에 위임하는 것** 이 Constraints Enforcer 패턴이 하는 일.

구현은 **제약 규칙을 붙일 속성을 식별** 하는 데서 시작.
이는 **매우 비즈니스 특화적인 단계** 로, 규칙은 **프로덕트 팀이나 법규** 가 주도할 수 있음.
예를 들어 orders 데이터셋이라면 **주문 금액** 과 **구매자의 청구 주소** 같은 속성이 반드시 정의돼야 함.
다만 이는 이커머스에 한정된 예시고, **모두에게 맞는 하나의 정답은 없음**.

속성을 식별했다면 **제약을 할당** 할 차례. 제약은 여러 카테고리로 나뉨.

```
[제약의 네 카테고리]
──────────────────────────────────────────────────────────────────────
 ① Type        "이 컬럼은 항상 같은 타입"        event_time TIMESTAMP
                └─ 데이터셋 스키마의 일부이자 9.2 스키마 일관성의 뼈대
 ② Nullability "결측이 가능한가 / 절대 아닌가"   visit_id STRING NOT NULL
                └─ 컨슈머 (조회하는 쪽)에게 "이 컬럼은 필터가 필요하다" 는 신호도 됨
 ③ Value       "허용되는 값·범위·표현식"         CHECK (x <= NOW())
                                                CHECK (x BETWEEN 1901 AND 2000)
 ④ Integrity   "참조하는 값이 실제로 존재하는가"  visits.page_id → pages.id (FK)
                └─ Normalizer(#57)로 모델링한 트랜잭션 DB 에서 주로 등장
──────────────────────────────────────────────────────────────────────
 ⇒ ①②③ 은 대부분의 테이블 파일 포맷도 지원. ④ 는 대개 관계형 DB 에만 있음.
```

- **타입 제약 (type constraints)**
  - 주어진 속성의 모든 값이 **항상 같은 타입** 임을 보장.
  - 컨슈머 (조회하는 쪽)가 **어떤 종류의 데이터를 다루는지 알게 되므로 처리가 크게 단순해짐**.
  - 타입 기반 제약은 **데이터셋 스키마의 일부** 이자 **Schema Consistency 패턴의 근간**.
- **널 허용 제약 (nullability constraints)**
  - 속성을 **절대 결측 없음** 또는 **결측 가능** 으로 정의.
  - not nullable 이면 **결측값이 있는 row 를 거부**. nullable 이면 결측값을 받아들임.
  - 이 설정은 컨슈머 (다운스트림)에게 **추가할 만한 연산** — 예컨대 널이 있을 수 있는 컬럼을
    필터링해 결측 row 를 제거하는 것 — 도 함께 알려 줌.
- **값 제약 (value constraints)**
  - 속성에 허용되는 **하나의 값 · 값 집합 · 표현식** 과 **비교 연산자** 에 기댐.
  - 삽입되는 값을 기대값과 비교해 **결과가 부정이면 그 레코드를 실패로 거부**.
  - 예 — `x <= NOW()`(삽입값 `x` 가 미래일 수 없음), `x BETWEEN 1901 AND 2000`(20세기여야 함).
- **무결성 제약 (integrity constraints)**
  - **Normalizer 패턴으로 모델링한 트랜잭션 데이터베이스** 의 일부인 경우가 많음.
  - 한 테이블에 있는 값이 **다른 테이블에 실제로 존재하는 값을 참조** 함을 보장.
  - 예 — 웹사이트 방문이 `pages` 테이블에 없는 페이지를 참조하면 무결성 제약이 깨지고,
    그 방문 row 는 `visits` 테이블에 추가되지 않음.

**데이터베이스 밖에서도 만나는 패턴**

- **테이블 파일 포맷** — Delta Lake 는 지정한 조건에 대해 각 값을 검증하는 **`CHECK` 연산자** 를 포함.
- **직렬화 포맷** — Apache Avro · Apache Protobuf 도 이 패턴을 구현.
  **타입 제약을 기본 제공** 하고, **추가 확장을 설치하면 값 제약** 까지 커버 가능.

Constraints Enforcer 패턴은 **컨슈머 (조회하는 쪽)에게는 정보 제공적(informative)** —
데이터셋의 형태와 가능한 값을 정의해 주므로. 그리고 **프로듀서에게는 상호작용적(interactive)** —
기대하는 검증 제어를 통과하지 않으면 레코드를 추가하지 못하게 막으므로.

#### 고려사항 (Consequences)

Constraints Enforcer 는 좋은 데이터 품질을 확보하는 **확실한 방법**.
데이터 검증 로직을 직접 쓰는 것보다 **간단** 하지만 단점도 있음.

- **All-or-nothing semantics (전부 아니면 전무 시맨틱)**
  - 데이터베이스 수준에 정의한 제약은 대개 **트랜잭션의 all-or-nothing 시맨틱을 따름**.
    적재하는 데이터셋의 **입력 row 중 하나라도 규칙을 어기면 어떤 row 도 받아들여지지 않음**.
  - 게다가 **데이터베이스는 흔히 첫 번째로 만난 오류에서 멈춤**.
    프로듀서로서 이슈가 여럿 있는 데이터셋을 만들었다면,
    **모든 문제를 알아내기 위해 데이터베이스와 여러 번 왕복** 해야 함.
  - **완화책** — 전체 이슈 목록을 한 번에 뽑고 싶다면 **프로듀서 쪽에 검증 규칙을 구현** 할 수 있음.
    ⚠ 다만 그렇게 하면 **앞서 말한 정보 제공적·상호작용적 이점을 잃음**.
- **Data producer shift (프로듀서 쪽으로 쏠림)**
  - 이 패턴은 **제약을 데이터 작성자에게 노출** 하므로 **프로듀서 지향적**.
  - 그런데 **컨슈머마다 데이터에 대한 기대가 다를 수 있음**.
    예 — 데이터베이스에서 nullable 인 필드가 **어떤 컨슈머에게는 필수** 일 수 있음.
    그 결과 컨슈머 (받아 쓰는 쪽)는 **이미 제약이 걸린 데이터셋 위에 또 검증·필터 로직** 을 얹어야 할 수 있음.
- **Constraints coverage (제약 커버리지)**
  - **모든 검증 규칙을 커버하는 것이 언제나 가능하지는 않음.**
    특히 **테이블 파일 포맷** 은 예컨대 **무결성 제약을 다루지 못할 수 있음**.
  - **AWAP 의 제약이 더 유연** — 유일한 한계가 **내가 쓰는 프로그래밍 언어** 이기 때문.
  - ⇒ 결과적으로 **데이터베이스 제약을 데이터 처리 잡에 정의한 제약으로 보완** 해야 할 수 있음.

```
[all-or-nothing 이 만드는 왕복]
──────────────────────────────────────────────────────────────────────
 INSERT 10,000 row  ─► DB: visit_id 가 NULL 인 row 발견 ─► 전체 롤백  ✗ 0건 적재
   프로듀서가 고침 ─► 재시도
 INSERT 10,000 row  ─► DB: event_time 이 미래인 row 발견 ─► 전체 롤백 ✗ 0건 적재
   프로듀서가 고침 ─► 재시도
 INSERT 10,000 row  ─► DB: page_id 가 pages 에 없음 ─► 전체 롤백      ✗ 0건 적재
──────────────────────────────────────────────────────────────────────
 ⇒ 이슈 3개를 알아내는 데 왕복 3회. 각 왕복이 전체 적재 시간만큼 걸림.
 ⇒ 완화 — 프로듀서 쪽에서 (AWAP 2차 감사처럼) 전체 오류를 모아 한 번에 보고할 것.
   대신 DB 제약이 주던 "informative · interactive" 이점은 그만큼 옅어짐.
```

#### 구현 예시 (Examples)

**예시 1 — Delta Lake 의 세 가지 제약 (Example 9-6)**

타입 제약과 널 허용 제약을 가진 테이블을 만들고,
`event_time` 값이 **항상 과거** 임을 보장하는 값 제약을 정의.

```sql
CREATE TABLE default.visits (
 visit_id STRING NOT NULL,          -- 타입 제약 + 널 허용 제약
 event_time TIMESTAMP NOT NULL
) USING delta;

ALTER TABLE default.visits ADD CONSTRAINT
  event_time_not_in_the_future CHECK (event_time < NOW() + INTERVAL "1 SECOND")
  -- 값 제약. 1초 여유를 둬 클럭 스큐로 인한 오탐을 피함
```

이후 삽입되는 row 중 하나라도 규칙을 어기면
`DELTA_VIOLATE_CONSTRAINT_WITH_VALUES` 또는 `DELTA_NOT_NULL_CONSTRAINT_VIOLATED` 오류가 발생.
그 결과 **해당 트랜잭션에서 추가된 어떤 레코드도 테이블에 쓰이지 않음**.

**예시 2 — Protobuf + protovalidate (Example 9-7)**

제약을 쓸 수 있는, 아마 더 놀라운 자리가 **직렬화 파일 포맷**.
Protobuf 라이브러리는 **타입 제약을 기본 구현** 하고, **`protovalidate`** 를 설치하면
**값 제약** 까지 범위를 넓힐 수 있음.

```protobuf
message Visit {
  string visit_id = 1 [(buf.validate.field).string.min_len = 5];   // 최소 길이
  google.protobuf.Timestamp event_time = 2 [
    (buf.validate.field).timestamp.lt_now = true,                  // lt_now = 현재보다 이전
    (buf.validate.field).required = true];
  string user_id = 3 [(buf.validate.field).required = true];
  string page = 4 [(buf.validate.field).cel = {                    // CEL 표현식으로 임의 규칙
    message: "Page cannot end with an html extension"
    expression: "this.endsWith('html') == false"
  }, (buf.validate.field).required = true];
}
```

**예시 3 — 제약 위반 시의 `ValidationError` (Example 9-8)**

visit 클래스 인스턴스에 `validate(...)` 를 호출하고 규칙 중 하나를 (의도했든 아니든) 어기면:

```
Traceback (most recent call last):
  File "...visits_generator.py", line 39, in <module>
    validate(visit_to_send)
  File "...protovalidate/validator.py", line 61, in validate
    raise ValidationError(msg, violations)
protovalidate.validator.ValidationError: invalid Visit
```

> 이 예시가 보여 주는 것 — **제약은 DB 에만 사는 것이 아님**.
> Kafka 로 보내기 **직전**, 즉 **프로듀서 프로세스 안에서** 이미 막을 수 있음.

#### 비교 — #59 AWAP 과 #60 Constraints Enforcer

| 축 | #59 AWAP | #60 Constraints Enforcer |
|---|---|---|
| 검증이 사는 곳 | 내 파이프라인 코드 | DB · 테이블 포맷 · 직렬화 스키마 |
| 표현력 | 프로그래밍 언어가 허용하는 전부 (볼륨 급감·분포 변화 등) | DB 가 지원하는 제약 종류까지 |
| 오류 보고 | 전체 이슈를 모아 한 번에 | 대개 첫 오류에서 중단 |
| 실패 단위 | 내가 정함 (fail / dispatch / nonblocking) | 대개 트랜잭션 전체 (all-or-nothing) |
| 대상 | 프로듀서 = 나 | 프로듀서 = 데이터를 쓰는 모든 주체 |
| 부담 | 구현 부담이 내 쪽 | 선언만 하면 DB 가 강제 |

<details>
<summary><b>⚠ 트러블 로그</b> — 대량 적재 테이블에 CHECK 제약만 걸어 두면 한 건 때문에 배치 전체가 0건 적재로 끝남.</summary>
<div markdown="1">

**예 —** `visits` 에 `CHECK (event_time < NOW())` 를 걸어 둔 상태에서,
모바일 SDK 의 **클럭 스큐로 3초 미래인 이벤트 12건** 이 섞여 들어옴.
2,400만 건 배치가 **전부 롤백** 되어 그날 파티션이 통째로 비고,
다음 날 백필까지 대시보드가 빈 채로 방치됨.

**권장 —** ① 제약에 **현실적인 허용 오차**(`NOW() + INTERVAL '1 SECOND'` 처럼)를 둘 것.
② 그리고 DB 제약만 믿지 말고, **AWAP 2차 감사에서 dispatch 전략** 으로
위반 row 를 격리 테이블에 보내 **나머지는 살릴 것**.

</div>
</details>

---

## 2. 스키마 일관성 (Schema Consistency)

`#60 Constraints Enforcer` 에서 본 **스키마 제약** 은 데이터 정합성 문제를 해결.
그러나 **스키마는 데이터 엔지니어링에서 특별한 위치** 를 차지하고,
그 문제는 **테이블의 필드 타입을 정의하는 것보다 훨씬 복잡**.

```
[제약이 막는 것 vs 못 막는 것]
──────────────────────────────────────────────────────────────────────
 #60 Constraints Enforcer 가 막는 것
   visit_id 에 NULL 이 들어옴                    ✗ 거부
   event_time 이 미래                            ✗ 거부
 못 막는 것 — 스키마 자체가 바뀌는 경우
   프로듀서가 visit_id 컬럼을 DROP               ⚠ 제약도 함께 사라짐
   프로듀서가 user_id 를 STRING → LONG 으로 변경  ⚠ 컨슈머 파싱이 깨짐
   ⇒ 값이 아니라 "계약(contract)" 이 바뀐 것 ⇒ #61 Schema Compatibility Enforcer 의 영역
──────────────────────────────────────────────────────────────────────
```

---

### 2-1. 패턴 #61: 스키마 호환성 적용자 (Schema Compatibility Enforcer)

> 데이터셋은 값이 시간에 따라 바뀔 수 있으므로 동적이고, `#60 Constraints Enforcer` 는
> 그렇게 진화한 엔트리를 미리 정의한 규칙에 비춰 검증.
> 그런데 **스키마 자체에도 이런 검증을 걸 수 있다면** 어떨까.

#### 상황 (Problem)

**책의 use case** — 상대 팀이 "안 쓰는 줄 알고" 필드를 지움:

- **Stateful Sessionizer 패턴** 으로 구현한 **세션화 잡** 을 운영 중. 몇 달간 훌륭하게 돌았음.
- 그런데 **입력 데이터를 생성하는 팀이 여러 변경을 가했고**, 그 결과
  **지난 한 달 동안 잡이 여러 번 실패**.
- 알고 보니 **새로 온 팀이 내 애플리케이션이 쓰던 필드들을 "쓸모없다" 고 판단해 제거** 한 것.
- 새 동료들과 논의한 뒤, **스키마를 깨는 변경을 애초에 막는 솔루션** 을 만들어 달라고 요청.
- **결정적 제약**: 제거된 필드는 프로듀서 입장에서 **자기 코드에서는 정말로 안 쓰는 필드**.
  **컨슈머 (다운스트림)의 사용 여부를 프로듀서가 알 방법이 없음** ⇒ 사람의 선의가 아니라 **도구가 막아야 함**.

#### 해결 (Solution)

프로듀서로서 **깨는 변경(breaking change)을 도입하지 않도록** 보장하려면
**Schema Compatibility Enforcer 패턴** 을 쓸 수 있음.
데이터 스토어에 따라 **세 가지 강제 모드** 중 하나를 사용.

```
[Figure 9-3 재현] 스키마 호환성 강제 모드 세 가지
────────────────────────────────────────────────────────────────────────────────────────────
 ① Mode with an external service — 외부 서비스/라이브러리
                      Validate schema                      Write if schema compatible
    ┌───────────────┐                  ┌─────────────────┐                        ┌────────┐
    │ Data provider │─────────────────►│ Schema registry │───────────────────────►│ Output │
    └───────▲───────┘                  └────────┬────────┘                        └────────┘
            │                                   │
            └───────────────────────────────────┘
                     Fail if schema is not compatible

 ② Implicit with inserts — 삽입 시 암묵적
                      Write if schema compatible
    ┌───────────────┐                                    ┌────────┐
    │ Data provider │───────────────────────────────────►│ Output │
    └───────────────┘                                    └────────┘

 ③ Event-driven for DDL — DDL 이벤트 기반
                      Evolve if schema compatible
    ┌───────────────────────┐                            ┌────────┐
    │ Schema evolution query│───────────────────────────►│ Output │
    └───────────────────────┘                            └────────┘
────────────────────────────────────────────────────────────────────────────────────────────
 ① 만 "호환성 모드를 명시적으로 설정" 할 수 있음. ②③ 은 강제는 하되 모드 선택지가 없음.
```

- **① 외부 서비스 또는 라이브러리 경유 (via an external service or library)**
  - **Apache Kafka 의 Schema Registry** 가 쓰는 방식 — 프로듀서·컨슈머가 통신하는 **API 를 노출**.
  - Schema Registry 는 **각 스키마를 버전 관리** 하고, **설정된 호환성 규칙에 비춰 스키마 변경을 검증**.
  - 서비스 대신 **라이브러리** 를 쓸 수도 있음. 예를 들어 Apache Avro 의 **`SchemaValidator` 클래스** 로
    스키마에 비호환 변경이 없음을 검증 가능.
    ⚠ 다만 **이 라이브러리는 호환성 규칙 자체를 설정할 수는 없음**.
- **② 삽입과 함께 암묵적으로 (implicit with inserts)**
  - **테이블 파일 포맷이나 관계형 데이터베이스** 의 강제 방식.
  - 새 테이블을 만들 때 **널 허용·타입·허용 범위 같은 제약을 정의** 하는데,
    **그와 동시에 제약을 지키지 않는 레코드의 기록을 막는 호환성 모드가 암묵적으로 설정** 됨.
  - ⚠ 그러나 **명시적인 스키마 호환성 모드를 정의할 방법은 없음** —
    그 구현은 외부 서비스·라이브러리에 기대야 함.
- **③ DDL 에 대한 이벤트 기반 (event driven for DDL)**
  - 암묵적 모드를 확장한 접근.
  - **PostgreSQL · SQL Server** 같은 일부 관계형 DB 에서는,
    **`DROP COLUMN`·`RENAME COLUMN` 같은 DDL 연산을 커밋하기 전에 SQL 함수를 실행하는
    이벤트 트리거** 를 걸 수 있음.
  - 함수 로직에 **스키마 강제 규칙** 을 넣고, 사용자가 비호환 변경을 시도하면 **연산을 롤백**.
  - 그렇게 세밀한 제어가 필요 없다면, **`ALTER TABLE` 권한을 아예 부여하지 않는 것** 으로
    모든 스키마 변경을 막을 수도 있음.

**호환성 모드 (compatibility modes)**

스키마 호환성 모드는 **컨슈머 (다운스트림)에게 어떤 진화에 대비해야 하는지 알려 줌**.
가장 흔한 호환성 시나리오는 **비전이(nontransitive) 규칙** —
**연속한 두 스키마 버전**(version 과 version+1, 또는 version 과 version−1)만 호환되면 되는 것.

- **하위 호환 (backward compatibility)**
  - **새 스키마를 쓰는 컨슈머가 옛 스키마로 생성된 데이터를 여전히 읽을 수 있음.**
  - 예 — 새 스키마에 **새 선택적(optional) 필드** 가 생겼다면,
    컨슈머는 새 스키마를 쓰므로 옛 스키마로 만들어진 레코드에서는 **그 필드가 단순히 비어 있음**.
- **상위 호환 (forward compatibility)**
  - **옛 스키마를 쓰는 컨슈머가 새 스키마로 생성된 데이터를 읽을 수 있음.**
  - 예 — 옛 스키마에 있던 **선택적 필드가 새 스키마에서 삭제** 됐다면,
    새 스키마로 생성된 레코드에는 그 속성이 없음. 컨슈머는 값이 비어 있는 것을 보게 되지만,
    **그 필드가 optional 로 표시돼 있었으므로 "비어 있을 수 있음" 은 이미 계약의 일부**.
- **완전 호환 (full compatibility)**
  - 하위·상위 호환을 **섞은 것** — 새 스키마 컨슈머가 옛 데이터를 읽을 수 있고,
    옛 스키마 컨슈머도 새 데이터를 읽을 수 있음.

호환성은 **전이적(transitive)** 일 수도 있음.
이는 **모든 과거(하위)·미래(상위) 스키마 사이의 호환성이 보장돼야 함** 을 뜻함.

```
[Table 9-1 재현] 스키마 호환성 동작 요약
──────────────────────────────────────────────────────────────────────────────────────────
 Compatibility modes      | Allowed actions       | Semantics
 -------------------------+-----------------------+--------------------------------------
 Backward nontransitive   | Delete field          | 새 버전 컨슈머가 옛 버전이 만든
 Backward transitive      | Add optional field    | 데이터를 읽을 수 있음
 -------------------------+-----------------------+--------------------------------------
 Forward nontransitive    | Add field             | 옛 버전 컨슈머가 새 버전이 만든
 Forward transitive       | Delete optional field | 데이터를 읽을 수 있음
 -------------------------+-----------------------+--------------------------------------
 Full nontransitive       | Add optional field    | 새 버전 컨슈머가 옛 버전이 만든
 Full transitive          | Delete optional field | 데이터를 읽을 수 있고,
                          |                       | 옛 버전 컨슈머도 새 버전이 만든
                          |                       | 데이터를 읽을 수 있음
──────────────────────────────────────────────────────────────────────────────────────────
 ⚠ 전이·비전이의 "허용 동작" 이 똑같다는 점이 헷갈리는 지점. 차이는 "몇 개 버전에 대해
   그 규칙을 검사하느냐" 뿐 — 비전이는 직전 버전 하나, 전이는 과거 모든 버전.
 ※ 호환성 모드에는 "None" 도 있지만, 아무것도 강제하지 않으므로 이 패턴에서는 쓰지 않음.
```

**전이 vs 비전이 — 하위 호환 예시 (Example 9-9 ~ 9-11)**

```
[전이 하위 호환이 깨지는 순간]
──────────────────────────────────────────────────────────────────────
 Schema Order (v0):                 Schema Order (v1):
   order_id LONG REQUIRED             order_id LONG REQUIRED
                                      amount DOUBLE DEFAULT 0.0   ← optional 로 추가

 Schema Order (v2):
   order_id LONG REQUIRED
   amount DOUBLE REQUIRED            ← 프로덕트 팀 요청으로 기본값 제거

 비전이(nontransitive) 관점:  v2 컨슈머 ─► v1 데이터 읽기
   v1 프로듀서가 amount 를 빠뜨려도 스키마 정의의 DEFAULT 0.0 이 채워 줌       ✓ 호환
 전이(transitive) 관점:      v2 컨슈머 ─► v0 데이터 읽기
   v0 에는 amount 필드 자체가 없고 채울 기본값도 없음                          ✗ 비호환
──────────────────────────────────────────────────────────────────────
 ⇒ 같은 v1→v2 변경이 비전이에서는 통과, 전이에서는 거부됨.
 ⇒ 전이를 켜 두면 안전하지만 "필드 제거·이름 변경" 이 영영 불가능해짐 (#62 Schema Migrator 의 전제).
```

호환성 모드와 스키마를 정의하고 나면, 프로듀서는 **스키마를 바꿀 때마다 이 외부 강제 컴포넌트와 상호작용**.
그래서 **생성된 데이터가 호환성 모드가 지원하지 않는 진화를 담고 있으면 거부됨**.

#### 고려사항 (Consequences)

이점이 위험을 능가하긴 하지만 유념할 점이 있음.

- **Interaction overhead (상호작용 오버헤드)**
  - 스키마 관리, 특히 **외부 스키마 레지스트리 컴포넌트를 경유하는 방식** 은
    **데이터 생성에 추가 오버헤드** 를 더함.
  - 프로듀서는 **레코드를 가장 최신 스키마 버전에 비춰 검증** 해야 함.
- **Schema evolution (스키마 진화)**
  - 이 패턴을 쓰면 **스키마 진화가 더 어려워짐**.
    어떤 스키마 변경이든 **데이터셋에 정의된 스키마 호환성 수준에 부합** 해야 함.
  - 그 결과 **필드 이름을 바꾸는 일이 "새 필드를 추가하고 이전 필드를 deprecate 하는 일"** 이 되기도 함.
  - 다만 그것이 **더 신뢰할 만한 데이터를 얻기 위해 치르는 값**.
    이 측면은 **#62 Schema Migrator** 에서 더 다룸.

#### 구현 예시 (Examples)

**예시 1 — Kafka Schema Registry 에 스키마 등록 (Example 9-12)**

스키마 호환성 강제를 데이터 엔지니어에게 대중화시킨 도구.
먼저 **스키마와 그 호환성 모드를 정의** — 여기서는 **상위 호환(forward)** 으로 설정.

```json
{"type": "record", "namespace": "com.waitingforcode.model","name": "Visit",
"fields": [
 {"name": "visit_id", "type": "string"},
 {"name": "event_time", "type": "int", "logicalType": "time"}
]}
```

**예시 2 — 호환성 오류 메시지 (Example 9-13)**

새 프로듀서가 **`visit_id` 필드 없이** 레코드를 만들려 한다고 하면,
이제 레코드를 쓰는 일이 **Schema Registry 에 대한 스키마 검증을 수반** 하므로 연산이 실패.

```
confluent_kafka.avro.error.ClientError: Incompatible Avro schema:409 message:
  {'error_code': 409, 'message': 'Schema being registered is incompatible with
  an earlier schema for subject "visits_forward-value",
  details: [{errorType:\'READER_FIELD_MISSING_DEFAULT_VALUE\',
  description:\'The field \'visit_id\' at path \'/fields/0\' in
  the old schema has no default value and is missing in the new schema\',
  ...
```

> 오류가 짚는 지점 — **옛 스키마의 `visit_id` 에 기본값이 없는데 새 스키마에서 사라졌음**.
> 즉 "필드를 지우려면 optional 이었어야 한다" 는 `Table 9-1` 의 규칙이 그대로 강제된 것.

**예시 3 — Delta Lake 의 암묵적 강제 (Example 9-14 · 9-15)**

작업 대상 테이블이 아래 컬럼들로 만들어져 있음:

```
root
|-- visit_id: string (nullable = true)
|-- page: string (nullable = true)
|-- event_time: long (nullable = true)
```

이제 프로듀서가 **`ad_id` 라는 추가 컬럼** 을 붙인다고 하면,
**Delta Lake 는 내 허락 없이 스키마를 수정하지 않으므로** 이 변경을 현재 스키마와 비호환으로 감지:

```
pyspark.errors.exceptions.captured.AnalysisException: A schema mismatch detected when
writing to the Delta table
...

Table schema:
root
-- visit_id: string (nullable = true)
-- page: string (nullable = true)

Data schema:
root
-- visit_id: string (nullable = true)
-- page: string (nullable = true)
-- ad_id: string (nullable = true)
```

#### 비교 — 세 가지 강제 모드

| 모드 | 대표 구현 | 호환성 모드 설정 | 막는 시점 | 한계 |
|---|---|---|---|---|
| **외부 서비스/라이브러리** | Kafka Schema Registry, Avro `SchemaValidator` | 가능 (backward/forward/full × 전이) | 스키마 등록 시 | 레지스트리 상호작용 오버헤드 · 라이브러리는 규칙 설정 불가 |
| **삽입 시 암묵적** | Delta Lake, 관계형 DB | 불가 — 제약 정의가 곧 모드 | 쓰기 시 | 명시적 모드 선택 불가 |
| **DDL 이벤트 기반** | PostgreSQL · SQL Server 이벤트 트리거 | 함수 로직으로 직접 구현 | DDL 커밋 직전 | 구현 부담 · DB 종속 |

<details>
<summary><b>⚠ 트러블 로그</b> — 호환성 모드를 정하지 않은 채 "스키마 레지스트리를 붙였다" 고 안심하면 아무것도 막히지 않음.</summary>
<div markdown="1">

**예 —** Schema Registry 를 붙였지만 subject 별 호환성이 전역 기본값 그대로였음.
프로듀서가 `user_id` 를 `string` → `long` 으로 바꾼 스키마를 등록했는데도 통과되어,
세션화 잡이 **`AvroTypeException: Found long, expecting string`** 으로
컨슈머 그룹 전체가 재시작 루프에 빠짐. 원인 파악에 반나절이 걸림.

**반대 함정 —** 그렇다고 처음부터 **`FULL_TRANSITIVE`** 로 잠가 두면,
잘못 지은 필드명 하나 바로잡는 일조차 불가능해져 **60개짜리 비대한 스키마** 로 굳어짐.

**권장 —** subject 마다 호환성 모드를 **명시적으로 선언** 하고,
**컨슈머 (다운스트림)가 여럿이면 `FULL`(비전이)을 기본** 으로 둘 것.
전이 모드는 **정말 과거 전체를 재처리하는 데이터셋에만** 적용할 것.

</div>
</details>

<details>
<summary><b>⚠ 트러블 로그</b> — `mergeSchema` 를 습관적으로 켜 두면 Delta Lake 의 암묵적 강제가 무력화됨.</summary>
<div markdown="1">

**예 —** 초기 개발 때 편하려고 `.option('mergeSchema', 'true')` 를 넣어 둔 잡을 그대로 운영에 올림.
프로듀서가 오타로 `event_time` 대신 **`event_tiem`** 컬럼을 쓰기 시작했는데,
Delta 가 **조용히 새 컬럼을 추가** 해 버림. 원래 `event_time` 은 그날부터 전부 `NULL`,
파티션 기준 집계가 **한 달치 빈 값** 으로 채워진 뒤에야 발견됨.

**권장 —** 운영 잡에서는 `mergeSchema` 를 끄고, 스키마 진화가 필요한 경우에만
**명시적인 `ALTER TABLE` + 리뷰** 를 거칠 것. 자동 병합은 "강제" 가 아니라 "포기".

</div>
</details>

---

`#61 Schema Compatibility Enforcer` 는 **프로듀서가 비호환 변경을 하지 못하게** 막고,
그런 변경이 가능한 환경에서 **컨슈머 (다운스트림)가 중단되지 않게** 보호.
그러나 스키마 관련 문제가 하나 남음 —
**컨슈머를 안전하게 유지하면서도, 필드 타입 진화·이름 변경 같은 파괴적 스키마 변경을 할 수 있게 하려면?**

```
[#61 이 막아 주는 것 vs #62 가 필요한 것]
──────────────────────────────────────────────────────────────────────
 #61 Schema Compatibility Enforcer
   프로듀서가 실수로 visit_id 를 DROP           ✗ 거부  ⇒ 원하는 동작
   프로듀서가 실수로 user_id 타입 변경           ✗ 거부  ⇒ 원하는 동작
 #62 Schema Migrator 가 필요한 상황
   from_page 라는 이름이 나빠서 referral 로 고치고 싶음  ✗ #61 이 거부  ⚠ 원치 않는 동작
   흩어진 login·email·age 를 user 로 묶고 싶음          ✗ #61 이 거부  ⚠
   ⇒ #61 은 "변경의 종류" 만 보므로, 의도한 파괴적 변경과 사고를 구분하지 못함.
   ⇒ 구분해 주는 것은 규칙이 아니라 "유예 기간(grace period)" ⇒ #62 의 영역
──────────────────────────────────────────────────────────────────────
```

---

### 2-2. 패턴 #62: 스키마 마이그레이터 (Schema Migrator)

> 스키마 정확성을 보장하면 프로듀서의 비호환 변경을 막고 컨슈머의 중단도 막아 줌.
> 그런데 **의도적으로 파괴적 변경을 하고 싶을 때** 는 어떻게 할 것인가.

#### 상황 (Problem)

**책의 use case** — 친절하게 필드를 계속 더하다 보니 속성이 60개가 됨:

- 잡이 다운스트림으로 생성하는 **방문 이벤트(visit event)의 구조를 개선** 하려는 중.
- **첫날부터 사용자 친화적이고 싶었기 때문에**, 컨슈머 (다운스트림)를 번거롭게 하지 않으려고
  **새 필드를 계속 추가만 해 왔음**.
- 그 결과 **도메인 관련 필드들이 메시지 전체에 흩어졌고**, 어떤 메시지는 **속성이 60개까지** 감.
  대부분의 용도에는 너무 많고, **도메인을 이해하는 일 자체가 매우 어려워짐**.
- 많은 컨슈머가 **처리의 어려움과 복잡해진 도메인** 을 호소.
  이상적으로는 **관련 속성이 같은 엔티티로 묶이길** 원함 —
  예를 들어 `login` · `email address` · `age` 같은 사용자 관련 속성은
  **`user` 라는 하나의 속성 아래** 있어야 함.
- **기존 스키마를 급진적으로 바꾸고 싶지는 않음** — 그러면 호환성이 깨짐.
  그러나 **속성 구성은 개선하고 싶고, 컨슈머에게는 새 포맷으로 옮길 시간을 주고 싶음**.
- **결정적 제약**: 이 변경은 **사고가 아니라 의도한 파괴적 변경**.
  `#61 Schema Compatibility Enforcer` 는 **변경의 종류만 통제** 하므로 이 요구를 풀 수 없음.

```
[문제의 구조] 친절함이 쌓여 만든 60개짜리 스키마
──────────────────────────────────────────────────────────────────────
 Day 1    visit_id · event_time · page                          속성 3개
   │      "컨슈머 귀찮게 하지 말자" ⇒ 새 필드는 추가만
 Month 6  + user_login + user_email + user_age + ad_id + ...    속성 20개
   │
 Today    + ... 총 60개, 도메인 경계 없이 평평하게 나열           속성 60개
          컨슈머 ─► "login·email·age 는 user 로 묶어 달라"
──────────────────────────────────────────────────────────────────────
 ⚠ 지금 바로 ALTER 하면 컨슈머 쿼리가 즉시 깨짐.
 ⚠ 아무것도 안 하면 스키마는 계속 비대해짐.
 ⇒ 필요한 것은 "옛 것과 새 것이 공존하는 기간".
```

#### 해결 (Solution)

**Schema Migrator 패턴** 은 **스키마 진화(schema evolution)를 가능하게** 함.

> **참고 사항 — 전이 호환성 (Transitive Compatibility)**
> Schema Migrator 는 **스키마 호환성이 전이적(transitive)이지 않을 것** 을 요구.
> 그렇지 않으면 **필드 제거나 이름 변경이 아예 불가능** —
> 전이 호환성 수준은 **모든 버전에 걸친 일관성** 을 보장하기 때문.

**1단계 — 진화 유형 식별**. 세 가지 시나리오가 가능.

- **Rename (이름 변경)**
  - 나 또는 컨슈머가 **어떤 속성의 이름이 잘못됐거나 이해하기 어렵다** 고 판단할 때 발생.
- **Type change (타입 변경)**
  - **문제 진술의 시나리오**.
  - 스키마를 **더 잘 구성** 하고 싶거나 (예: 여러 차례 변경 후 단순화),
    **처리에 최적화** 하고 싶은 경우 (예: 이질적인 날짜/시간 텍스트 속성을 epoch timestamp 로 조정).
- **Removal (제거)**
  - 제거하려는 속성을 **처리하는 컨슈머 (다운스트림)가 없다는 100% 보장** 이 있으면 쉬움.
  - 보장이 없다면 **대체재를 찾아 주거나, 제거 자체를 취소** 해야 함.

**2단계 — rename · type change 처리**. 가장 까다로운 두 시나리오는 절차가 같음.

- 먼저 **이름을 바꾼(또는 타입을 바꾼) 속성을 담은 새 필드를 생성**.
- 다음으로 **컨슈머와 전환 기간(transition time)을 합의**.
- 그 기간 동안 컨슈머는 **이전 속성과 새 속성을 동시에 수신**.
- **데드라인에 도달한 뒤에야** 수정된 버전의 속성만 담은 **새 스키마 버전을 생성**.

**3단계 — removal 처리**. 제거는 조금 다름 —
**필드 제거 기간(field removal period)에 대해 컨슈머와 합의** 하고,
데드라인이 지나면 **삭제된 속성이 없는 새 스키마 버전을 생성**.

```
[Schema Migrator 타임라인] from_page → referral 이름 변경
──────────────────────────────────────────────────────────────────────
 T0  스키마 v1        from_page                          컨슈머 전원 v1 사용
       │ 새 필드 추가 (기존 필드는 그대로 둠)
 T1  스키마 v2        from_page  +  referral             ⚠ 두 컬럼 동시 존재
       │             ← 전환 기간 (grace period) →         레코드 크기 증가 구간
       │             컨슈머가 하나씩 referral 로 이전
 T2  데드라인          모든 컨슈머 이전 완료 확인          (#70 Fine-Grained Tracker 로 검증)
       │ 옛 필드 제거
 T3  스키마 v3                      referral             ✓ 깔끔한 최종 스키마
──────────────────────────────────────────────────────────────────────
 ⚠ 이 패턴의 전부는 T1~T2 구간 — "동시에 둘 다 보낸다".
 ⚠ T2 를 정하지 않으면 T1 상태가 영구화되어 스키마가 계속 비대해짐.
```

> **참고 사항 — 데이터 계보 (Data Lineage)**
> 어떤 속성이 컨슈머 (다운스트림)에 의해 **실제로 사용되는지 탐지** 하려면
> 챕터 10 의 **Fine-Grained Tracker 패턴(#70)** 에 기댈 수 있음.

#### 고려사항 (Consequences)

Schema Migrator 는 **스키마 마이그레이션을 위한 유예 기간에 의존**.
그 기간 동안 **옛 스키마가 여전히 유효** 하고 컨슈머가 처리할 수 있음. 이는 **데이터 크기에 영향**.

- **Size impact (크기 영향)**
  - 마이그레이션에 안전 장치를 제공하는 대신, **스토리지 공간 · 네트워크 전송 · I/O** 형태의 비용이 발생 —
    저장할 데이터가 더 많아지기 때문.
  - 어떤 데이터 포맷은 **필드가 많은 것을 공식적으로 권장하지 않음**.
    예를 들어 **Protobuf 는 "Proto Best Practices" 에서 수백 개 필드 사용을 경고** —
    **채워지지 않은 필드조차 최소 65바이트** 를 차지하기 때문.
    Protobuf 가 생성하는 빌더의 전체 크기가 **Java 같은 언어의 컴파일 한계** 에 도달할 수도 있음.
  - 크기는 **메타데이터·통계 계층에도 영향**.
    집필 시점 기준 **Delta Lake 는 기본적으로 첫 32개 컬럼에 대해서만 통계를 수집**.
    이 값을 바꿀 수는 있지만 **쓰기 시간에 영향** 을 줄 수 있음.
- **Impossible removal (제거가 불가능한 경우)**
  - 필드 제거 시나리오에는 구현상 한계가 있음.
  - 어떤 컨슈머가 그 필드를 쓰고 있다면, **대체 속성을 제공할 수 없는 한 제거는 불가능**.

#### 구현 예시 (Examples)

스키마 마이그레이션 워크플로는 여러 기술에서 동일하므로,
여기서는 **하나의 데이터 포맷** 에 집중해 **Schema Migrator 를 따르지 않으면 무슨 일이 생기는지** 확인.

**예시 1 — 컨슈머의 쿼리 (Example 9-16)**

이 예시의 컨슈머는 **로그인한 사용자의 방문** 을 전용 테이블로 추출 중:

```sql
INSERT INTO dedp.connected_users_visits
 SELECT visit_id, event_time, user_id, page, ip, login, from_page FROM dedp.visits
 WHERE is_connected = true AND from_page IS NOT NULL;
```

**예시 2 — 패턴 없이 바로 rename 했을 때의 오류 (Example 9-17)**

`from_page` 컬럼의 이름이 나쁘고 `referral` 이 더 낫다는 것을 방금 깨달았다고 하자.
**가장 나쁜 선택** 은 이름 변경 연산을 **바로 실행** 하는 것 —
`ALTER TABLE dedp.visits RENAME COLUMN from_page TO referral`.
컨슈머는 **새 데이터를 볼 수조차 없음** — 쿼리가 먼저 실패하기 때문:

```
ERROR: column "from_page" does not exist
LINE 2:    SELECT visit_id, event_time, user_id,    ..
```

**예시 3 — Schema Migrator 방식의 rename (Example 9-18)**

이 문제를 피하려면 **새 컬럼을 먼저 만들어** 이름 변경을 마이그레이션해야 함:

```sql
ALTER TABLE dedp.visits ADD COLUMN referral VARCHAR(25) NOT NULL
```

**이전 컬럼은 컨슈머가 워크로드를 적응시킨 뒤에야** 제거할 수 있음.

> **참고 사항 — Protobuf 의 rename 은 왜 안전한가**
> Protobuf 의 이름 변경 연산은 안전 —
> **인코딩된 버전이 필드 이름을 저장하지 않고 태그(tag)만 저장** 하기 때문.
> 다만 **타입 변경은 안전하지 않을 수 있음** (Protobuf 공식 문서 참고).
> Protobuf 와 Delta Lake 도 비슷한 워크플로를 가지므로 책에서는 생략, GitHub 저장소에서 확인 가능.

<details>
<summary><b>⚠ 트러블 로그</b> — 유예 기간의 "종료일" 을 정하지 않으면 마이그레이션이 영원히 끝나지 않음.</summary>
<div markdown="1">

**예 —** `from_page` → `referral` 마이그레이션에서 새 컬럼만 추가하고 데드라인을 잡지 않았음.
반년 뒤 확인해 보니 **컨슈머 7개 중 4개가 여전히 `from_page`** 를 읽고 있었고,
그사이 같은 방식으로 진행한 마이그레이션이 3건 더 쌓여 **속성이 60개에서 71개** 로 늘어남.
Protobuf 메시지가 비대해지면서 **빈 필드만으로 레코드당 700바이트 이상** 이 낭비됨.

**반대 함정 —** 그렇다고 "안 쓰는 것 같으니 지우자" 며 임의로 DROP 하면
분기 마감 리포트처럼 **한 달에 한 번만 도는 배치** 가 그때서야
`column "from_page" does not exist` 로 터짐.

**권장 —** 새 컬럼을 추가하는 PR 에 **제거 예정일을 티켓으로 함께 등록** 하고,
데드라인 전에 **#70 Fine-Grained Tracker 로 실제 조회 여부를 확인** 한 뒤 제거할 것.

</div>
</details>

<details>
<summary><b>⚠ 트러블 로그</b> — 마이그레이션 컬럼을 테이블 끝에 붙이면 Delta Lake 의 data skipping 이 조용히 죽음.</summary>
<div markdown="1">

**예 —** 이미 컬럼이 30개인 Delta 테이블에 마이그레이션용 컬럼 5개를 뒤에 추가함.
Delta Lake 는 기본으로 **첫 32개 컬럼에만 통계(min/max/nullCount)를 수집** 하므로,
새로 만든 `referral` 컬럼은 **33번째 이후로 밀려 통계 대상에서 제외**.
`WHERE referral = 'newsletter'` 쿼리가 **data skipping 없이 전체 파일을 스캔** 하게 되어
응답이 4초에서 90초로 늘어남. 쿼리는 성공하니 아무도 원인을 알아채지 못함.

**권장 —** 마이그레이션 대상 컬럼은 **필터에 쓰이는지 먼저 확인** 하고,
쓰인다면 `delta.dataSkippingNumIndexedCols` 를 조정하거나
**컬럼 순서를 재배치** 할 것. 조정은 쓰기 시간을 늘리므로 값을 무작정 키우지는 말 것.

</div>
</details>

---

## 3. 품질 관찰 (Quality Observation)

**데이터셋은 동적** 이라는 사실을 기억할 것. 데이터셋은 변하고,
**오늘 정의한 제약 규칙이 내일은 유효하지 않을 수 있음**.
그래서 데이터셋에 무슨 일이 일어나는지 **관찰** 하고,
기존 규칙을 조정하거나 새 제약을 추가할 준비를 해 두는 것이 중요.

```
[관찰 패턴의 자리 — 파이프라인 대비 위치가 곧 패턴의 이름]
──────────────────────────────────────────────────────────────────────
 #63 Offline Observer                #64 Online Observer
 ─────────────────────────────       ─────────────────────────────
 관찰 잡이 사는 곳:                    관찰 잡이 사는 곳:
   별도 파이프라인 (분리)                생성 파이프라인 안 (내장)
 스케줄: 자유 (야간 일괄 등)            스케줄: 데이터 생성과 동시
 인사이트 도달: 늦음                    인사이트 도달: 생성 직후
 생성 파이프라인 영향: 없음              생성 파이프라인 영향: 있음 (지연·실패 전파)
 ⇒ 둘의 차이는 "무엇을 재느냐" 가 아니라 "언제 재느냐" 뿐 — 관찰 로직 자체는 동일.
 ⇒ 선택 기준은 "정확한 시점을 얻기 위해 얼마나 비용을 낼 의향이 있는가".
──────────────────────────────────────────────────────────────────────
```

---

### 3-1. 패턴 #63: 오프라인 옵서버 (Offline Observer)

> 관찰 패턴은 **데이터 파이프라인에서 차지하는 자리** 로 구분 가능.
> 첫 번째 유형은 **데이터 처리 워크플로에 간섭하지 않는 별도의 관찰 컴포넌트** 로 존재.

#### 상황 (Problem)

**책의 use case** — 지금은 멀쩡하지만 이전 프로젝트 경험상 곧 깨질 것을 아는 상황:

- 이번 달에 **새 데이터 파이프라인을 시작** 했고, **데이터 품질 이슈는 많이 겪지 않았음**.
- 데이터셋은 **완전히 구조화** 되어 있고, **모든 비즈니스 규칙이 품질 확보 패턴으로 올바르게 강제** 되는 중.
- 그러나 **이전 프로젝트 경험상 이 상태가 지속되지 않을 것** 을 앎 —
  **업스트림 데이터셋이 앞으로 몇 달간 진화** 할 것이기 때문.
- 그래서 **값의 분포(distribution of values)** 나 **컬럼별 NULL 개수** 같은
  **데이터셋의 속성을 모니터링** 하고 싶음.
- **결정적 제약**: **지금은 모든 것이 정상** 이므로,
  이 **모니터링 계층이 메인 파이프라인을 막아서는 안 됨**.

#### 해결 (Solution)

모니터링이 처리 워크플로를 막지 않아야 하는 시나리오라면 **Offline Observer 패턴** 이 최선.

- 구현은 **데이터 관찰 가능성 잡(data observability job)을 만드는 것** 으로 구성.
  이 잡이 **처리된 레코드를 분석** 하고 **기존 모니터링 계층에 추가 인사이트를 더함**.
- 인사이트는 **비즈니스 맥락에 따라 다르지만** 다음을 포함할 수 있음:
  - **값의 분포**
  - **nullable 필드의 NULL 개수**
  - **입력 데이터셋에 새로 생겼지만 아직 처리되지 않은 필드**
- 그렇게 이 파라미터들을 저장해 두면 **시간에 따른 데이터 품질 이슈를 포착** 할 수 있음.
- 데이터 관찰 잡은 **데이터 생성 프로세스에 영향을 주지 않음**.
  **독립적으로 실행** 되고 **완전히 다른 스케줄로 실행될 수도 있음**.
  예를 들어 모든 데이터 생성기가 낮 시간에 돈다면,
  **자원 경합(resource concurrency)을 피하려고 관찰 잡을 전부 야간에** 스케줄할 수 있음.

```
[Offline Observer — 두 파이프라인이 서로를 모름]
──────────────────────────────────────────────────────────────────────
 [생성 파이프라인]  09:00 ─► 10:00 ─► 11:00 ─► ... ─► 23:00   (매시간)
        │
        └─► visits_output 테이블
                  ▲
                  │ (읽기만 함 — 쓰기 경로에 개입하지 않음)
                  │
 [관찰 파이프라인]                                    02:00     (야간 1회)
        └─► 관찰 상태 기록 ─► 집계 ─► visits_monitoring 테이블 ─► 대시보드
──────────────────────────────────────────────────────────────────────
 ✓ 생성 잡은 관찰 잡의 존재를 모름 ⇒ 관찰이 실패해도 데이터는 정상 배포됨.
 ⚠ 대신 09:00 에 들어온 이상 데이터를 알게 되는 시점은 다음 날 02:00.
```

> **참고 사항 — 관찰 가능성과 감사의 차이 (Observability Versus Auditing)**
> **관찰 가능성은 감사와 같지 않음.**
> **감사(audit)** 는 데이터셋을 **검증** 하고 **차단(blocking) 연산** —
> 이슈를 감지하면 파이프라인을 막음.
> **관찰 가능성(observability)** 은 데이터셋을 **모니터링** 하는 **비차단(nonblocking) 접근** —
> 이슈 감지를 돕지만 **파이프라인의 진행을 막지는 않음**.

#### 고려사항 (Consequences)

데이터 생성과 데이터 관찰을 **분리(decorrelate)하는 것** 은 프로덕션 자원에 영향을 주지 않아 좋음.
안타깝게도 동전의 뒷면이 있음.

- **Time accuracy (시점 정확성)**
  - 오프라인 관찰 잡은 **어떤 스케줄로도 실행 가능** 하고,
    **데이터 생성기보다 훨씬 늦게** 돌 수도 있으므로 **적시에 일어나지 않을 수 있음**.
  - 즉 **인사이트가 너무 늦게 도착** 할 수 있음 —
    **모든 다운스트림 컨슈머가 새 품질 이슈를 담은 데이터셋을 이미 처리** 했을 수 있기 때문.
- **Compute resources (컴퓨트 자원)**
  - 관찰 잡이 옆에서 돌기 때문에 **데이터 생성 잡보다 덜 자주 스케줄하고 싶은 유혹** 이 생김.
    예를 들어 **시간별 배치 처리** 에 대해 관찰 잡은 **24시간에 한 번** 만 실행하는 식.
  - 유효한 접근이지만, **시간 단위 변경분 대신 24시간치를 한꺼번에 처리** 해야 하므로
    **더 많은 컴퓨트 자원이 필요할 수 있음** 을 인지해야 함.
  - 결국 **관찰 대상 데이터셋을 샘플링** 해 일부만 쓰는 것을 고려할 수 있음.
    안타깝게도 부분집합만 뽑아 관찰하면 **흥미로운 관찰을 놓칠 수 있음**.

#### 구현 예시 (Examples)

**예시 1 — Airflow 관찰 파이프라인의 태스크 구성 (Example 9-19)**

데이터 생성 파이프라인과 **다른 스케줄로 실행** 되어, 지금까지 생성된 데이터셋의 품질을 단언하고
통계를 모니터링 계층에 기록:

```python
wait_for_new_data = SqlSensor(...)
record_new_observation_state = PostgresOperator(...)
insert_new_observations = PostgresOperator(...)
wait_for_new_data >> record_new_observation_state >> insert_new_observations
```

**예시 2 — 관찰 상태 기록 쿼리 (Example 9-20)**

처리할 새 데이터가 있을 때마다, 관찰 잡은 **처음·마지막 처리 row 의 ID 를 담은 새 관찰 상태를 기록**.
이 연산은 **멱등성을 위해 필요** —
관찰 대상 테이블에 **row 변경이 생기더라도 분석 범위가 동일하게(따라서 일관되게) 유지** 되도록 보장:

```sql
INSERT INTO dedp.visits_monitoring_state (execution_time, first_row_id, last_row_id)
  SELECT
   '{{ execution_date }}' AS execution_time,
   MIN(id) AS first_row_id, MAX(id) AS last_row_id   -- 이번 관찰 창의 경계를 고정
  FROM dedp.visits_output
  WHERE id > COALESCE(
    (SELECT last_row_id FROM dedp.visits_monitoring_state WHERE
      execution_time = '{{ prev_execution_date }}'::TIMESTAMP),  -- 직전 실행이 어디까지 봤는지
    0
  )
```

**예시 3 — 데이터 관찰 쿼리 (Example 9-21)**

이후 관찰 파이프라인은 **고정해 둔 first/last row ID 위에서 집계를 수행** 해 관찰 결과를 생성:

```sql
INSERT INTO dedp.visits_monitoring(execution_time, all_rows, invalid_event_time,
 invalid_user_id, invalid_page, invalid_context)
 SELECT
  '{{ execution_date }}' AS execution_time,
  COUNT(*) AS all_rows,
  ...
  SUM(CASE WHEN context IS NULL THEN 1 ELSE 0 END) AS invalid_context  -- 컬럼별 NULL 집계
FROM dedp.visits_output
 WHERE id BETWEEN
 (SELECT first_row_id FROM dedp.visits_monitoring_state WHERE
  execution_time = '{{ execution_date }}')
 AND
  (SELECT last_row_id FROM dedp.visits_monitoring_state WHERE
  execution_time = '{{ execution_date }}');
```

**예시 4 — 스트리밍 파이프라인의 Offline Observer (Example 9-22)**

스트리밍에서도 구현 가능. 배치와 마찬가지로 **처리된 데이터 위에서 도는 별도의 잡** 이
관찰 결과를 생성. 여기서는 **프로듀서의 처리 지연** 과 **몇 가지 품질 지표** 를 분석:

```python
visits_to_observe = (input_data_stream
 .selectExpr('CAST(value AS STRING)')
 .select(functions.from_json(functions.col('value'), visit_schema).alias('visit'))
 .selectExpr('visit.*')
 .select('visit_id', 'event_time', 'user_id', 'page', 'context.referral',...)
 )
query = (visits_to_observe.writeStream.foreachBatch(generate_and_write_observations)
.option('checkpointLocation', checkpoint_location).start())
```

**예시 5 — 데이터 프로파일 리포트 생성 (Example 9-23)**

모든 관찰 로직은 `generate_and_write_observations` 함수 안에 있음.
첫 단계에서는 **앞의 Airflow 버전과 동일한 데이터 관찰 쿼리를 실행** 하고,
이어서 **`ydata-profiling` 라이브러리로 HTML 데이터 프로파일 리포트를 생성**:

```python
def generate_profile_html_report(visits_dataframe: DataFrame, batch_version: int):
 profile = ProfileReport(visits_dataframe, minimal=True)
 profile.to_file(f'{base_dir}/profile_{batch_version}.html')
```

생성된 HTML 페이지는 **관찰 대상 데이터셋의 특성** 을 기술하며,
이를 근거로 **강제(enforcement) 단계의 품질 규칙을 추가·수정·삭제** 할 수 있음.

```
[Figure 9-4 재현] 관찰 대상 데이터셋의 데이터 프로파일 (ydata-profiling)
──────────────────────────────────────────────────────────────────────────────
 Dataset statistics                      | Variable types
 ----------------------------------------+-------------------------
 Number of variables       14            | Categorical      12
 Number of observations    1830          | DateTime          2
 Missing cells             4349          |
 Missing cells (%)         17.0%         |
──────────────────────────────────────────────────────────────────────────────
 Variables
 ------------------------------------------------------------------------------
 visit_id   (Categorical, MISSING)      | event_time  (Date, MISSING)
   Distinct           27                |   Distinct           393
   Distinct (%)       1.5%              |   Distinct (%)       23.9%
   Missing            183               |   Missing            183
   Missing (%)        10.0%             |   Missing (%)        10.0%
   Memory size        0.0 B             |   Minimum   2024-01-01 01:00:00
                                        |   Maximum   2024-01-01 06:08:00
──────────────────────────────────────────────────────────────────────────────
 이 리포트가 보여 주는 것 — 전체 셀의 17%가 결측이고, visit_id 조차 10%가 비어 있음.
 ⇒ "visit_id 는 NOT NULL 이어야 한다" 는 제약이 빠져 있었다는 신호 ⇒ 강제 규칙을 갱신.
 ⚠ Distinct 가 1830건 중 27개뿐이라는 점도 함께 읽어야 함 (식별자 후보인데 중복이 많음).
```

**예시 6 — 지연(lag) 탐지 함수 (Example 9-24)**

지연 탐지는 **체크포인트 위치에 데이터 생성 잡이 마지막으로 커밋한 오프셋** 과
**입력 토픽의 가장 최근 오프셋** 을 비교:

```python
def get_last_offsets_per_partition(self) -> Dict[str, int]:
 last_processed_offsets = self._read_last_processed_offsets()   # 체크포인트에서 읽음
 last_available_offsets = self._read_last_available_offsets()   # 토픽에서 읽음

 offsets_lag = {}
 for partition, offset in last_available_offsets.items():
  lag = offset - last_processed_offsets[partition]              # 파티션별 지연
  offsets_lag[partition] = lag
 return offsets_lag
```

<details>
<summary><b>⚠ 트러블 로그</b> — 관찰 범위를 시각 조건으로 잡으면 재실행할 때마다 숫자가 달라져 추세를 못 읽음.</summary>
<div markdown="1">

**예 —** `record_new_observation_state` 를 생략하고
관찰 쿼리를 `WHERE created_at BETWEEN '{{ execution_date }}' AND ...` 로 작성함.
그런데 관찰 대상 테이블이 늦게 도착한 데이터를 뒤늦게 UPSERT 하는 구조였고,
같은 날짜에 대해 관찰 잡을 재실행하자 **`invalid_user_id` 가 412건에서 87건** 으로 바뀜.
"품질이 좋아졌다" 로 보고했다가 다음 주에 다시 뒤집혀 대시보드 신뢰를 잃음.

**권장 —** 관찰 범위는 **시각이 아니라 row ID 경계로 고정** 하고,
그 경계를 별도 상태 테이블에 먼저 기록할 것. 관찰도 멱등해야 추세가 의미를 가짐.

</div>
</details>

<details>
<summary><b>⚠ 트러블 로그</b> — 관찰 잡을 아끼려고 주기를 늘리면 관찰 잡 자체가 OOM 으로 죽음.</summary>
<div markdown="1">

**예 —** 시간별 생성 잡에 대해 관찰을 24시간 1회로 잡음.
평소엔 문제없었으나 마케팅 캠페인 주간에 하루치가 **8억 건** 으로 불었고,
`ydata-profiling` 이 전체를 메모리에 올리다 **executor OOM** 으로 종료.
정작 품질 이슈를 가장 봐야 할 날에 **일주일 내내 관찰 공백** 이 생김.

**권장 —** 관찰 주기는 생성 주기와 **최대 4배 이내** 로 두고,
그보다 늘려야 한다면 `ProfileReport(minimal=True)` 와 **샘플링을 함께 적용** 할 것.
샘플링은 일부 관찰을 놓치지만, 관찰이 아예 죽는 것보다는 나음.

</div>
</details>

---

### 3-2. 패턴 #64: 온라인 옵서버 (Online Observer)

> **데이터 처리와 데이터 관찰 사이의 지연** 이 문제라면,
> 더 실시간에 가까운 반대 패턴인 **Online Observer** 를 선택할 수 있음.

#### 상황 (Problem)

**책의 use case** — Offline Observer 가 잡긴 잡았는데 사용자보다 늦었음:

- 지난주 **데이터 분석 동료들이 우편번호(zip code) 필드의 예상치 못한 포맷** 을 호소.
- 알고 보니 **업스트림 데이터셋에 데이터 회귀(data regression)** 가 있었고,
  **기존 데이터 신뢰 규칙으로는 막을 수 없었음**.
- **Offline Observer 는 그 이슈를 실제로 발견** 했음.
  그러나 **주 1회 실행** 이므로 **사용자보다 먼저 문제를 감지하지 못함**.
- **결정적 제약**: 앞으로는 **컨슈머 (조회하는 쪽)가 품질 이슈를 먼저 알게 되는 상황을 피하고**,
  **일주일보다 빨리 고칠 수 있어야** 함.

#### 해결 (Solution)

이 문제는 **Offline Observer 한계의 완벽한 예시** 이고,
극복은 반대 패턴인 **Online Observer** 로 비교적 간단.

- Online Observer 도 **관찰 지표를 생성하는 데이터 관찰 잡에 의존** 하는 것은 동일.
  차이는 **실행 시점**.
- Online Observer 의 잡은 **데이터 생성 파이프라인의 본질적인 일부(intrinsic part)** 이고,
  그 결과 **생성된 인사이트가 데이터 생성 직후에 가용**.
  이 접근은 **컨슈머 (다운스트림)와의 커뮤니케이션·기술 이슈를 상당수 예방**.

**어디에 관찰 잡을 둘 것인가** — 데이터 생성기를 ETL/ELT 단계로 생각하면
가장 인기 있는 자리는 **Transform 단계 뒤**.
여기서 **Parallel Split 패턴(#37)** 또는 **Local Sequencer 패턴(#33)** 으로 관찰 잡을 오케스트레이션.

```
[Figure 9-5 재현] 배치 파이프라인에서 Online Observer 를 넣는 두 위치
────────────────────────────────────────────────────────────────────────────────
 Parallel split approach
                                          ┌──────────────────────┐
                                     ┌───►│         Load         │
                                     │    └──────────────────────┘
 ┌─────────┐    ┌───────────┐        │
 │ Extract │───►│ Transform │────────┤
 └─────────┘    └───────────┘        │
                                     │    ┌──────────────────────┐
                                     └───►│ Run the observation  │
                                          │         job          │
                                          └──────────────────────┘

 Local sequencer approach
 ┌─────────┐    ┌───────────┐    ┌──────┐    ┌──────────────────────┐
 │ Extract │───►│ Transform │───►│ Load │───►│ Run the observation  │
 └─────────┘    └───────────┘    └──────┘    │         job          │
                                             └──────────────────────┘
────────────────────────────────────────────────────────────────────────────────
 Parallel split — 관찰과 적재를 동시에 ⇒ 빠르지만 "적재 이후 상태" 는 못 봄.
 Local sequencer — 적재 뒤에 관찰 ⇒ 컨슈머가 실제로 보는 데이터셋을 검증하지만 완료가 늦어짐.
```

**스트리밍의 경우** — 관찰 로직을 **데이터 생성 잡 안으로 통합** 해야 함.
배치 구현과 비슷하게 들리지만 **중대한 차이** 가 있음 —
**관찰 단계를 별도 파이프라인으로 실행할 수 없다는 것**.
결과적으로 **예상치 못한 에러나 메모리 이슈 같은 데이터 관찰 쪽 문제가 잡 전체에 영향**.
**데이터셋을 샘플링** 해 이 위험을 완화할 수 있으나,
그 과정에서 **일부 인사이트를 놓치게 된다는 사실을 받아들여야** 함.

```
[Figure 9-6 재현] 스트리밍 잡의 Online Observer — 관찰이 잡 안으로 들어옴
──────────────────────────────────────────────────────────────────────────────────
                  ┌───────────────────────────────────┐
                  │        Data processing job        │
                  │  ┌─────────────────────────────┐  │      ╭──────────────╮
  ╭──────────╮    │  │ Write processed data to the │  │─────►│    Output    │
  │  Input   │───►│  │       output location       │  │      │   location   │
  │ location │    │  └─────────────────────────────┘  │      ╰──────────────╯
  ╰──────────╯    │  ┌─────────────────────────────┐  │      ╭──────────────╮
                  │  │  Write data observation to  │  │─────►│ Observation  │
                  │  │   the observation location  │  │      │   location   │
                  │  └─────────────────────────────┘  │      ╰──────────────╯
                  └───────────────────────────────────┘
──────────────────────────────────────────────────────────────────────────────────
 ⚠ 두 쓰기가 같은 잡 안에 있음 ⇒ 관찰 쪽 OOM 이 데이터 생성까지 함께 멈춤.
 ⇒ 완화책은 샘플링 — 대신 놓치는 인사이트를 감수.
```

> **참고 사항 — 데이터만이 아님 (Not Only the Data)**
> 이 절은 **처리된 데이터에 관한 관찰 가능성** 을 다루지만,
> 관찰 가능성은 **더 넓은 범위** 를 포괄.
> **CPU · 메모리 · 디스크 사용량** 같은 **기술 메타데이터** 도 포함.
> 대부분 **준실시간(near real-time) 측정** 이므로 **Online Observer 패턴에서 가용**.

#### 고려사항 (Consequences)

Online Observer 는 Offline Observer 의 **시점 정확성 문제를 해결** 하지만 함정이 있음.

- **Extra delays (추가 지연)**
  - **Local Sequencer 방식** 으로 관찰 잡을 통합하면 **파이프라인 끝에 단계가 하나 더 붙음**.
  - 당연히 **파이프라인 완료가 지연** 됨.
    메인 워크플로에 이 모니터링 단계를 추가하는 일이 **공짜가 아니라는 점** 을 기억할 것.
- **Parallel splits (병렬 분할)**
  - **Parallel Split 방식** 은 **관찰과 적재를 동시에 실행** 해 병렬성을 더함.
    그러나 **부분적으로만 유효한 데이터셋을 관찰할 위험** 을 함께 들여옴.
  - 책의 예 — **날짜/시간 속성을 가진 데이터셋을 데이터베이스에 적재** 하는 상황.
    **날짜/시간 포맷이 DB 가 기대하는 것과 다르면 DB 에는 그 값들이 누락**.
    그런데 **관찰 단계는 이 이슈를 보지 못함**.
  - **완화책 1** — **Local Sequencer 방식** 을 써서 **컨슈머 (다운스트림)에게 실제로 노출되는 데이터셋** 을 관찰.
  - **완화책 2** — 관찰 범위를 **처리는 되었지만 노출되지 않은 데이터셋** 으로 한정.
    이 논리에서는 관찰이 **적재 태스크 대신 변환(transformation)에 초점**.
    다만 **지금은 적재 이슈가 없더라도 이 전략이 파이프라인 전체 수명 동안 적절하지 않을 수 있음**.

#### 구현 예시 (Examples)

관찰 코드 자체는 **Offline Observer 와 동일** 하므로 생략.
대신 **오프라인 관찰 코드를 더 반응적인 온라인 코드로 바꾸는 방법** 을 확인.

**예시 1 — Airflow 배치 파이프라인에 관찰을 병합 (Example 9-25)**

배치 파이프라인이 이제 **데이터 관찰 단계를 데이터 처리 파이프라인에 통합**:

```python
wait_for_new_data = SqlSensor(...)
record_new_synchronization_state = PostgresOperator(...)
clean_previously_added_visits = PostgresOperator(...)
copy_new_visits = PostgresOperator(...)
record_new_observation_state = PostgresOperator(...)   # Example 9-19 의 관찰 태스크가
insert_new_observations = PostgresOperator(...)        # 같은 DAG 안으로 들어옴

wait_for_new_data >> record_new_synchronization_state
  >> clean_previously_added_visits >> copy_new_visits
copy_new_visits >> record_new_observation_state >> insert_new_observations
```

**관찰 실패가 파이프라인 실패가 되는 위험** 을 어떻게 다룰 것인가.
**아무 일도 하지 않는 최종 태스크를 추가** 하고, 이 태스크가 **관찰 잡과 독립적으로 트리거** 되게 함.
그러면 **관찰이 실패해도 이 태스크의 실행이 파이프라인을 성공으로 표시**.
Apache Airflow 에서는 **트리거 규칙을 `all_done` 으로 설정** 해 달성.

```
[관찰 실패를 파이프라인 실패로 만들지 않기]
──────────────────────────────────────────────────────────────────────
 copy_new_visits ✓
      ├─► record_new_observation_state ─► insert_new_observations  ✗ 실패
      │
      └──────────────────────────────────────────────────► final_task
                                                            trigger_rule = all_done
                                                            ⇒ DAG 는 ✓ success
──────────────────────────────────────────────────────────────────────
 ⇒ 데이터는 이미 배포됐으므로, 관찰이 죽었다고 배포까지 실패로 되돌릴 이유가 없음.
 ⚠ 대신 관찰 태스크의 실패는 별도 알림 채널로 반드시 드러낼 것 (조용한 관찰 공백 방지).
```

**예시 2 — 스트리밍의 lag 탐지 적응 (Example 9-26)**

스트리밍에서도 코드가 병합되는데, **두 가지 큰 변화** 를 수반 — **lag 탐지** 와 **accumulator**.
데이터 리더가 이제 **파티션 번호** 와 **오프셋 위치** 두 컬럼을 더 포함하고,
lag 탐지기는 이를 이용해 **마이크로배치에서 가장 최근에 처리된 레코드** 를 얻음:

```python
@dataclasses.dataclass
class PartitionWithOffset:
 partition: int
 offset: int


class PartitionToMaxOffsetAccumulatorParam(AccumulatorParam):
 def zero(self, default_max: PartitionWithOffset):
  return []

 def addInPlace(self, partitions_with_offsets: List[PartitionWithOffset],
    new_max_candidate: PartitionWithOffset):
  partitions_with_offsets.append(new_max_candidate)
  return partitions_with_offsets

def write_to_kafka_with_observer(visits: DataFrame, batch_number: int):
 ctx = visits_to_analyze.sparkSession.sparkContext
 max_offsets_tracker = ctx.accumulator([], PartitionToMaxOffsetAccumulatorParam())

 def analyze_generated_records(visits_iterator: Iterator[Row]):
   for visit_record in visits_iterator:
    # ...
      if visit_record.offset > max_local_offset:
       max_local_offset = visit_record.offset      # 파티션별 최대 오프셋 추적
      current_partition = visit_record.partition

      max_offsets_tracker.add(PartitionWithOffset(partition=current_partition,
       offset=max_local_offset))

 visits_to_analyze.foreachPartition(analyze_generated_records)
```

**예시 3 — 무효 레코드 요약 (Example 9-27)**

또 다른 수정은 **accumulator 사용** —
**무효 row 수와 파티션별 최대 오프셋을 동시에 만들어 내는 복잡한 SQL 쿼리를 피하기 위함**:

```python
accumulators = {'event_time': spark_context.accumulator(0),
  'user_id': spark_context.accumulator(0), 'page': spark_context.accumulator(0)}
all_events_accumulator = spark_context.accumulator(0)

def analyze_generated_records(visits_iterator: Iterator[Row]):
 for visit_record in visits_iterator:
  if not visit_record.event_time:
   accumulators['event_time'].add(1)     # 레코드를 훑는 김에 같이 셈
  if not visit_record.user_id:
   accumulators['user_id'].add(1)
  if not visit_record.page:
   accumulators['page'].add(1)
# ...

observation_dump = {
 '@timestamp': datetime.utcnow().isoformat(),
 'invalid_event_time': accumulators['event_time'].value,   # value 호출 시점에 집계됨
 'invalid_user_id': accumulators['user_id'].value,
 'invalid_page': accumulators['page'].value,
 'all_events': all_events_accumulator.value,
# ...
```

> **참고 사항 — Spark accumulator 의 동작**
> accumulator 는 **Apache Spark 전용 컴포넌트** 로,
> **`value` 메서드를 호출하지 않는 한 각 executor 에서 로컬로 동작**.
> `value` 함수가 실제로 호출되면 **executor 들이 로컬 accumulator 를 클러스터 메인 노드로 전송** 하고,
> 메인 노드가 결과를 집계.
> 이 데이터 관찰 예시에서 accumulator 는 **입력 데이터셋을 두 번 쿼리하는 일**
> (한 번은 lag, 한 번은 무효 컬럼)을 **피하는 훌륭한 방법**.

#### 비교 — #63 Offline Observer 와 #64 Online Observer

| 항목 | #63 Offline Observer | #64 Online Observer |
|---|---|---|
| **관찰 잡의 위치** | 별도 파이프라인 (분리) | 생성 파이프라인 내부 (내장) |
| **인사이트 도달 시점** | 관찰 잡 스케줄에 따름 (최대 며칠 지연) | 데이터 생성 직후 |
| **생성 파이프라인 영향** | 없음 | 지연 추가 · 실패 전파 가능 |
| **배치 구현** | 독립 DAG + row ID 경계 고정 | Parallel Split(#37) 또는 Local Sequencer(#33) |
| **스트리밍 구현** | 처리 결과 위에 별도 잡 | 생성 잡 안에 통합 (분리 불가) |
| **주요 함정** | 인사이트가 늦음 · 몰아 처리 시 컴퓨트 폭증 | 완료 지연 · Parallel Split 의 관찰 범위 불일치 |
| **완화책** | 관찰 주기 단축 · 샘플링 | `all_done` 트리거 · 샘플링 · Local Sequencer 선택 |

<details>
<summary><b>⚠ 트러블 로그</b> — Parallel Split 로 관찰을 붙이면 "적재 단계에서 깨진 데이터" 를 영원히 못 봄.</summary>
<div markdown="1">

**예 —** 변환 결과를 관찰하면서 동시에 PostgreSQL 로 적재하도록 Parallel Split 을 구성함.
변환 출력의 `event_time` 은 `2024-01-01T01:00:00+09:00` 형식이었는데
대상 컬럼이 `TIMESTAMP WITHOUT TIME ZONE` 이라 **적재 시 오프셋이 잘려 9시간 밀림**.
관찰 잡은 변환 출력만 보므로 **`invalid_event_time = 0`** 을 계속 보고했고,
분석가가 리포트 시각이 이상하다고 지적할 때까지 **3주간 정상으로 표시**됨.

**권장 —** 컨슈머가 실제로 읽는 대상이 적재 결과라면 **Local Sequencer 로 적재 뒤에 관찰** 할 것.
속도 때문에 Parallel Split 을 쓴다면, **관찰 범위가 변환까지라는 사실을 대시보드에 명시** 할 것.

</div>
</details>

<details>
<summary><b>⚠ 트러블 로그</b> — 스트리밍 잡에 관찰을 내장하면서 프로파일링까지 넣으면 생성 잡이 함께 죽음.</summary>
<div markdown="1">

**예 —** Structured Streaming 의 `foreachBatch` 안에서 `ProfileReport` 를 매 마이크로배치마다 호출함.
트래픽이 몰린 저녁 시간대에 배치 크기가 커지자 **프로파일링이 드라이버 메모리를 소진** 했고,
**데이터 생성 자체가 멈춰** Kafka 컨슈머 lag 이 40분치까지 밀림.
관찰을 붙인 목적이 "빨리 알기" 였는데 정작 **파이프라인을 세운 원인** 이 됨.

**권장 —** 스트리밍에서는 **가벼운 집계(accumulator)만 잡 안에 두고**,
프로파일링처럼 무거운 작업은 **샘플링하거나 Offline Observer 로 분리** 할 것.

</div>
</details>

---

## 4. 요약

챕터 9 는 **"이 데이터를 믿어도 되는가"** 를 **값을 막고(9.1 품질 확보)** →
**스키마를 막고(9.2 스키마 일관성)** → **막는 규칙 자체를 지켜보는(9.3 품질 관찰)** 세 단계로 다룸.

- **품질 확보** — 신뢰할 만한 데이터셋을 만드는 **서로 다른 계층의 방어선**.
  #59 **AWAP** 은 **파이프라인 계층** 에서, 입력과 출력 양쪽에 감사 단계를 두어
  **저품질 데이터를 처리하고 노출하는 일 자체를 피함**.
  #60 **Constraints Enforcer** 는 **데이터베이스 계층** 에서, 삽입되는 필드의 조건을 **선언적으로** 정의해
  더 강한 보호를 제공.
- **스키마 일관성** — 값이 아니라 **계약(contract)** 을 지키는 일.
  #61 **Schema Compatibility Enforcer** 는 **외부 컴포넌트** 를 통해 스키마를 일관되게 유지하며,
  **컨슈머 (다운스트림)를 깨는 스키마 변경을 애초에 거부**.
  #62 **Schema Migrator** 는 반대로 **의도한 파괴적 변경을 안전하게 통과** 시킴 —
  핵심 장치는 규칙이 아니라 **유예 기간**.
- **품질 관찰** — 제약과 제어는 저품질 데이터셋의 발행을 막지만,
  **이슈가 없음을 보장하지는 않음**. 정확히는 **내가 정의한 규칙에 대해서만** 이슈가 없음을 보장.
  규칙 정의를 빠뜨릴 수도, 진화한 데이터셋에 맞춰 조정해야 할 수도 있음.
  #63 **Offline Observer** 와 #64 **Online Observer** 가 그 규칙을 갱신할 입력을 만듦.
  둘의 차이는 **무엇을 재느냐가 아니라 언제 재느냐**.
- **남은 문제** — 챕터 9 의 모든 패턴은 **잡이 돌고 있다** 는 전제 위에 서 있음.
  업스트림 흐름이 끊겨 **AWAP 잡 자체가 실행되지 않으면** 감사도 제약도 옵서버도 전부 침묵.
  ⇒ 그래서 **챕터 10 데이터 관찰 가능성(#65 Flow Interruption Detector 부터)** 이 필요.

| 패턴 | 카테고리 | 한 줄 요약 | 핵심 트레이드오프 |
|---|---|---|---|
| #59 AWAP | Quality Enforcement | 파이프라인이 완전한 데이터셋 위에서 동작하고 <br>저품질 데이터를 노출하지 않게 보장 | 데이터 관련 검증의 컴퓨팅 비용 / <br>완벽하지 않고 시간이 지나면 규칙 조정 필요 · 추가 지연 |
| #60 Constraints Enforcer | Quality Enforcement | 프로듀서가 데이터 품질 이슈를 <br>도입하지 못하게 보장 | 선언만 하면 DB 가 강제 / <br>all-or-nothing 으로 인한 긴 왕복 · 컨슈머마다 다른 기대 |
| #61 Schema Compatibility Enforcer | Schema Consistency | 스키마 변경이 컨슈머와 <br>호환되도록 보장 | 깨는 변경을 사전 차단 / <br>스키마 레지스트리 통신 오버헤드 · 어려워지는 스키마 진화 |
| #62 Schema Migrator | Schema Consistency | 컨슈머 (다운스트림)를 깨지 않고 <br>스키마를 마이그레이션 | 파괴적 변경을 안전하게 수행 / <br>레코드 크기가 크게 늘 수 있음 · 때로는 필드를 제거할 수 없음 |
| #63 Offline Observer | Quality Observation | 관찰을 별도 파이프라인으로 <br>구현 | 생성 파이프라인에 영향 없음 / <br>인사이트가 늦을 수 있음 · 관찰 데이터셋 처리에 컴퓨트가 과대해질 수 있음 |
| #64 Online Observer | Quality Observation | 관찰을 관찰 대상 파이프라인의 <br>일부로 구현 | 생성 직후 인사이트 / <br>추가 처리 지연 · Parallel Split 은 빠르지만 관찰 범위가 달라질 수 있음 |

```
[챕터 9 선택 가이드 — #59~#64]
──────────────────────────────────────────────────────────────────────
 ① 잡은 성공했는데 숫자가 틀릴 때
   볼륨 급감·분포 이상처럼 "값의 관계" 가 문제  ─► #59 AWAP (2차 감사, 데이터셋 수준 검증)
   변환 로직이 결측을 만드는지 확인하고 싶음     ─► #59 AWAP (2차 감사, 레코드 수준 검증)
   입력 파일이 애초에 이상한지 싸게 확인         ─► #59 AWAP (1차 감사, 메타데이터 검사)

 ② 일부만 살리고 싶을 때
   유효 레코드는 배포, 무효는 격리              ─► #59 AWAP · data dispatching
   결함이 있어도 배포하되 꼬리표를 달고 싶음      ─► #59 AWAP · nonblocking audit

 ③ 검증 코드를 더 짜고 싶지 않을 때
   NULL·타입·값 범위·참조 무결성                ─► #60 Constraints Enforcer (DB · Delta CHECK)
   Kafka 로 보내기 전에 프로듀서에서 막고 싶음    ─► #60 Constraints Enforcer (protovalidate)

 ④ 값이 아니라 스키마가 바뀔 때
   프로듀서 여럿 · 명시적 호환성 모드가 필요      ─► #61 · 외부 서비스 (Schema Registry)
   테이블에 쓰는 순간 막으면 충분                ─► #61 · 암묵적 (Delta Lake · RDB)
   DDL 자체를 통제해야 함                       ─► #61 · DDL 이벤트 트리거 (또는 ALTER 권한 회수)

 ⑤ 스키마를 의도적으로 바꿔야 할 때
   필드 이름이 나쁨 · 타입을 바꾸고 싶음         ─► #62 (새 필드 추가 후 유예 기간)
   필드를 지우고 싶은데 사용처를 모름            ─► #62 + #70 Fine-Grained Tracker
   전이(transitive) 호환성이 걸려 있음           ─► 먼저 비전이로 낮출 것 (#62 의 전제)

 ⑥ 규칙이 낡았는지 지켜봐야 할 때
   지금은 문제없고 생성 파이프라인을 못 건드림    ─► #63 Offline Observer
   야간 일괄로 자원 경합을 피하고 싶음            ─► #63 Offline Observer
   컨슈머보다 먼저 알아야 함 (주 1회로는 늦음)    ─► #64 Online Observer
   배치이고 완료 지연을 감수 가능                ─► #64 · Local Sequencer (#33)
   배치이고 속도가 중요 · 관찰 범위 한정 수용     ─► #64 · Parallel Split (#37)
   스트리밍                                    ─► #64 · 잡 내부 통합 (+ 샘플링)
──────────────────────────────────────────────────────────────────────
 ⚠ AWAP 은 1차 감사의 이중 읽기와 스트리밍 지연을, 제약은 all-or-nothing 롤백을,
   호환성 강제는 전이 모드로 인한 진화 봉쇄를, #62 는 유예 기간의 종료일을,
   #63 은 관찰의 멱등성을, #64 는 관찰 실패의 전파를 각각 조심할 것.
```

**정리 1 — 계층이 다르지 두 패턴이 경쟁하지 않음** — #59 는 **파이프라인 안**, #60 은 **저장 계층**.
DB 제약으로 막을 수 있는 것(널·타입·범위·참조)은 **#60 에 맡기고**,
**"어제보다 방문이 50% 줄었다"** 처럼 **한 row 만 봐서는 알 수 없는 것** 을 **#59 가 맡음**.
책의 Example 9-3 이 pandas 로 `NULL` 을 직접 검사한 이유도 **CSV 가 제약 없는 포맷** 이기 때문 —
Delta 테이블이었다면 그 검사는 `#60` 이 대신했을 것.

**정리 2 — 관통하는 원칙은 "배포 전에 멈춘다"** — AWAP 의 이름 순서가 곧 설계.
**감사 → 쓰기(스테이징) → 감사 → 배포** 에서 **Load 가 마지막** 이라는 점이 패턴의 전부라고 해도 됨.
스테이징 계층이 없으면 감사는 사후 부검이 되고, 잘못된 숫자는 이미 컨슈머 (조회하는 쪽)의 화면에 도달함.

**정리 3 — #61 과 #62 는 방향이 반대인 한 쌍** — `#61` 은 **도구가 사람의 선의를 대신 막아 주는 패턴**,
`#62` 는 **도구가 막을 수 없는 변경을 사람의 합의로 통과시키는 절차**.
그래서 `#62` 에서 기술적인 부분은 `ADD COLUMN` 한 줄뿐이고,
나머지는 전부 **컨슈머와 전환 기간을 합의하는 일**. 데드라인이 없으면 패턴 자체가 성립하지 않음.
`#62` 가 **전이 호환성이 아닐 것** 을 전제하는 이유도 같음 —
전이 모드에서는 필드 제거·이름 변경이 애초에 불가능하기 때문.

**정리 4 — 관찰은 감사가 아님** — 책이 `Observability Versus Auditing` 박스를 따로 둔 이유.
**감사(#59 AWAP)는 차단하고, 관찰(#63·#64)은 차단하지 않음**.
그래서 관찰 결과는 파이프라인을 세우는 데 쓰이는 것이 아니라,
**다음 번 강제 규칙을 무엇으로 정할지 결정하는 입력** 으로 쓰임.
`#64` 예시가 `all_done` 트리거로 관찰 실패를 흡수하는 것도 같은 이유.

**정리 5 — 강제는 "규칙이 맞다" 는 가정 위에 서 있음** — #59 의 *Rules coverage*,
*An issue may not be an issue* 두 고려사항이 같은 이야기를 함.
**볼륨이 3배로 뛴 것이 사고가 아니라 성공** 일 수 있고, **오늘의 규칙이 내일의 데이터** 를 못 덮을 수 있음.
그래서 감사 결과를 **차단/경고 두 등급** 으로 나누고,
규칙의 유효성 자체를 **#63·#64 로 계속 갱신** 해야 함.

> 제약과 제어를 강제해도 **이슈가 없음이 보장되지는 않음** — 보장되는 것은
> **내가 정의한 규칙에 대해서만 이슈가 없다** 는 사실뿐.
> ※ 챕터 9 의 패턴은 모두 **데이터가 도착했다** 는 전제 위에서 동작.
> 그 전제가 깨지는 순간을 감시하는 것이 **챕터 10 의 #65 Flow Interruption Detector** —
> 감시 대상을 **프로세스 상태에서 출력 데이터의 신선도로** 옮김.

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
3. [요약](#3-요약)

> 본 문서는 챕터 9 의 **#59~#61** — **9.1 Quality Enforcement · 9.2 Schema Consistency(전반)** — 을 다룸.
> **#62 Schema Migrator** 와 **9.3 Quality Observation(#63 Offline Observer · #64 Online Observer)** 은 별도로 정리.

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
 ※ 본 문서는 9.1 전체와 9.2 의 #61 — 즉 #59~#61 을 다룸.
```

**세 카테고리가 나뉜 이유**

- **① 품질 확보** — 저품질 데이터를 **컨슈머 (다운스트림)에게 노출하지 않는 것** 이 첫 번째 방어선.
- **② 스키마 일관성** — 프로듀서는 대개 문제없이 데이터를 생성하다가, **어느 날 스키마를 바꿈**.
  진화 유형에 따라 이것이 **파이프라인의 치명적 실패** 와 **데이터 제공자에 대한 신뢰 상실** 로 이어짐.
- **③ 품질 관찰** — **오늘 정한 강제 규칙이 내일의 데이터에도 유효한가** 를 보장하는 일.
  데이터와 스키마를 통제하는 것에 더해, **컨슈머 (조회하는 쪽)보다 먼저 새 이슈를 발견** 하려면
  데이터셋을 관찰해야 함. 관찰 기법이 **처리 중인 데이터셋의 가장 최신 개요** 를 제공해
  강제 규칙을 최신 상태로 유지하게 해 줌.

### 패턴 흐름 — 챕터 8에서 챕터 9 로

```
[패턴 흐름 — 챕터 8에서 챕터 9 로]
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
      ▼
   #62 Schema Migrator — 컨슈머 (다운스트림)를 깨지 않고 스키마를 진화 (본 문서 범위 밖)
      │
      ▼
 9.3 Quality Observation — 규칙이 낡지 않게 관찰 (#63·#64, 본 문서 범위 밖)
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

## 3. 요약

챕터 9 는 **"이 데이터를 믿어도 되는가"** 를 **값을 막고(9.1 품질 확보)** →
**스키마를 막고(9.2 스키마 일관성)** → **막는 규칙 자체를 지켜보는(9.3 품질 관찰)** 세 단계로 다룸.
본 문서는 그중 **#59~#61** 을 정리.

- **품질 확보** — 신뢰할 만한 데이터셋을 만드는 **서로 다른 계층의 방어선**.
  #59 **AWAP** 은 **파이프라인 계층** 에서, 입력과 출력 양쪽에 감사 단계를 두어
  **저품질 데이터를 처리하고 노출하는 일 자체를 피함**.
  #60 **Constraints Enforcer** 는 **데이터베이스 계층** 에서, 삽입되는 필드의 조건을 **선언적으로** 정의해
  더 강한 보호를 제공.
- **스키마 일관성** — 값이 아니라 **계약(contract)** 을 지키는 일.
  #61 **Schema Compatibility Enforcer** 는 **외부 컴포넌트** 를 통해 스키마를 일관되게 유지하며,
  **컨슈머 (다운스트림)를 깨는 스키마 변경을 애초에 거부**.
- **남은 문제** — 제약과 제어는 저품질 데이터셋의 발행을 막지만,
  **이슈가 없음을 보장하지는 않음**. 정확히는 **내가 정의한 규칙에 대해서만** 이슈가 없음을 보장.
  규칙 정의를 빠뜨릴 수도, 진화한 데이터셋에 맞춰 조정해야 할 수도 있음.
  ⇒ 그래서 **9.3 품질 관찰(#63 Offline Observer · #64 Online Observer)** 이 필요.

| 패턴 | 카테고리 | 한 줄 요약 | 핵심 트레이드오프 |
|---|---|---|---|
| #59 AWAP | Quality Enforcement | 파이프라인이 완전한 데이터셋 위에서 동작하고 <br>저품질 데이터를 노출하지 않게 보장 | 데이터 관련 검증의 컴퓨팅 비용 / <br>완벽하지 않고 시간이 지나면 규칙 조정 필요 · 추가 지연 |
| #60 Constraints Enforcer | Quality Enforcement | 프로듀서가 데이터 품질 이슈를 <br>도입하지 못하게 보장 | 선언만 하면 DB 가 강제 / <br>all-or-nothing 으로 인한 긴 왕복 · 컨슈머마다 다른 기대 |
| #61 Schema Compatibility Enforcer | Schema Consistency | 스키마 변경이 컨슈머와 <br>호환되도록 보장 | 깨는 변경을 사전 차단 / <br>스키마 레지스트리 통신 오버헤드 · 어려워지는 스키마 진화 |

```
[챕터 9 선택 가이드 — #59~#61]
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
──────────────────────────────────────────────────────────────────────
 ⚠ AWAP 은 1차 감사의 이중 읽기와 스트리밍 지연을, 제약은 all-or-nothing 롤백을,
   호환성 강제는 전이 모드로 인한 진화 봉쇄를 각각 조심할 것.
```

**정리 1 — 계층이 다르지 두 패턴이 경쟁하지 않음** — #59 는 **파이프라인 안**, #60 은 **저장 계층**.
DB 제약으로 막을 수 있는 것(널·타입·범위·참조)은 **#60 에 맡기고**,
**"어제보다 방문이 50% 줄었다"** 처럼 **한 row 만 봐서는 알 수 없는 것** 을 **#59 가 맡음**.
책의 Example 9-3 이 pandas 로 `NULL` 을 직접 검사한 이유도 **CSV 가 제약 없는 포맷** 이기 때문 —
Delta 테이블이었다면 그 검사는 `#60` 이 대신했을 것.

**정리 2 — 관통하는 원칙은 "배포 전에 멈춘다"** — AWAP 의 이름 순서가 곧 설계.
**감사 → 쓰기(스테이징) → 감사 → 배포** 에서 **Load 가 마지막** 이라는 점이 패턴의 전부라고 해도 됨.
스테이징 계층이 없으면 감사는 사후 부검이 되고, 잘못된 숫자는 이미 컨슈머 (조회하는 쪽)의 화면에 도달함.

**정리 3 — 강제는 "규칙이 맞다" 는 가정 위에 서 있음** — #59 의 *Rules coverage*,
*An issue may not be an issue* 두 고려사항이 같은 이야기를 함.
**볼륨이 3배로 뛴 것이 사고가 아니라 성공** 일 수 있고, **오늘의 규칙이 내일의 데이터** 를 못 덮을 수 있음.
그래서 감사 결과를 **차단/경고 두 등급** 으로 나누고,
규칙의 유효성 자체를 **9.3 품질 관찰(#63·#64)** 로 계속 갱신해야 함.

> 제약과 제어를 강제해도 **이슈가 없음이 보장되지는 않음** — 보장되는 것은
> **내가 정의한 규칙에 대해서만 이슈가 없다** 는 사실뿐.
> ※ 본 문서에서 다루지 않은 **#62 Schema Migrator** — 필드 이름 변경·타입 진화 같은
> **의도적인 파괴적 변경** 을 컨슈머 (다운스트림)를 깨지 않고 수행하는 패턴.
> **전이(transitive) 호환성이 아닐 것** 을 전제로 함 —
> 전이 모드에서는 필드 제거·이름 변경이 애초에 불가능하기 때문.

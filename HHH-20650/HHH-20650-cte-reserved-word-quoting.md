# HHH-20650 CTE 이름의 예약어 인용

- 이슈: https://hibernate.atlassian.net/browse/HHH-20650
- PR: https://github.com/hibernate/hibernate-orm/pull/12993
- 상태: **CTE 이름 병합됨** · CTE 컬럼 / 생성 SEARCH·CYCLE 컬럼 인용은 후속
- 날짜: 2026-07-08

## 배경 / 이슈
Hibernate가 CTE 식별자를 렌더할 때 `dialect.getKeywords()`를 참조하지 않고 원시 문자열 그대로 출력한다. 이 경로는 CTE 이름, CTE 컬럼 목록, 그리고 `SEARCH`/`CYCLE` 절을 에뮬레이션할 때 생성하는 `depth`/`path` 컬럼에 모두 해당한다. 이 식별자 중 하나가 대상 방언의 예약어이면 생성된 SQL이 syntax error로 거부된다.

CUBRID에서는 `DEPTH`·`DATA`가 예약어라 다음이 실패한다.
- `SEARCH` 절이 있는 재귀 CTE → 순서 컬럼 `depth` 생성 → `with rec(..., depth, ...) as (...)` → `Syntax error: unexpected 'depth'`
- 이름이 `data`인 CTE → `with data as (...)` → `Syntax error: unexpected 'data'`

일반 테이블·컬럼 식별자는 예약어일 때 이미 인용되지만 CTE 렌더 경로는 그렇지 않아, `registerKeyword`로 등록해도 이 경로엔 효과가 없다. 기본 H2 환경에서는 해당 단어들이 예약어가 아니라 재현되지 않는다.

## 원인 분석 (AS-IS)
- CTE 이름(`CteTable.cteName`)과 컬럼(`CteColumn.columnExpression`)이 **`IdentifierHelper` 인용 파이프라인을 우회하는 원시 문자열**이다. 일반 식별자는 `IdentifierHelper.toIdentifier()` → `normalizeQuoting()`을 거쳐 예약어/글로벌·자동 인용이 적용되지만, CTE 경로는 이 단계를 건너뛴다.
- 첫 접근은 렌더 시점(`AbstractSqlAstTranslator`)에서 인용하는 방식이었으나, 이는 증상만 덮는 밴드에이드다. 리뷰 피드백(@mbellade)에 따라 **파이프라인 앞단에서 식별자를 정규화**하는 것이 올바른 위치다. 그래야 각 방언이 글로벌 인용/자동 인용 설정으로 커스터마이즈할 수 있고, 렌더러는 이미 정규화된 식별자만 출력한다.

## 변경 / 해결 (TO-BE)
- **CTE 이름을 `IdentifierHelper` 경유로 라우팅.** `BaseSqmToSqlAstConverter.getCteName()`에서 생성한 이름을 `IdentifierHelper.toIdentifier(name).render(dialect)`로 통과시켜, 다른 식별자와 동일하게 예약어/글로벌·자동 인용 처리를 받게 했다. 렌더 시점 특수 처리는 제거했다.
- **재현 테스트 추가**: `CteReservedWordQuotingTest` — 이름이 예약어인 재귀 CTE를 실행하고 생성된 SQL이 그 이름을 인용하는지 단언한다(수정 전 실패, 수정 후 통과).
- **범위 = CTE 이름.** CTE 컬럼 목록과 생성 `SEARCH`/`CYCLE` `depth`/`path` 컬럼의 인용은 이 변경에 포함하지 않았다(아래 후속 참조).

## 검증
- **H2** 전체 `hibernate-core` 스위트: **0 회귀**.
- 재현 테스트: 수정 없으면 실패, 있으면 통과.
- **CUBRID 10.2 / 11.0 / 11.2 / 11.3 / 11.4** CI: CTE 이름 인용 동작 확인. (자동 인용 설정이 켜진 경우에 적용 — 헬퍼가 설정에 따라 인용하므로.)

## 결과 / 영향
- CTE 이름의 예약어 인용이 upstream `main`에 **병합**되었다(메인테이너 승인). 이 문제는 예약어를 재귀 CTE 이름/컬럼으로 쓰는 어떤 방언에서도 발생할 수 있어, CUBRID 외에도 일반적으로 유효한 개선이다.
- **후속 과제 (미해결):** CTE 컬럼 목록 + 생성 `SEARCH`/`CYCLE` 컬럼의 예약어 인용. 난점은 `CteColumn.columnExpression`이 **이중 용도**라는 점이다.
  1. SQL에 출력되는 **렌더링 식별자** — 예약어면 인용 필요
  2. `SEARCH`/`CYCLE` 컬럼을 SQM 속성 이름과 대조하는 **매칭 키**(`regionMatches`) — 원시 이름 필요

  생성 시점에 인용하면 원시 이름과의 대조가 깨진다. 리뷰에 세 가지 설계 옵션을 질의해 둔 상태다: (1) `columnExpression`은 매칭용 원시 이름으로 두고 렌더 시점에만 인용, (2) `CteColumn`이 원시 이름과 정규화 이름을 함께 보유, (3) 대조 로직을 인용-인지(quoting-aware)로 변경. 이름 스코프만 병합되었고, 컬럼 인용은 이 설계 결정과 함께 후속 PR로 다룬다.

## 참고
- JIRA: https://hibernate.atlassian.net/browse/HHH-20650
- PR: https://github.com/hibernate/hibernate-orm/pull/12993
- Commit: `11b19d41ce` (CTE 이름 → IdentifierHelper), `ad1bf2a24b` (재현 테스트)
- 리뷰 지점: `BaseSqmToSqlAstConverter.getCteName()` (이름), `CteColumn.columnExpression` / `forEachCteColumn` (컬럼 후속)

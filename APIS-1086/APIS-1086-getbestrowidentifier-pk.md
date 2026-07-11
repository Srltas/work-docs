# APIS-1086 getBestRowIdentifier가 PK-only 테이블에서 빈 결과를 반환

- 이슈: https://jira.cubrid.org/browse/APIS-1086
- PR: (열리면 링크)
- 상태: 분석·설계 완료 · 구현 전
- 날짜: 2026-07-11

## 배경 / 이슈

`DatabaseMetaData.getBestRowIdentifier`는 "행을 유일하게 식별하는 최적 컬럼 집합"(JDBC API 매뉴얼: *"a table's optimal set of columns that uniquely identifies a row"*)을 돌려주는 표준 JDBC API다. ORM·데이터 편집 툴이 방금 읽은 행을 정확히 UPDATE/DELETE 하기 위한 **행 식별(row identity)** 을 구성할 때 쓰며, 보통 그 답은 기본키(PK)다.

그런데 CUBRID JDBC는 **테이블의 유일한 키가 PRIMARY KEY일 때 빈 ResultSet을 반환**한다. 같은 컬럼을 UNIQUE 제약으로 만들면 정상 반환되지만 PK로 만들면 아무것도 나오지 않는다. PK가 있는 테이블은 유일 식별자가 명백히 존재하므로 빈 결과는 API 계약 위반이다(스펙이 "PK 특정 반환"을 강제하진 않지만, 유일 식별자가 있는데 빈 결과를 주는 것이 위반의 핵심). 5-DB JDBC 타입 매핑 실측 중 CUBRID만 이 메서드가 비어 나와 추적했고, 그 내용을 본 이슈로 정리한다.

- 실측: Testcontainers `cubrid/cubrid:11.4` + cubrid-jdbc `11.3.2.0053`. 키 가능 개념 20종을 **PK로** 만들면 `getBestRowIdentifier` 반환 **0/20**, 동일 컬럼을 **UNIQUE로** 만들면 **20/20**.
- 크로스-DB: PostgreSQL 등은 PK를 정상 반환. CUBRID만 PK를 못 봄.

## 원인 분석 (AS-IS)

현재 증상은 **단일·복합 PK 모두 빈 결과**다. 원인은 사실상 2겹이며, 초기 분석의 일부를 정정한다.

1. **브로커가 PK를 제약 스키마 경로로 내보내지 않음.** 행을 방출하는 루틴은 `cas_execute.c`의 `fetch_constraint()`(5649–5744)로, `db_constraint_type()` 스위치(5675–5687)에서 `PRIMARY_KEY(5)`가 `default: goto const_next`로 빠져 **행 자체가 스킵**된다. 카운트 루틴 `sch_constraint()`(7730–7765)도 PK 미포함, 방출부의 `PRIMARY_KEY` 출력 컬럼은 상수 0으로 하드코딩(5716), 인코더 `constraint_dbtype_to_castype()`(8911–8925)에도 PK 매핑이 없다. → PK-only 테이블은 `SCH_CONSTRAINT` 응답이 **0행**.

2. **(정정) 드라이버 필터는 "type 0만 수용"이 아니다.** `CUBRIDDatabaseMetaData.getBestRowIdentifier()`의 `if (us.getInt(0) != 0) continue;`(1373)에서 컬럼 0은 브로커가 `constraint_dbtype_to_castype`로 축약한 **CCI 코드(0=unique 계열, 1=index 계열)**이지 원시 제약 타입(0~6)이 아니다. 5가 내려올 일이 없으며 이 줄은 사실 "index 계열 스킵"이다. 초기 분석의 "type 5라 걸러진다"는 부정확.

3. **(신규) 드라이버 선택 로직이 복합키에서 독립적으로 깨져 있음.** 이 메서드는 제약 **이름**의 콤마 개수로 컬럼 수를 추정(1375–1383)하고 `minindex`에 off-by-one이 있다. 현재 와이어 포맷은 "컬럼당 1행, 이름=제약명"이라 콤마가 없어 `min=0`이 되고, 방출 루프(1412–1447)는 잘못된 위치에서 **1행만** 낸다. 단일 컬럼은 범위 밖 `moveCursor`가 올바른 행으로 복원돼 우연히 동작하지만, **복합키는 첫 컬럼 대신 두 번째 컬럼을 1행만 반환**한다. 지금은 ①(PK 미전달)에 가려 드러나지 않는다.

4. **(신규) 잠재 NPE.** 방출 행이 `SCOPE`(value[0])를 설정하지 않는데, `CUBRIDComparator.compare_getBestRowIdentifier`(87–89)가 컬럼 0을 `Short`로 캐스팅·역참조한다. 출력이 2행 이상이면 `sortTuples`에서 NPE(복합 UNIQUE 키도 동일).

부가 사실:
- PK 데이터 자체는 별도 경로로 정상 제공된다: `getPrimaryKeys`는 `SCH_PRIMARY_KEY(16)` → 브로커 `sch_primary_key()`(카탈로그 SQL, `is_primary_key='YES'`)를 쓴다. `getBestRowIdentifier`/`getIndexInfo`만 깨진 `SCH_CONSTRAINT(11)` 경로를 쓴다.
- 기대 동작 테스트가 이미 있으나 비활성: JDBC 테스트 스위트의 `TestCUBRIDDatabaseMetaData.test9()`(복합 PK)가 `@Ignore` 상태.

## 변경 / 해결 (TO-BE)

근본원인 방식으로 **브로커 + 드라이버 동시 수정**.

**브로커 (`cas_execute.c`)**
- `fetch_constraint`/`sch_constraint` 스위치에 `DB_CONSTRAINT_PRIMARY_KEY` case 추가(오름차순 방출, 카운트와 방출 일치 — 프로토콜 정합성).
- `PRIMARY_KEY` 출력 컬럼을 `(type == PRIMARY_KEY) ? 1 : 0`으로.
- `constraint_dbtype_to_castype`에서 PK → `CCI_CONSTRAINT_TYPE_UNIQUE`(컬럼 0이 0으로 도착 → 드라이버 수용).

**드라이버 (`CUBRIDDatabaseMetaData.getBestRowIdentifier`)**
- 선택/출력 재작성 — 제약별(이름) 그룹화 → 컬럼 수 최소 unique 계열(PK 포함) 선택 → 선택 제약의 **모든** 컬럼을 키 순서로 방출 + `SCOPE(bestRowSession)` 설정.
- 타입 해석부를 헬퍼로 추출하고 행-끝 가드 추가(이름 미발견 시 무한루프 방지).

**테스트**
- `TestCUBRIDDatabaseMetaData.test9()` `@Ignore` 제거 + 단일 컬럼 PK 케이스 추가.

대안(참고): 드라이버 단독으로 `SCH_PRIMARY_KEY` 폴백을 넣으면 브로커 미수정으로도 해결 가능하나, CCI/`getIndexInfo` 등 타 클라이언트는 여전히 PK를 못 본다.

## 검증

(구현 전 — 계획)
- 엔진 재빌드·설치·브로커 재시작 → 드라이버 빌드(cubrid-jdbc 11.3.3) → CTP JDBC 스위트 실행.
- 통과 기준: 활성화한 `test9`(첫 컬럼 `b`, `DATA_TYPE` INTEGER, `SCOPE` non-null), 신규 단일 컬럼 PK 케이스, 기존 메타데이터 테스트 회귀 없음(`getIndexInfo`/`getPrimaryKeys` 포함).
- 원시 점검: PK-only → PK 컬럼 반환, UNIQUE-only 동작 불변, 복합 UNIQUE NPE 해소.

## 결과 / 영향

- 규약 비준수 해소: PK-only 테이블에서 `getBestRowIdentifier`가 PK 컬럼을 반환.
- 파급: 같은 `SCH_CONSTRAINT`를 쓰는 `getIndexInfo`도 PK 백킹 인덱스를 노출하게 됨(JDBC상 타당한 개선, 중복 노출 여부 검증 필요). `getPrimaryKeys`는 불변.
- 주의: 브로커 count/emit 스위치는 반드시 함께 수정(프로토콜 일치). 신규 드라이버 + 구버전 브로커 조합은 PK-only에서 여전히 빈 결과(브로커·드라이버 협조 릴리스 전제).
- 남은 과제: 구현·검증, PR.

## 참고

- JIRA: https://jira.cubrid.org/browse/APIS-1086
- PR: (예정)
- 엔진 소스: `cubrid/src/broker/cas_execute.c` — `fetch_constraint`(5649–5744), `sch_constraint`(7730–7765), `constraint_dbtype_to_castype`(8911–8925); `cubrid/src/compat/dbtype_def.h:467-475`.
- 드라이버 소스: `cubrid-jdbc` `CUBRIDDatabaseMetaData` — `getBestRowIdentifier`(1322–1454), `getPrimaryKeys`(1488–1546, `SCH_PRIMARY_KEY=16`), `getIndexInfo`(2181–2291); `CUBRIDComparator`(87–89); `jci/USchType.java`(`SCH_CONSTRAIT=11`, `SCH_ATTRIBUTE=4`, `SCH_PRIMARY_KEY=16`).
- JDBC 규약: `java.sql.DatabaseMetaData.getBestRowIdentifier` — "optimal set of columns that uniquely identifies a row" (Java SE 8 javadoc).
- 관련 분석: [5-DB 실측](../analysis/2026-07-10-5db-jdbc-type-mapping-measured.md), [타입 매핑 소스 분석](../analysis/2026-07-09-cubrid-jdbc-type-mapping.md)

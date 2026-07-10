# CUBRID JDBC `getBestRowIdentifier`가 기본키(PK)를 반환하지 않음

- 분류: bug
- 날짜: 2026-07-10
- 관련: 실측 노트 [5-DB JDBC 데이터 타입 매핑 실측 결과](2026-07-10-5db-jdbc-type-mapping-measured.md), 소스 분석 [CUBRID 타입 매핑 전수 분석](2026-07-09-cubrid-jdbc-type-mapping.md)

## 목적

`DatabaseMetaData.getBestRowIdentifier`의 역할을 쉽게 정리하고, **CUBRID JDBC에서 기본키(PK)만 있는 테이블에 이 메서드가 빈 결과를 돌려주는 버그**를 근본원인까지 기록한다.

## 배경 — getBestRowIdentifier가 하는 일 (쉽게)

"내가 방금 읽은 이 행을, 나중에 정확히 그 행만 다시 집으려면 **어느 컬럼을 WHERE에 걸면 되나?**"를 드라이버에 물어보는 표준 JDBC API다. 보통 답은 **기본키(PK)**이고, PK가 없으면 유니크 인덱스, 그것도 없으면 ROWID 같은 의사 컬럼이다. ORM·데이터 편집 툴이 **행 식별(row identity)** — 읽은 행을 정확히 UPDATE/DELETE — 을 구성할 때 쓴다.

즉 **PK가 있는 테이블이면 PK 컬럼이 나오는 것이 정상**이다. (실제로 PostgreSQL 등은 PK를 돌려준다.) 5-DB 타입 매핑 실측(위 관련 노트) 중, CUBRID만 이 메서드가 비어 나와 원인을 추적했다.

## 범위 / 방법

- **실측**: Testcontainers `cubrid/cubrid:11.4` + cubrid-jdbc `11.3.2.0053`. 같은 테이블을 PK로 만들 때와 UNIQUE로 만들 때의 `getBestRowIdentifier` 결과를 비교.
- **소스 추적**: 드라이버(cubrid-jdbc)와 엔진(cubrid, 소스 트리 11.5.0) 직접 확인. (엔진 트리는 11.5.0이나 해당 로직은 오래되어 11.4 실측 동작과 일치.)

## 발견 / 관찰

### 증상 · 재현

```sql
CREATE TABLE t_pk (id INT PRIMARY KEY, v VARCHAR(10));
```
```java
DatabaseMetaData md = conn.getMetaData();
try (ResultSet rs = md.getBestRowIdentifier(
        null, null, "t_pk", DatabaseMetaData.bestRowSession, true)) {
    while (rs.next()) System.out.println(rs.getString("COLUMN_NAME"));
}
// 기대: "id" 1행.  CUBRID 실측: 0행(빈 ResultSet).
```

같은 컬럼을 UNIQUE로 만들면 정상 반환된다:
```sql
CREATE TABLE t_uk (id INT NOT NULL UNIQUE, v VARCHAR(10));  -- getBestRowIdentifier → "id" 반환
```

- **실측 수치**: 키 가능한 개념 20종을 **PK로** 만들면 `getBestRowIdentifier` 반환 **0/20**, 동일 컬럼을 **UNIQUE로** 만들면 **20/20** 반환. (PK 여부만 바뀌었는데 결과가 갈림)
- **크로스-DB 대비**: PostgreSQL은 PK 테이블에서 PK를 정상 반환. CUBRID만 PK를 못 봄.

### 근본원인 (2계층 모두 PRIMARY KEY를 배제)

제약 타입 상수 — `cubrid/src/compat/dbtype_def.h:467`:
```
DB_CONSTRAINT_UNIQUE = 0, INDEX = 1, NOT_NULL = 2,
REVERSE_UNIQUE = 3, REVERSE_INDEX = 4, PRIMARY_KEY = 5, FOREIGN_KEY = 6
```

1. **엔진/브로커가 PK를 클라이언트로 보내지 않음** — `cubrid/src/broker/cas_execute.c` `sch_constraint()` (line 7730~). 제약을 순회하며 아래 스위치로 **개수만 세어** 전달하는데, `PRIMARY_KEY(5)`는 case가 없어 `default: break`로 빠진다:
   ```c
   switch (db_constraint_type (tmp_c)) {
     case DB_CONSTRAINT_UNIQUE:          // 0
     case DB_CONSTRAINT_INDEX:           // 1
     case DB_CONSTRAINT_REVERSE_UNIQUE:  // 3
     case DB_CONSTRAINT_REVERSE_INDEX:   // 4
       ... num_const++ ...
     default: break;                     // PRIMARY_KEY(5) 여기로 → 미전달
   }
   ```
   → PK만 있는 테이블은 스키마 제약 정보가 **0건**으로 내려온다.

2. **드라이버가 UNIQUE(type 0)만 수용** — `cubrid-jdbc` `CUBRIDDatabaseMetaData.getBestRowIdentifier()`는 스키마 제약 행을 돌며 `if (us.getInt(0) != 0) continue;` 로 **제약 타입이 0(UNIQUE)이 아니면 스킵**한다. 설령 PK가 내려와도 type 5라 걸러진다.

두 계층 모두 PRIMARY KEY를 제외하므로, **PK 컬럼은 `getBestRowIdentifier`에 절대 잡히지 않는다.** (PK가 내부적으로 유니크 인덱스로 뒷받침되더라도, 제약 타입 코드는 `PRIMARY_KEY(5)`라 두 필터 모두 통과 못 함. 기본 설정에서 PK를 UNIQUE로 접지 않음.)

## 결론

- **JDBC 규약 비준수 버그**다. `getBestRowIdentifier`의 본래 목적은 "최적 행 식별자(대개 PK)"를 돌려주는 것인데, CUBRID JDBC는 **가장 흔한 경우인 PK-only 테이블에서 빈 결과**를 준다.
- **영향**: PK로 행 식별을 구성하는 ORM/데이터 툴이 CUBRID에서만 이 경로가 깨질 수 있다. UNIQUE 제약을 별도로 건 테이블에서만 우연히 동작한다.
- 우리 타입 매핑 실측에서 §3(getBestRowIdentifier) 열을 얻으려 UNIQUE로 우회한 것도 이 버그 때문이었다.

## 다음 단계

- **이슈화 후보(CBRD)**: 드라이버·브로커 공동 수정. 방향 두 가지 —
  1. 브로커 `sch_constraint`가 `PRIMARY_KEY`도 전달하고, 드라이버가 이를 수용(PK를 유니크 식별자로 인정).
  2. 또는 드라이버가 PK-only일 때 `getPrimaryKeys` 경로로 폴백(‑`getPrimaryKeys`는 정상 동작).
- 표준 준수 재현 테스트(PK 테이블 → PK 반환) 추가.
- 회귀 검증기(`CubridBestRowIdentifierVerifier`)에 "PK-only → 빈 결과"를 현재 동작 특성화로 남기고, 수정 시 뒤집기.

## 참고

- 엔진 소스: `cubrid/src/broker/cas_execute.c` `sch_constraint()` (line 7730~, 스위치 7747~7750), `cubrid/src/compat/dbtype_def.h:467` (DB_CONSTRAINT 상수)
- 드라이버 소스: `cubrid-jdbc` `CUBRIDDatabaseMetaData.getBestRowIdentifier()` (제약 타입 0 필터)
- JDBC 규약: `java.sql.DatabaseMetaData.getBestRowIdentifier` — "optimal set of columns that uniquely identifies a row"
- 실측/분석 노트: [5-DB 실측](2026-07-10-5db-jdbc-type-mapping-measured.md), [소스 분석](2026-07-09-cubrid-jdbc-type-mapping.md)

# 5-DB JDBC 데이터 타입 매핑 실측 결과 (Testcontainers)

- 분류: analysis
- 날짜: 2026-07-10
- 관련: 앞선 소스 분석 노트 [CUBRID 11.4 데이터 타입 → java.sql.Types 매핑 전수 분석](2026-07-09-cubrid-jdbc-type-mapping.md)

## 목적

CUBRID JDBC 드라이버 소스로 분석했던 타입 매핑(5개 표)을 **실제 DB에 접속해 런타임으로 실측 검증**하고, 같은 개념 타입을 5개 DBMS(CUBRID/Oracle/MySQL/SQL Server/PostgreSQL)가 각각 어떤 `java.sql.Types`로 보고하는지 5개 메서드 표로 대조한다.

## 배경

기존 분석은 드라이버 소스 정적 분석이었다. 이를 근거로만 두지 않고, Testcontainers로 5개 DB를 실제 기동해 개념 타입별로 컬럼을 만들고 JDBC 메타데이터를 실측했다. CUBRID는 분석 5개 표를 그대로 assert로 걸어 회귀 검증했다(스위트 10/10 통과).

## 범위 / 방법

- **하네스**: 개념 타입 카탈로그(32종) × 메타데이터 메서드 × 5 DB. 개념마다 컬럼 생성 후 캡처 → DB별 JSON → 대조 매트릭스 생성. Testcontainers, JUnit 5, JDK 26, Docker.
- **표기**: 셀 값은 `java.sql.Types` 상수명(접두어 `Types.` 생략). `N/A`=해당 DB에 개념 없음 · `—`=메서드가 값 미반환(NULL/미해당) · `<unknown>`=`java.sql.Types` 밖의 드라이버 고유 코드.

### 측정 대상 버전 (DBMS 이미지 · JDBC 드라이버)

| DBMS | 엔진(도커 이미지) | JDBC 드라이버 |
|---|---|---|
| CUBRID | `cubrid/cubrid:11.4` (11.4) | cubrid-jdbc `11.3.2.0053` |
| Oracle | `gvenzl/oracle-free:slim-faststart` (Database Free 23ai) | ojdbc17 `23.26.2.0.0` |
| MySQL | `mysql:8.4` (8.4) | mysql-connector-j `9.7.0` |
| SQL Server | `mcr.microsoft.com/mssql/server:2025-latest` (2025) | mssql-jdbc `13.4.0.jre11` |
| PostgreSQL | `postgres:18.4-alpine` (18.4) | pgjdbc `42.7.13` |

(엔진 버전은 이미지 태그 기준. Oracle이 23ai임은 `getTypeInfo`에 `VECTOR`·`JSON` 네이티브 타입이 나타나는 것으로 실측 확인.)

## 발견 / 관찰

측정한 5개 메서드를 각각 표로 정리한다. 표1~4는 개념×5DB, 표5는 CUBRID 전용(합성 결과셋).

### 표1. `getColumnType` (실행 SELECT의 RSMD)

| 개념 | CUBRID | Oracle | MySQL | SQL Server | PostgreSQL |
|---|---|---|---|---|---|
| int16 | SMALLINT | NUMERIC | SMALLINT | SMALLINT | SMALLINT |
| int32 | INTEGER | NUMERIC | INTEGER | INTEGER | INTEGER |
| int64 | BIGINT | NUMERIC | BIGINT | BIGINT | BIGINT |
| decimal | NUMERIC | NUMERIC | DECIMAL | DECIMAL | NUMERIC |
| real | REAL | `<unknown>` | REAL | REAL | REAL |
| double | DOUBLE | `<unknown>` | DOUBLE | DOUBLE | DOUBLE |
| char | CHAR | CHAR | CHAR | CHAR | CHAR |
| varchar | VARCHAR | VARCHAR | VARCHAR | VARCHAR | VARCHAR |
| bit8 | BIT | N/A | N/A | N/A | N/A |
| binary | BINARY | VARBINARY | BINARY | BINARY | BINARY |
| varbinary | VARBINARY | VARBINARY | VARBINARY | VARBINARY | BINARY |
| blob | BLOB | BLOB | LONGVARBINARY | VARBINARY | BINARY |
| clob | CLOB | CLOB | LONGVARCHAR | VARCHAR | VARCHAR |
| date | DATE | TIMESTAMP | DATE | DATE | DATE |
| time | TIME | N/A | TIME | TIME | TIME |
| timestamp | TIMESTAMP | TIMESTAMP | TIMESTAMP | TIMESTAMP | TIMESTAMP |
| datetime | TIMESTAMP | TIMESTAMP | TIMESTAMP | TIMESTAMP | TIMESTAMP |
| timestamptz | TIMESTAMP | `<unknown>` | N/A | `<unknown>` | TIMESTAMP |
| timestampltz | TIMESTAMP | `<unknown>` | N/A | N/A | N/A |
| datetimetz | TIMESTAMP | N/A | N/A | N/A | N/A |
| datetimeltz | TIMESTAMP | N/A | N/A | N/A | N/A |
| json | VARCHAR | JSON | LONGVARCHAR | N/A | OTHER |
| enum | VARCHAR | N/A | CHAR | N/A | N/A |
| set | OTHER | N/A | N/A | N/A | N/A |
| multiset | OTHER | N/A | N/A | N/A | N/A |
| sequence | OTHER | N/A | N/A | N/A | ARRAY |
| int8 | N/A | NUMERIC | TINYINT | TINYINT | N/A |
| nchar | N/A | NCHAR | N/A | NCHAR | N/A |
| nvarchar | N/A | NVARCHAR | N/A | NVARCHAR | N/A |
| boolean | N/A | N/A | BIT | BIT | BIT |
| money | N/A | N/A | N/A | DECIMAL | DOUBLE |
| uuid | N/A | N/A | N/A | CHAR | OTHER |

### 표2. `getColumns` 의 `DATA_TYPE`

표1과 거의 동일하며, **실측 차이는 CUBRID `bit8` 한 곳뿐**: `getColumnType`=`BIT` vs `getColumns`=`BINARY`(정밀도 무관 BINARY). 나머지 셀은 표1과 동일.

| 개념 | CUBRID | Oracle | MySQL | SQL Server | PostgreSQL |
|---|---|---|---|---|---|
| int16 | SMALLINT | NUMERIC | SMALLINT | SMALLINT | SMALLINT |
| int32 | INTEGER | NUMERIC | INTEGER | INTEGER | INTEGER |
| int64 | BIGINT | NUMERIC | BIGINT | BIGINT | BIGINT |
| decimal | NUMERIC | NUMERIC | DECIMAL | DECIMAL | NUMERIC |
| real | REAL | `<unknown>` | REAL | REAL | REAL |
| double | DOUBLE | `<unknown>` | DOUBLE | DOUBLE | DOUBLE |
| char | CHAR | CHAR | CHAR | CHAR | CHAR |
| varchar | VARCHAR | VARCHAR | VARCHAR | VARCHAR | VARCHAR |
| **bit8** | **BINARY** | N/A | N/A | N/A | N/A |
| binary | BINARY | VARBINARY | BINARY | BINARY | BINARY |
| varbinary | VARBINARY | VARBINARY | VARBINARY | VARBINARY | BINARY |
| blob | BLOB | BLOB | LONGVARBINARY | VARBINARY | BINARY |
| clob | CLOB | CLOB | LONGVARCHAR | VARCHAR | VARCHAR |
| date | DATE | TIMESTAMP | DATE | DATE | DATE |
| time | TIME | N/A | TIME | TIME | TIME |
| timestamp | TIMESTAMP | TIMESTAMP | TIMESTAMP | TIMESTAMP | TIMESTAMP |
| datetime | TIMESTAMP | TIMESTAMP | TIMESTAMP | TIMESTAMP | TIMESTAMP |
| timestamptz | TIMESTAMP | `<unknown>` | N/A | `<unknown>` | TIMESTAMP |
| timestampltz | TIMESTAMP | `<unknown>` | N/A | N/A | N/A |
| datetimetz | TIMESTAMP | N/A | N/A | N/A | N/A |
| datetimeltz | TIMESTAMP | N/A | N/A | N/A | N/A |
| json | VARCHAR | JSON | LONGVARCHAR | N/A | OTHER |
| enum | VARCHAR | N/A | CHAR | N/A | N/A |
| set | OTHER | N/A | N/A | N/A | N/A |
| multiset | OTHER | N/A | N/A | N/A | N/A |
| sequence | OTHER | N/A | N/A | N/A | ARRAY |
| int8 | N/A | NUMERIC | TINYINT | TINYINT | N/A |
| nchar | N/A | NCHAR | N/A | NCHAR | N/A |
| nvarchar | N/A | NVARCHAR | N/A | NVARCHAR | N/A |
| boolean | N/A | N/A | BIT | BIT | BIT |
| money | N/A | N/A | N/A | DECIMAL | DOUBLE |
| uuid | N/A | N/A | N/A | CHAR | OTHER |

### 표3. `getBestRowIdentifier` 의 `DATA_TYPE` (UNIQUE 키 컬럼)

| 개념 | CUBRID | Oracle | MySQL | SQL Server | PostgreSQL |
|---|---|---|---|---|---|
| int16 | SMALLINT | NUMERIC | SMALLINT | SMALLINT | — |
| int32 | INTEGER | NUMERIC | INTEGER | INTEGER | — |
| int64 | BIGINT | NUMERIC | BIGINT | BIGINT | — |
| decimal | NUMERIC | NUMERIC | DECIMAL | DECIMAL | — |
| real | REAL | `<unknown>` | REAL | REAL | — |
| double | DOUBLE | `<unknown>` | DOUBLE | DOUBLE | — |
| char | CHAR | CHAR | CHAR | CHAR | — |
| varchar | VARCHAR | VARCHAR | VARCHAR | VARCHAR | — |
| bit8 | — | N/A | N/A | N/A | N/A |
| binary | — | VARBINARY | BINARY | BINARY | — |
| varbinary | — | VARBINARY | VARBINARY | VARBINARY | — |
| blob | — | — | — | — | — |
| clob | — | — | — | — | — |
| date | DATE | TIMESTAMP | DATE | DATE | — |
| time | TIME | N/A | TIME | TIME | — |
| timestamp | TIMESTAMP | TIMESTAMP | TIMESTAMP | TIMESTAMP | — |
| datetime | TIMESTAMP | TIMESTAMP | TIMESTAMP | TIMESTAMP | — |
| timestamptz | TIMESTAMP | — | N/A | `<unknown>` | — |
| timestampltz | TIMESTAMP | `<unknown>` | N/A | N/A | N/A |
| datetimetz | TIMESTAMP | N/A | N/A | N/A | N/A |
| datetimeltz | TIMESTAMP | N/A | N/A | N/A | N/A |
| json | — | — | — | N/A | — |
| enum | VARCHAR | N/A | CHAR | N/A | N/A |
| set | — | N/A | N/A | N/A | N/A |
| multiset | — | N/A | N/A | N/A | N/A |
| sequence | — | N/A | N/A | N/A | — |
| int8 | N/A | NUMERIC | TINYINT | TINYINT | N/A |
| nchar | N/A | NCHAR | N/A | NCHAR | N/A |
| nvarchar | N/A | NVARCHAR | N/A | NVARCHAR | N/A |
| boolean | N/A | N/A | BIT | BIT | — |
| money | N/A | N/A | N/A | DECIMAL | — |
| uuid | N/A | N/A | N/A | CHAR | — |

- **PostgreSQL 전 열 `—`**: pgjdbc는 명시적 PK가 아닌 `NOT NULL UNIQUE` 컬럼엔 최적 행 식별자를 반환하지 않음.
- **CUBRID `bit8`/`binary`/`varbinary`/`blob`/`clob`/`json`/컬렉션 `—`**: 비트열은 드라이버 스위치에 case 없음, JSON/LOB/컬렉션은 UNIQUE 키 불가.

### 표4. `getTypeInfo` 카탈로그

per-column이 아니라 DB가 광고하는 타입 목록이라 규모가 제각각. **CUBRID(33행)는 전량**, 나머지는 요약.

**CUBRID (33행)** — 소스 분석 표5와 1:1 일치:

| TYPE_NAME | DATA_TYPE | PRECISION |
|---|---|---|
| BIT | BIT | 8 |
| NUMERIC | TINYINT | 3 |
| NUMERIC | BIGINT | 38 |
| BIT VARYING | LONGVARBINARY | 1073741823 |
| BIT VARYING | VARBINARY | 1073741823 |
| BIT | BINARY | 1073741823 |
| VARCHAR | LONGVARCHAR | 1073741823 |
| CHAR | CHAR | 1073741823 |
| NUMERIC | NUMERIC | 38 |
| INTEGER | INTEGER | 10 |
| BIGINT | BIGINT | 19 |
| SMALLINT | SMALLINT | 5 |
| DOUBLE | FLOAT | 38 |
| FLOAT | REAL | 38 |
| DOUBLE | DOUBLE | 38 |
| VARCHAR | VARCHAR | 1073741823 |
| STRING | VARCHAR | 1073741823 |
| DATE | DATE | 10 |
| TIME | TIME | 11 |
| TIMESTAMP | TIMESTAMP | 22 |
| TIMESTAMPTZ | TIMESTAMP_WITH_TIMEZONE | 22 |
| TIMESTAMPLTZ | TIMESTAMP_WITH_TIMEZONE | 22 |
| DATETIME | TIMESTAMP | 26 |
| DATETIMETZ | TIMESTAMP_WITH_TIMEZONE | 26 |
| DATETIMELTZ | TIMESTAMP_WITH_TIMEZONE | 26 |
| BLOB | BLOB | 1073741823 |
| CLOB | CLOB | 1073741823 |
| ENUM | VARCHAR | 1073741823 |
| MULTISET | ARRAY | 1073741823 |
| SET | ARRAY | 1073741823 |
| LIST | ARRAY | 1073741823 |
| SEQUENCE | ARRAY | 1073741823 |
| JSON | VARCHAR | 1073741823 |

주: TZ 4종을 여기선 `TIMESTAMP_WITH_TIMEZONE`으로 광고 — 표1/표2의 `TIMESTAMP`와 **메서드 간 불일치**(실측 재현). `NUMERIC`이 TINYINT/BIGINT/NUMERIC 3중, `DOUBLE`이 FLOAT/DOUBLE 2중으로 노출. **정렬 안 됨**(DATA_TYPE 순 아님).

**타 DB 요약**

| DB | 행 수 | 특징 |
|---|---|---|
| Oracle | 31 | `NUMBER`가 BIT/TINYINT/BIGINT/NUMERIC/INTEGER/SMALLINT로 6중 노출; `TIMESTAMP WITH (LOCAL) TIME ZONE`·`VECTOR`·`INTERVAL*`→`<unknown>`; `DATE`→TIME·TIMESTAMP 2중; `JSON`→JSON; `NCHAR`/`NVARCHAR2`→NCHAR/NVARCHAR; BLOB/CLOB/NCLOB·STRUCT/ARRAY/REF 노출 |
| MySQL | 44 | `*_UNSIGNED` 변형 다수; `ENUM`·`SET`→CHAR; TEXT/BLOB 계열→LONGVARCHAR/LONGVARBINARY; `DATETIME`·`TIMESTAMP`→TIMESTAMP; `BOOL`→BOOLEAN; `VECTOR`→LONGVARBINARY |
| SQL Server | 37 | `datetimeoffset`·`sql_variant`→`<unknown>`; `uniqueidentifier`→CHAR; `money`/`smallmoney`→DECIMAL; `xml`/`ntext`→LONGNVARCHAR; `timestamp`(rowversion)→BINARY; `*identity` 변형 |
| PostgreSQL | ~600 | pg_catalog 전 타입 노출 — 배열 `_*`→ARRAY(대다수), range/geometry/network/`json`/`jsonb`/`uuid`/`interval`/`varbit`→OTHER, 스칼라는 표준 매핑; `xml`→SQLXML, `refcursor`→REF_CURSOR |

### 표5. 합성 결과셋 RSMD (CUBRID 전용)

`DatabaseMetaData`가 돌려주는 결과셋(예: `getTypeInfo()`)의 `ResultSetMetaData` 경로(소스 분석 표2, RSMD②). CUBRID 전용 특수 경로라 타 DB는 별도 측정하지 않음. 실측 assert 통과:

| 검증 컬럼 | 실측 `getColumnType` |
|---|---|
| TYPE_NAME | VARCHAR |
| DATA_TYPE | SMALLINT |
| CASE_SENSITIVE | BIT |

주: 표1(일반 쿼리 RSMD)이 NULL 타입을 `OTHER`로, 이 합성 경로는 `NULL`로 보고하는 차이 등은 소스 분석 표2에 정리.

## 결론

- **CUBRID 소스 분석(5개 표)이 라이브 실측으로 검증됨** — 매핑 값과 메서드 간 불일치(TZ의 `TIMESTAMP` vs `TIMESTAMP_WITH_TIMEZONE`)까지 그대로 재현.
- **표3의 JSON 행은 정정 필요**: 드라이버 스위치엔 case가 있으나 CUBRID가 JSON에 UNIQUE를 거부해 `getBestRowIdentifier`로는 도달 불가(키 불가 그룹).
- 5-DB 대조로 CUBRID의 선택이 업계 대비 어디에 있는지 맥락화: TZ를 표준 `TIMESTAMP`로 접는 건 PostgreSQL과 동류, Oracle/SQL Server는 비표준 코드로 구분 노출. 정수(Oracle=전부 NUMERIC), LOB, money·uuid, json은 벤더마다 상이.
- 전체 매트릭스(전 개념 × 5DB, DB별 getTypeInfo 전량)는 내부 산출물(`jdbc-metadata-report`)로 보관.

## 참고

- 하네스: `jdbc-testcontainer` 모듈(Testcontainers + JUnit 5, JDK 26) — 개념 카탈로그 + 범용 캡처 러너 + CUBRID 회귀 검증기 + 매트릭스 생성기
- 소스 분석 노트: [2026-07-09-cubrid-jdbc-type-mapping.md](2026-07-09-cubrid-jdbc-type-mapping.md)
- [CUBRID 11.4 매뉴얼 – 데이터 타입](https://www.cubrid.org/manual/ko/11.4/sql/datatype.html)

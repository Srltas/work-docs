# 5-DB JDBC 데이터 타입 매핑 실측 결과 (Testcontainers)

- 분류: analysis
- 날짜: 2026-07-10
- 관련: 앞선 소스 분석 노트 [CUBRID 11.4 데이터 타입 → java.sql.Types 매핑 전수 분석](2026-07-09-cubrid-jdbc-type-mapping.md)

## 목적

CUBRID JDBC 드라이버 소스로 분석했던 타입 매핑(5개 표)을 **실제 DB에 접속해 런타임으로 실측 검증**하고, 같은 개념 타입을 5개 DBMS(CUBRID/Oracle/MySQL/SQL Server/PostgreSQL)가 각각 어떤 `java.sql.Types`로 보고하는지 5개 메서드 표로 대조한다.

## 배경

기존 분석은 드라이버 소스 정적 분석이었다. 이를 근거로만 두지 않고, Testcontainers로 5개 DB를 실제 기동해 개념 타입별로 컬럼을 만들고 JDBC 메타데이터를 실측했다. CUBRID는 분석 5개 표를 그대로 assert로 걸어 회귀 검증했다(스위트 10/10 통과).

## 범위 / 방법

- **하네스**: 개념 타입 카탈로그 × 메타데이터 메서드 × 5 DB. 개념마다 컬럼 생성 후 캡처 → DB별 JSON → 대조 매트릭스 생성. Testcontainers, JUnit 5, JDK 26, Docker.
- **표기**: 셀 값은 `java.sql.Types` 상수명(접두어 `Types.` 생략). `N/A`=해당 DB에 개념 없음 · `—`=메서드가 값 미반환(NULL/미해당) · `<unknown>`=`java.sql.Types` 밖의 드라이버 고유 코드.
- **본 노트의 표1~3은 CUBRID가 가진 개념만** 표시한다. CUBRID에 없는 개념(int8·nchar·nvarchar·boolean·money·uuid)은 제외했다(초점이 CUBRID 매핑 검증이므로). 타 DB의 getTypeInfo 전체 목록은 문서 끝 [부록](#부록-db별-gettypeinfo-카탈로그) 참조.

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

### 1) CUBRID — 소스 분석이 실측으로 확인됨

- 표1(getColumnType)·표2(합성 RSMD)·표3(getColumns)·표4(getBestRowIdentifier)·표5(getTypeInfo)의 기대값이 **라이브 CUBRID 동작과 전부 일치**.
- `getTypeInfo`는 TZ 타입을 `TIMESTAMP_WITH_TIMEZONE`으로 광고하지만 `getColumnType`/`getColumns`는 평범한 `TIMESTAMP`로 보고 — **메서드 간 불일치가 실측으로 재현됨**. `NUMERIC→TINYINT/BIGINT`, `DOUBLE→FLOAT`, 컬렉션→`ARRAY` 행도 그대로 관찰.
- **발견**: 표3의 `getBestRowIdentifier`에서 CUBRID는 비트열/컬렉션/LOB/JSON에 값을 못 주고, PK 기반으론 아무 것도 못 준다(별도 버그 노트로 정리). 컬렉션의 `ARRAY` 광고도 `getArray()` 미구현이라 사실상 거짓 약속.

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

### 표2. `getColumns` 의 `DATA_TYPE`

표1과 거의 동일하며, **실측 차이는 CUBRID `bit8` 한 곳뿐**(`getColumnType`=`BIT` vs `getColumns`=`BINARY`).

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
| set | — | N/A | N/A | N/A | — |
| multiset | — | N/A | N/A | N/A | — |
| sequence | — | N/A | N/A | N/A | — |

- **PostgreSQL 전 열 `—`**: pgjdbc는 명시적 PK가 아닌 `NOT NULL UNIQUE` 컬럼엔 최적 행 식별자를 반환하지 않음(하네스가 CUBRID에 맞춰 UNIQUE로 만든 영향 — CUBRID는 PK가 잡히지 않아 UNIQUE 필요).
- **CUBRID `bit8`/`binary`/`varbinary`/`blob`/`clob`/`json`/컬렉션 `—`**: 비트열은 드라이버 스위치에 case 없음, JSON/LOB/컬렉션은 UNIQUE 키 불가.

### 표4. `getTypeInfo` 카탈로그 (CUBRID)

CUBRID getTypeInfo 전량(33행) — 소스 분석 표5와 1:1 일치. **타 4개 DB의 getTypeInfo는 문서 끝 [부록](#부록-db별-gettypeinfo-카탈로그)에** (Oracle·MySQL·SQL Server 전량, PostgreSQL 대표 발췌).

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

주: TZ 4종을 여기선 `TIMESTAMP_WITH_TIMEZONE`으로 광고 — 표1/표2의 `TIMESTAMP`와 **메서드 간 불일치**(실측 재현). `NUMERIC`이 TINYINT/BIGINT/NUMERIC 3중, `DOUBLE`이 FLOAT/DOUBLE 2중, 컬렉션이 `ARRAY`(하지만 `getArray()` 미구현). **정렬 안 됨**(DATA_TYPE 순 아님).

### 표5. 합성 결과셋 RSMD (CUBRID 전용)

`DatabaseMetaData`가 돌려주는 결과셋(예: `getTypeInfo()`) **자체의** `ResultSetMetaData` 경로(소스 분석 표2, RSMD②). 일반 쿼리 결과와 다른 코드 경로라 CUBRID만 측정. 실측 assert 통과:

| 검증 컬럼 | 실측 `getColumnType` |
|---|---|
| TYPE_NAME | VARCHAR |
| DATA_TYPE | SMALLINT |
| CASE_SENSITIVE | BIT |

주: 표1(일반 쿼리 RSMD)이 NULL 타입을 `OTHER`로, 이 합성 경로는 `NULL`로 보고하는 차이 등은 소스 분석 표2에 정리.

### 벤더 간 두드러진 차이

- **Oracle은 정수·decimal을 `NUMERIC` 하나로 수렴** (네이티브 정수 계열이 `NUMBER` 뿐): int16/32/64/decimal이 한 열로 납작해짐.
- **Oracle `BINARY_FLOAT`/`BINARY_DOUBLE`, TZ 타입, SQL Server `datetimeoffset`은 `java.sql.Types` 밖의 고유 코드**(`<unknown>`)로 나오고 클래스도 독자(`oracle.sql.TIMESTAMPTZ`, `microsoft.sql.DateTimeOffset`). 반면 **CUBRID·PostgreSQL은 TZ를 표준 `TIMESTAMP`로 접음**(타임존 정보 유실 방향).
- **Oracle `DATE`는 `TIMESTAMP`로 보고**(Oracle DATE에 시각 성분 포함), **순수 `TIME` 타입은 없음**(N/A).
- **LOB 표현이 제각각**: BLOB/CLOB을 실제 `Types.BLOB`/`CLOB`으로 쓰는 건 CUBRID·Oracle뿐. MySQL=`LONGVARBINARY`/`LONGVARCHAR`, SQL Server=`VARBINARY`/`VARCHAR`, PostgreSQL=`BINARY`/`VARCHAR`.
- **json**: Oracle만 네이티브 `Types.JSON`(23ai). CUBRID=`VARCHAR`, MySQL=`LONGVARCHAR`, PostgreSQL=`OTHER`, SQL Server=미지원(N/A).
- **컬렉션**: CUBRID는 `getColumnType`/`getColumns`에서 `OTHER`, `getTypeInfo`에선 `ARRAY`로 광고하나 `getArray()` 미구현 → `ARRAY`는 거짓 약속. PostgreSQL 배열만 실제 `java.sql.Array`를 이행.
- **`getBestRowIdentifier`**: PostgreSQL은 UNIQUE-only 컬럼에 값 미반환(PK 필요). CUBRID는 비트열/컬렉션/LOB/JSON 및 PK에서 미반환.

## 결론

- **CUBRID 소스 분석(5개 표)이 라이브 실측으로 검증됨** — 매핑 값과 메서드 간 불일치(TZ의 `TIMESTAMP` vs `TIMESTAMP_WITH_TIMEZONE`)까지 그대로 재현.
- **표3의 JSON 행 및 PK 미반환**: 별도 버그 노트로 정리(`getBestRowIdentifier`가 PK를 못 봄).
- 5-DB 대조로 CUBRID의 선택이 업계 대비 어디에 있는지 맥락화: TZ를 표준 `TIMESTAMP`로 접는 건 PostgreSQL과 동류, Oracle/SQL Server는 비표준 코드로 구분 노출. 정수(Oracle=전부 NUMERIC)·LOB·json은 벤더마다 상이.
- 전체 매트릭스(전 개념 × 5DB, DB별 getTypeInfo 전량)는 내부 산출물(`jdbc-metadata-report`)로 보관.

## 부록: DB별 getTypeInfo 카탈로그

값은 `java.sql.Types` 상수명(접두어 생략), `<unknown>`=`java.sql.Types` 밖 고유 코드.

### Oracle (전량, 31행)

| TYPE_NAME | DATA_TYPE | PRECISION |
|---|---|---|
| VECTOR | `<unknown>` | 0 |
| INTERVALDS | `<unknown>` | 4 |
| INTERVALYM | `<unknown>` | 5 |
| TIMESTAMP WITH LOCAL TIME ZONE | `<unknown>` | 11 |
| TIMESTAMP WITH TIME ZONE | `<unknown>` | 13 |
| NCHAR | NCHAR | 2000 |
| NVARCHAR2 | NVARCHAR | 4000 |
| NUMBER | BIT | 1 |
| NUMBER | TINYINT | 3 |
| NUMBER | BIGINT | 38 |
| LONG RAW | LONGVARBINARY | 2147483647 |
| RAW | VARBINARY | 2000 |
| LONG | LONGVARCHAR | 2147483647 |
| CHAR | CHAR | 2000 |
| NUMBER | NUMERIC | 38 |
| NUMBER | INTEGER | 10 |
| NUMBER | SMALLINT | 5 |
| FLOAT | FLOAT | 63 |
| REAL | REAL | 63 |
| VARCHAR2 | VARCHAR | 4000 |
| BOOLEAN | BOOLEAN | 0 |
| DATE | TIME | 7 |
| DATE | TIMESTAMP | 7 |
| TIMESTAMP | TIMESTAMP | 11 |
| STRUCT | STRUCT | 0 |
| ARRAY | ARRAY | 0 |
| BLOB | BLOB | -1 |
| CLOB | CLOB | -1 |
| REF | REF | 0 |
| NCLOB | NCLOB | -1 |
| JSON | JSON | -1 |

### MySQL (전량, 44행)

| TYPE_NAME | DATA_TYPE | PRECISION |
|---|---|---|
| BIT | BIT | 1 |
| TINYINT | TINYINT | 3 |
| TINYINT UNSIGNED | TINYINT | 3 |
| BIGINT | BIGINT | 19 |
| BIGINT UNSIGNED | BIGINT | 20 |
| LONG VARBINARY | LONGVARBINARY | 16777215 |
| MEDIUMBLOB | LONGVARBINARY | 16777215 |
| LONGBLOB | LONGVARBINARY | 2147483647 |
| BLOB | LONGVARBINARY | 65535 |
| VECTOR | LONGVARBINARY | 65532 |
| VARBINARY | VARBINARY | 65535 |
| TINYBLOB | VARBINARY | 255 |
| BINARY | BINARY | 255 |
| LONG VARCHAR | LONGVARCHAR | 16777215 |
| MEDIUMTEXT | LONGVARCHAR | 16777215 |
| LONGTEXT | LONGVARCHAR | 2147483647 |
| TEXT | LONGVARCHAR | 65535 |
| CHAR | CHAR | 255 |
| ENUM | CHAR | 65535 |
| SET | CHAR | 64 |
| DECIMAL | DECIMAL | 65 |
| NUMERIC | DECIMAL | 65 |
| INTEGER | INTEGER | 10 |
| INT | INTEGER | 10 |
| MEDIUMINT | INTEGER | 7 |
| INTEGER UNSIGNED | INTEGER | 10 |
| INT UNSIGNED | INTEGER | 10 |
| MEDIUMINT UNSIGNED | INTEGER | 8 |
| SMALLINT | SMALLINT | 5 |
| SMALLINT UNSIGNED | SMALLINT | 5 |
| FLOAT | REAL | 12 |
| DOUBLE | DOUBLE | 22 |
| DOUBLE PRECISION | DOUBLE | 22 |
| REAL | DOUBLE | 22 |
| DOUBLE UNSIGNED | DOUBLE | 22 |
| DOUBLE PRECISION UNSIGNED | DOUBLE | 22 |
| VARCHAR | VARCHAR | 65535 |
| TINYTEXT | VARCHAR | 255 |
| BOOL | BOOLEAN | 3 |
| DATE | DATE | 10 |
| YEAR | DATE | 4 |
| TIME | TIME | 16 |
| DATETIME | TIMESTAMP | 26 |
| TIMESTAMP | TIMESTAMP | 26 |

### SQL Server (전량, 37행)

| TYPE_NAME | DATA_TYPE | PRECISION |
|---|---|---|
| datetimeoffset | `<unknown>` | 34 |
| time | TIME | 16 |
| xml | LONGNVARCHAR | 0 |
| sql_variant | `<unknown>` | 8000 |
| uniqueidentifier | CHAR | 36 |
| ntext | LONGNVARCHAR | 1073741823 |
| nvarchar | NVARCHAR | 4000 |
| sysname | NVARCHAR | 128 |
| nchar | NCHAR | 4000 |
| bit | BIT | 1 |
| tinyint | TINYINT | 3 |
| tinyint identity | TINYINT | 3 |
| bigint | BIGINT | 19 |
| bigint identity | BIGINT | 19 |
| image | LONGVARBINARY | 2147483647 |
| varbinary | VARBINARY | 8000 |
| binary | BINARY | 8000 |
| timestamp | BINARY | 8 |
| text | LONGVARCHAR | 2147483647 |
| char | CHAR | 8000 |
| numeric | NUMERIC | 38 |
| numeric() identity | NUMERIC | 38 |
| decimal | DECIMAL | 38 |
| money | DECIMAL | 19 |
| smallmoney | DECIMAL | 10 |
| decimal() identity | DECIMAL | 38 |
| int | INTEGER | 10 |
| int identity | INTEGER | 10 |
| smallint | SMALLINT | 5 |
| smallint identity | SMALLINT | 5 |
| float | DOUBLE | 53 |
| real | REAL | 24 |
| varchar | VARCHAR | 8000 |
| date | DATE | 10 |
| datetime2 | TIMESTAMP | 27 |
| datetime | TIMESTAMP | 23 |
| smalldatetime | TIMESTAMP | 16 |

### PostgreSQL (대표 발췌)

전체 ~600행(pg_catalog 전 타입 노출). 배열 `_*`→`ARRAY`, range/geometry/network/각종 pg_catalog 타입→`OTHER` 다수는 생략하고 대표 타입만:

| TYPE_NAME | DATA_TYPE | PRECISION |
|---|---|---|
| bool | BIT | 0 |
| int2 | SMALLINT | 0 |
| int4 | INTEGER | 0 |
| int8 | BIGINT | 0 |
| numeric | NUMERIC | 1000 |
| float4 | REAL | 0 |
| float8 | DOUBLE | 0 |
| money | DOUBLE | 0 |
| bytea | BINARY | 0 |
| char | CHAR | 0 |
| varchar | VARCHAR | 10485760 |
| text | VARCHAR | 0 |
| bit | BIT | 83886080 |
| varbit | OTHER | 83886080 |
| date | DATE | 0 |
| time | TIME | 6 |
| timestamp | TIMESTAMP | 6 |
| timestamptz | TIMESTAMP | 6 |
| interval | OTHER | 6 |
| uuid | OTHER | 0 |
| json | OTHER | 0 |
| jsonb | OTHER | 0 |
| xml | SQLXML | 0 |
| _int4 (int4 배열) | ARRAY | 0 |
| refcursor | REF_CURSOR | 0 |

## 참고

- 하네스: `jdbc-testcontainer` 모듈(Testcontainers + JUnit 5, JDK 26) — 개념 카탈로그 + 범용 캡처 러너 + CUBRID 회귀 검증기 + 매트릭스 생성기
- 소스 분석 노트: [2026-07-09-cubrid-jdbc-type-mapping.md](2026-07-09-cubrid-jdbc-type-mapping.md)
- getBestRowIdentifier PK 버그 → 작업 기록: [APIS-1086](../APIS-1086/APIS-1086-getbestrowidentifier-pk.md)
- [CUBRID 11.4 매뉴얼 – 데이터 타입](https://www.cubrid.org/manual/ko/11.4/sql/datatype.html)

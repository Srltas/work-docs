# 5-DB JDBC 데이터 타입 매핑 실측 결과 (Testcontainers)

- 분류: analysis
- 날짜: 2026-07-10
- 관련: 앞선 소스 분석 노트 [CUBRID 11.4 데이터 타입 → java.sql.Types 매핑 전수 분석](2026-07-09-cubrid-jdbc-type-mapping.md)

## 목적

CUBRID JDBC 드라이버 소스로 분석했던 타입 매핑(5개 표)을 **실제 DB에 접속해 런타임으로 실측 검증**하고, 같은 개념 타입을 5개 DBMS(CUBRID/Oracle/MySQL/SQL Server/PostgreSQL)가 각각 어떤 `java.sql.Types`로 보고하는지 대조한다.

## 배경

기존 분석은 드라이버 소스 정적 분석이었다. 이를 근거로만 두지 않고, Testcontainers로 5개 DB를 실제 기동해 개념 타입별로 컬럼을 만들고 JDBC 메타데이터 4종을 실측했다. CUBRID는 분석 5개 표를 그대로 assert로 걸어 회귀 검증했다.

## 범위 / 방법

- **하네스**: 개념 타입 카탈로그(32종) × 메타데이터 4종 × 5 DB. 각 개념마다 컬럼 생성 후 아래를 캡처 → DB별 JSON → 대조 매트릭스 생성.
  - `getColumnType` (실행 SELECT의 ResultSetMetaData)
  - `getColumns` 의 `DATA_TYPE`
  - `getBestRowIdentifier` 의 `DATA_TYPE` (UNIQUE 키 컬럼)
  - `getTypeInfo` (드라이버가 광고하는 타입 카탈로그)
- **실행 환경**: Testcontainers, JUnit 5, JDK 26, Docker. 각 DB 컨테이너 이미지로 기동.
- **CUBRID 회귀**: 분석 문서의 표1(getColumnType)·표2(합성 RSMD)·표3(getColumns)·표4(getBestRowIdentifier)·표5(getTypeInfo)를 assert로 검증 → **CUBRID 스위트 10/10 통과**.

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
- 표5의 특이점도 실측 확인: `getTypeInfo`는 TZ 타입을 `TIMESTAMP_WITH_TIMEZONE`으로 광고하지만, `getColumnType`/`getColumns`는 같은 타입을 평범한 `TIMESTAMP`로 보고 — **메서드 간 불일치가 실측으로 재현됨**. 또 `NUMERIC→TINYINT`/`NUMERIC→BIGINT`, `DOUBLE→FLOAT`, 컬렉션→`ARRAY` 행도 그대로 관찰.
- **새 발견**: 표4의 `JSON→VARCHAR` 행은 실제로 **키로 도달 불가**. CUBRID가 JSON 컬럼에 UNIQUE 제약을 거부(`json domain not suitable for UNIQUE`)해 `getBestRowIdentifier`가 그 컬럼을 볼 수 없음. BIT·BIT VARYING도 `getBestRowIdentifier`에서 값 미반환(코드에 case 없음).

### 2) 개념 타입 × 5DB — `getColumnType` (핵심 발췌)

`N/A` = 해당 DB에 개념 없음 · `<unknown>` = `java.sql.Types` 밖의 드라이버 고유 코드.

| 개념 | CUBRID | Oracle | MySQL | SQL Server | PostgreSQL |
|---|---|---|---|---|---|
| int16 | SMALLINT | **NUMERIC** | SMALLINT | SMALLINT | SMALLINT |
| int32 | INTEGER | **NUMERIC** | INTEGER | INTEGER | INTEGER |
| int64 | BIGINT | **NUMERIC** | BIGINT | BIGINT | BIGINT |
| decimal | NUMERIC | NUMERIC | DECIMAL | DECIMAL | NUMERIC |
| real | REAL | **`<unknown>`** | REAL | REAL | REAL |
| double | DOUBLE | **`<unknown>`** | DOUBLE | DOUBLE | DOUBLE |
| char | CHAR | CHAR | CHAR | CHAR | CHAR |
| varchar | VARCHAR | VARCHAR | VARCHAR | VARCHAR | VARCHAR |
| bit8 | BIT | N/A | N/A | N/A | N/A |
| varbinary | VARBINARY | VARBINARY | VARBINARY | VARBINARY | BINARY |
| blob | BLOB | BLOB | LONGVARBINARY | VARBINARY | BINARY |
| clob | CLOB | CLOB | LONGVARCHAR | VARCHAR | VARCHAR |
| date | DATE | **TIMESTAMP** | DATE | DATE | DATE |
| time | TIME | **N/A** | TIME | TIME | TIME |
| timestamp | TIMESTAMP | TIMESTAMP | TIMESTAMP | TIMESTAMP | TIMESTAMP |
| timestamptz | TIMESTAMP | **`<unknown>`** | N/A | **`<unknown>`** | TIMESTAMP |
| boolean | N/A | N/A | BIT | BIT | BIT |
| json | VARCHAR | **JSON** | LONGVARCHAR | N/A | OTHER |
| enum | VARCHAR | N/A | **CHAR** | N/A | N/A |
| set | OTHER | N/A | N/A | N/A | N/A |
| money | N/A | N/A | N/A | **DECIMAL** | **DOUBLE** |
| uuid | N/A | N/A | N/A | **CHAR** | **OTHER** |

(`getColumns`는 위와 거의 동일. 단 CUBRID `bit8`만 getColumnType=`BIT` vs getColumns=`BINARY`로 갈림.)

### 3) 벤더 간 두드러진 차이

- **Oracle은 모든 정수·decimal을 `NUMERIC` 하나로 수렴** (네이티브 정수 계열이 `NUMBER` 뿐). int8/16/32/64/decimal이 매트릭스에서 한 열로 납작해짐.
- **Oracle `BINARY_FLOAT`/`BINARY_DOUBLE`, TZ 타입, SQL Server `datetimeoffset`은 `java.sql.Types` 밖의 고유 코드**(`<unknown>`)로 나오고 클래스도 독자(`oracle.sql.TIMESTAMPTZ`, `microsoft.sql.DateTimeOffset`). 반면 **CUBRID·PostgreSQL은 TZ를 표준 `TIMESTAMP`로 접음**(타임존 정보 유실 방향).
- **Oracle `DATE`는 `TIMESTAMP`로 보고**(Oracle DATE에 시각 성분 포함), **순수 `TIME` 타입은 없음**(N/A).
- **LOB 표현이 제각각**: BLOB/CLOB을 실제 `Types.BLOB`/`CLOB`으로 쓰는 건 CUBRID·Oracle뿐. MySQL=`LONGVARBINARY`/`LONGVARCHAR`, SQL Server=`VARBINARY`/`VARCHAR`, PostgreSQL=`BINARY`/`VARCHAR`.
- **money·uuid 벤더차**: SQL Server `money`→`DECIMAL` vs PostgreSQL `money`→`DOUBLE`; SQL Server `uniqueidentifier`→`CHAR` vs PostgreSQL `uuid`→`OTHER`.
- **json**: Oracle만 네이티브 `Types.JSON`(23ai). CUBRID=`VARCHAR`, MySQL=`LONGVARCHAR`, PostgreSQL=`OTHER`, SQL Server=미지원(N/A).
- **`getBestRowIdentifier`**: **PostgreSQL은 모든 개념에서 값 미반환** — pgjdbc는 명시적 PK가 아닌 `NOT NULL UNIQUE`엔 최적 행 식별자를 돌려주지 않음. CUBRID는 비트열/컬렉션/LOB/JSON에서 미반환, Oracle은 TZ/LOB에서 미반환.
- **`getTypeInfo` 카탈로그 규모 격차**: PostgreSQL은 pg_catalog 전 타입을 노출해 ~600행(배열 `_*`→`ARRAY`, range/기타→`OTHER`)인 반면, CUBRID 33행·MySQL 44행·SQL Server 37행·Oracle 31행.

## 결론

- **CUBRID 소스 분석(5개 표)이 라이브 실측으로 검증됨** — 매핑 값과 메서드 간 불일치까지 그대로 재현. 소스 기반 분석의 신뢰성 확보.
- **표4의 JSON 행은 정정 필요**: 코드상 case는 있으나 CUBRID가 JSON에 UNIQUE를 거부해 `getBestRowIdentifier`로는 도달 불가(키 불가 그룹).
- 5-DB 대조로 CUBRID의 선택이 업계 대비 어디에 있는지 맥락화됨 — TZ를 표준 `TIMESTAMP`로 접는 건 PostgreSQL과 동류이고, Oracle/SQL Server는 비표준 코드로 구분해 노출. LOB·money·uuid·json은 벤더마다 상이.
- 전체 매트릭스(§1~§4 전 개념 × 5DB + DB별 getTypeInfo 전량)는 내부 산출물(`jdbc-metadata-report`)로 보관.

## 참고

- 하네스: `jdbc-testcontainer` 모듈(Testcontainers + JUnit 5, JDK 26) — 개념 카탈로그 + 범용 캡처 러너 + CUBRID 회귀 검증기 + 매트릭스 생성기
- 소스 분석 노트: [2026-07-09-cubrid-jdbc-type-mapping.md](2026-07-09-cubrid-jdbc-type-mapping.md)
- [CUBRID 11.4 매뉴얼 – 데이터 타입](https://www.cubrid.org/manual/ko/11.4/sql/datatype.html)

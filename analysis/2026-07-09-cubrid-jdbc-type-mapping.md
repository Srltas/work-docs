# CUBRID 11.4 데이터 타입 → `java.sql.Types` 매핑 전수 분석

- 분류: analysis
- 날짜: 2026-07-09
- 관련: [CUBRID 11.4 매뉴얼 – 데이터 타입](https://www.cubrid.org/manual/ko/11.4/sql/datatype.html), CUBRID JDBC 드라이버 소스

## 목적

CUBRID JDBC 드라이버가 CUBRID의 각 데이터 타입을 `java.sql.Types`로 올바르게 매핑하는지 전수 조사한다. (1) CUBRID 11.4가 제공하는 데이터 타입 목록을 매뉴얼에서 확보하고, (2) 드라이버가 실제로 반환하는 `java.sql.Types` 값을 소스에서 확인하여 대조한다.

## 배경

JDBC 메타데이터 검증 과정에서 타입 매핑이 부정확해 보인다는 의심이 있었다. 드라이버는 U_TYPE → `java.sql.Types` 매핑을 **최소 5개 지점**에서 각각 따로 수행하며, 지점마다 처리 대상·결과가 다르다.

- **RSMD** `java.sql.ResultSetMetaData` — `CUBRIDResultSetMetaData`에 **생성자가 2개** 있고 생성자별로 매핑이 다르다.
  - ① `CUBRIDResultSetMetaData(UColumnInfo[])` — 일반 SQL 쿼리 결과셋용 (`switch`)
  - ② `CUBRIDResultSetMetaData(CUBRIDResultSetWithoutQuery)` — 드라이버가 서버 쿼리 없이 만든 **합성 결과셋**(주로 DatabaseMetaData 반환 ResultSet)용 (`if` 체인)
- **DBMD** `java.sql.DatabaseMetaData` — 컬럼 타입을 보고하는 메서드가 여럿이다.
  - `getColumns()` 의 `DATA_TYPE`
  - `getBestRowIdentifier()` 의 `DATA_TYPE`
  - `getTypeInfo()` — 드라이버가 광고하는 지원 타입 카탈로그

이 5개 지점이 서로 일치하는지가 핵심 점검 대상이었다.

## 범위 / 방법

- **CUBRID 타입 정의**: 11.4 매뉴얼 데이터 타입 페이지에서 숫자/문자/비트/날짜·시간/컬렉션/LOB/ENUM/JSON/OID 전수 확보. deprecation·삭제 상태도 원문으로 재확인.
- **드라이버 매핑**: 소스 직접 확인. 아래 line 번호는 확인 시점 기준(버전에 따라 이동 가능).
  - 내부 타입 상수: `cubrid/jdbc/jci/UUType.java` (`U_TYPE_*`, 0~34)
  - RSMD 생성자 ①: `cubrid/jdbc/driver/CUBRIDResultSetMetaData.java` line 62, `switch` line 99
  - RSMD 생성자 ②: 같은 파일 line 453, `if` 체인 line 470~520
  - DBMD `getColumns`: `cubrid/jdbc/driver/CUBRIDDatabaseMetaData.java` line 1160~1247
  - DBMD `getBestRowIdentifier`: 같은 파일 line 1407, `switch` line 1513~1619
  - DBMD `getTypeInfo`: 같은 파일 line 1945~2015 (병렬 배열 카탈로그)
- **제외(엔진에서 제거된 타입)**: `NCHAR`, `NCHAR VARYING`, `MONETARY`.
  - `NCHAR` / `NCHAR VARYING` — 매뉴얼 과거형("9.0 버전부터 더 이상 지원하지 않으며, 대신 CHAR, VARCHAR 타입을 사용").
  - `MONETARY` — 엔진에서 제거됨. 단 11.4 매뉴얼 문구는 미래형("MONETARY 타입은 제거될 예정이며(deprecated)")으로 남아 있어 표기가 다름.
  - 세 타입 모두 드라이버 코드에는 매핑이 잔존하나 서버가 방출하지 않는 데드 경로.
- **판정 범례**: ✅ 적절 · ⚠️ 검토 필요 · 그 외 비고 참조.

## 발견 / 관찰

### 표 1. RSMD ① — `CUBRIDResultSetMetaData(UColumnInfo[] col_info)`

일반 SQL 쿼리 결과셋에서 `getColumnType()`이 반환하는 값. (line 62, `switch` line 99)

| # | CUBRID 타입 (별칭) | `U_TYPE` (값) | `java.sql.Types` | line | 판정 |
|---|---|---|---|---|---|
| 1 | SHORT / SMALLINT | SHORT(9) | `SMALLINT` | :151 | ✅ |
| 2 | INTEGER / INT | INT(8) | `INTEGER` | :160 | ✅ |
| 3 | BIGINT | BIGINT(21) | `BIGINT` | :171 | ✅ |
| 4 | NUMERIC / DECIMAL / DEC | NUMERIC(7) | `NUMERIC` *(mysql 모드 `DECIMAL`)* | :189 | ✅ |
| 5 | FLOAT / REAL | FLOAT(11) | `REAL` | :177 | ✅ *(규약상 정상, 관찰 참조)* |
| 6 | DOUBLE / DOUBLE PRECISION | DOUBLE(12) | `DOUBLE` | :183 | ✅ |
| 7 | CHAR / CHARACTER | CHAR(1) | `CHAR` | :100 | ✅ |
| 8 | VARCHAR / CHAR VARYING / STRING | VARCHAR(2) | `VARCHAR` | :109 | ✅ |
| 9 | BIT | BIT(5) | prec==8 → `BIT`, 그 외 → `BINARY` | :127 | ⚠️ BIT(8)을 Boolean 취급 |
| 10 | BIT VARYING | VARBIT(6) | `VARBINARY` | :142 | ✅ |
| 11 | DATE | DATE(13) | `DATE` | :209 | ✅ |
| 12 | TIME | TIME(14) | `TIME` | :215 | ✅ |
| 13 | TIMESTAMP | TIMESTAMP(15) | `TIMESTAMP` | :221 | ✅ |
| 14 | DATETIME | DATETIME(22) | `TIMESTAMP` | :239 | ✅ |
| 15 | TIMESTAMPTZ / TIMESTAMP WITH TIME ZONE | TIMESTAMPTZ(29) | `TIMESTAMP` | :227 | ⚠️ TZ 손실 |
| 16 | TIMESTAMPLTZ | TIMESTAMPLTZ(30) | `TIMESTAMP` | :233 | ⚠️ |
| 17 | DATETIMETZ / DATETIME WITH TIME ZONE | DATETIMETZ(31) | `TIMESTAMP` | :245 | ⚠️ |
| 18 | DATETIMELTZ | DATETIMELTZ(32) | `TIMESTAMP` | :251 | ⚠️ |
| 19 | SET | SET(16) | `OTHER` | :270 | ⚠️ getTypeInfo는 `ARRAY` |
| 20 | MULTISET | MULTISET(17) | `OTHER` | :273 | ⚠️ |
| 21 | LIST / SEQUENCE | SEQUENCE(18) | `OTHER` | :278 | ⚠️ |
| 22 | BLOB | BLOB(23) | `BLOB` | :426 | ✅ |
| 23 | CLOB | CLOB(24) | `CLOB` | :432 | ✅ |
| 24 | ENUM | ENUM(25) | `VARCHAR` | :118 | ✅ *(표준 부재)* |
| 25 | JSON | JSON(34) | `VARCHAR` | :438 | ⚠️ 표준 부재(`OTHER`도 가능) |
| 26 | object / OID | OBJECT(19) | `OTHER` | :264 | ✅ |
| 27 | (NULL 타입 컬럼) | NULL(0) | `OTHER` | :257 | ⚠️ `Types.NULL` 주석 처리됨 |

`switch`에 case 없음 → `default`(line 447) → `Types.NULL(0)`: 부호 없는 정수 `USHORT(26)`/`UINT(27)`/`UBIGINT(28)`, `RESULTSET(20)`.

### 표 2. RSMD ② — `CUBRIDResultSetMetaData(CUBRIDResultSetWithoutQuery r)`

드라이버가 서버 쿼리 없이 만든 **합성 결과셋**(대부분 DatabaseMetaData가 돌려주는 ResultSet)의 컬럼 타입. `switch`가 아니라 `if` 체인(line 470~520)이며 **7개 타입만** 처리한다. 이 결과셋 컬럼은 이 7종만 쓰이도록 설계되어 있으나, 미해당 타입은 조용히 `Types.NULL(0)`이 된다.

| # | 처리 U_TYPE | `java.sql.Types` | 클래스명 | line | 비고 |
|---|---|---|---|---|---|
| 1 | BIT(5) | `BIT` | `byte[]` | :470 | **생성자①과 다름**: 정밀도 무관 항상 `BIT`, prec 강제 1, 클래스도 `byte[]`(Boolean 아님) |
| 2 | INT(8) | `INTEGER` | `java.lang.Integer` | :476 | |
| 3 | SHORT(9) | `SMALLINT` | `java.lang.Short` | :482 | |
| 4 | VARCHAR(2) | `VARCHAR` | `java.lang.String` | :488 | |
| 5 | ENUM(25) | `VARCHAR` | `java.lang.String` | :497 | |
| 6 | JSON(34) | `VARCHAR` | `java.lang.String` | :506 | |
| 7 | NULL(0) | `NULL` | `""` | :515 | **생성자①은 `OTHER`** — 여기선 `NULL` |
| — | 그 외 전부 | `NULL`(0, 배열 기본값) | — | — | `if` 체인 미해당 → `col_type` 초기값 0 |

### 표 3. DBMD — `getColumns()` 의 `DATA_TYPE`

카탈로그로 컬럼 목록을 조회할 때의 타입. (`value[4]`, line 1160~1247)

| # | CUBRID 타입 (별칭) | `U_TYPE` (값) | `java.sql.Types` | line | 판정 |
|---|---|---|---|---|---|
| 1 | SHORT / SMALLINT | SHORT(9) | `SMALLINT` | :1181 | ✅ |
| 2 | INTEGER / INT | INT(8) | `INTEGER` | :1187 | ✅ |
| 3 | BIGINT | BIGINT(21) | `BIGINT` | :1184 | ✅ |
| 4 | NUMERIC / DECIMAL / DEC | NUMERIC(7) | `NUMERIC` *(mysql 분기 없음)* | :1190 | ✅ |
| 5 | FLOAT / REAL | FLOAT(11) | `REAL` | :1193 | ✅ |
| 6 | DOUBLE / DOUBLE PRECISION | DOUBLE(12) | `DOUBLE` | :1196 | ✅ |
| 7 | CHAR / CHARACTER | CHAR(1) | `CHAR` | :1166 | ✅ |
| 8 | VARCHAR / CHAR VARYING / STRING | VARCHAR(2) | `VARCHAR` | :1169 | ✅ |
| 9 | BIT | BIT(5) | **항상 `BINARY`** (정밀도 무관) | :1160 | ⚠️ RSMD①과 불일치 |
| 10 | BIT VARYING | VARBIT(6) | `VARBINARY` | :1163 | ✅ |
| 11 | DATE | DATE(13) | `DATE` | :1205 | ✅ |
| 12 | TIME | TIME(14) | `TIME` | :1202 | ✅ |
| 13 | TIMESTAMP | TIMESTAMP(15) | `TIMESTAMP` | :1208 | ✅ |
| 14 | DATETIME | DATETIME(22) | `TIMESTAMP` | :1211 | ✅ |
| 15 | TIMESTAMPTZ | TIMESTAMPTZ(29) | `TIMESTAMP` | :1232 | ⚠️ TZ 손실 |
| 16 | TIMESTAMPLTZ | TIMESTAMPLTZ(30) | `TIMESTAMP` | :1235 | ⚠️ |
| 17 | DATETIMETZ | DATETIMETZ(31) | `TIMESTAMP` | :1238 | ⚠️ |
| 18 | DATETIMELTZ | DATETIMELTZ(32) | `TIMESTAMP` | :1241 | ⚠️ |
| 19 | SET | SET(16) | `OTHER` | :1217 | ⚠️ getTypeInfo는 `ARRAY` |
| 20 | MULTISET | MULTISET(17) | `OTHER` | :1220 | ⚠️ |
| 21 | LIST / SEQUENCE | SEQUENCE(18) | `OTHER` | :1223 | ⚠️ |
| 22 | BLOB | BLOB(23) | `BLOB` | :1226 | ✅ |
| 23 | CLOB | CLOB(24) | `CLOB` | :1229 | ✅ |
| 24 | ENUM | ENUM(25) | `VARCHAR` | :1172 | ✅ |
| 25 | JSON | JSON(34) | `VARCHAR` | :1244 | ⚠️ |
| 26 | object / OID | OBJECT(19) | `OTHER` | :1214 | ✅ |
| 27 | (NULL 타입 컬럼) | NULL(0) | **처리 없음** (`DATA_TYPE` 미설정) | — | ⚠️ 항목 자체가 없음 |

if/else 체인에 항목 없음: `NULL(0)`, 부호 없는 정수 `USHORT`/`UINT`/`UBIGINT`, `RESULTSET(20)`. 미매칭 시 `value[4]`가 null인 채 반환.

### 표 4. DBMD — `getBestRowIdentifier()` 의 `DATA_TYPE`

행 식별자(PK/유니크 제약 컬럼)의 타입. (`value[2]`, `switch` line 1513~1619)

| # | CUBRID 타입 | `U_TYPE` (값) | `java.sql.Types` | TYPE_NAME | line | 판정 |
|---|---|---|---|---|---|---|
| 1 | CHAR | CHAR(1) | `CHAR` | CHAR | :1514 | ✅ |
| 2 | VARCHAR | VARCHAR(2) | `VARCHAR` | VARCHAR | :1519 | ✅ |
| 3 | ENUM | ENUM(25) | `VARCHAR` | ENUM | :1524 | ✅ |
| 4 | SHORT / SMALLINT | SHORT(9) | `SMALLINT` | SMALLINT | :1529 | ✅ |
| 5 | INTEGER / INT | INT(8) | `INTEGER` | INTEGER | :1534 | ✅ |
| 6 | BIGINT | BIGINT(21) | `BIGINT` | BIGINT | :1539 | ✅ |
| 7 | DOUBLE | DOUBLE(12) | `DOUBLE` | DOUBLE | :1544 | ✅ |
| 8 | FLOAT / REAL | FLOAT(11) | `REAL` | FLOAT | :1549 | ✅ |
| 9 | NUMERIC | NUMERIC(7) | `NUMERIC` | NUMERIC | :1554 | ✅ |
| 10 | DATE | DATE(13) | `DATE` | DATE | :1559 | ✅ |
| 11 | TIME | TIME(14) | `TIME` | TIME | :1564 | ✅ |
| 12 | TIMESTAMP | TIMESTAMP(15) | `TIMESTAMP` | TIMESTAMP | :1569 | ✅ |
| 13 | DATETIME | DATETIME(22) | `TIMESTAMP` | DATETIME | :1574 | ✅ |
| 14 | (NULL) | NULL(0) | `NULL` | "" | :1579 | RSMD①은 `OTHER` |
| 15 | BLOB | BLOB(23) | `BLOB` | BLOB | :1584 | ✅ |
| 16 | CLOB | CLOB(24) | `CLOB` | CLOB | :1589 | ✅ |
| 17 | TIMESTAMPTZ | TIMESTAMPTZ(29) | `TIMESTAMP` | TIMESTAMPTZ | :1594 | ⚠️ TZ 손실 |
| 18 | TIMESTAMPLTZ | TIMESTAMPLTZ(30) | `TIMESTAMP` | TIMESTAMPLTZ | :1599 | ⚠️ |
| 19 | DATETIMETZ | DATETIMETZ(31) | `TIMESTAMP` | DATETIMETZ | :1604 | ⚠️ |
| 20 | DATETIMELTZ | DATETIMELTZ(32) | `TIMESTAMP` | DATETIMELTZ | :1609 | ⚠️ |
| 21 | JSON | JSON(34) | `VARCHAR` | JSON | :1614 | ⚠️ |

**switch에 case 없음 → `DATA_TYPE`(value[2]) = null**: `BIT(5)`, `VARBIT(6)`, `OBJECT(19)`, `SET/MULTISET/SEQUENCE(16~18)`, 부호 없는 정수(26~28). 특히 **BIT/BIT VARYING이 빠져 있어**, 비트열 컬럼이 유니크 키에 포함되면 `DATA_TYPE`이 null이 된다. (다른 지점은 모두 BIT를 처리함)

### 표 5. DBMD — `getTypeInfo()` 카탈로그

드라이버가 "지원한다"고 광고하는 타입 목록. per-column 조회가 아니라 정적 카탈로그이며, 위 표들과 여러 곳에서 어긋난다. (병렬 배열 `column1`=TYPE_NAME / `column2`=DATA_TYPE)

> ⚠️ **getTypeInfo는 `sortTuples`를 호출하지 않는다** (다른 메타데이터 메서드는 모두 정렬). 배열 순서 그대로 반환하므로 JDBC 규약의 "DATA_TYPE 순 정렬"을 위반하며, 아래 중복 DATA_TYPE 행에서 잘못된 행이 먼저 매치될 수 있다.

| # | TYPE_NAME | `java.sql.Types` (DATA_TYPE) | 비고 |
|---|---|---|---|
| 1 | BIT | `BIT` | |
| 2 | NUMERIC | `TINYINT` | ❌ CUBRID에 TINYINT 타입 없음. 삭제하거나, 남긴다면 가장 가까운 `SMALLINT`로. NUMERIC은 부적절 |
| 3 | NUMERIC | `BIGINT` | ❌ 레거시 오매핑. 정답 `BIGINT→BIGINT`(11번)가 이미 있음 → **삭제 대상**. 미정렬이라 이 행이 먼저 매치되면 BIGINT 컬럼을 NUMERIC으로 생성할 위험 |
| 4 | BIT VARYING | `LONGVARBINARY` | |
| 5 | BIT VARYING | `VARBINARY` | |
| 6 | BIT | `BINARY` | |
| 7 | VARCHAR | `LONGVARCHAR` | |
| 8 | CHAR | `CHAR` | |
| 9 | NUMERIC | `NUMERIC` | |
| 10 | INTEGER | `INTEGER` | |
| 11 | BIGINT | `BIGINT` | |
| 12 | SMALLINT | `SMALLINT` | |
| 13 | DOUBLE | `FLOAT` | ✅ 정상. `Types.FLOAT`은 JDBC상 배정밀도(=`Types.DOUBLE`)이므로 CUBRID `DOUBLE`로 매핑하는 것이 옳음. 15번과 이름은 같지만 DATA_TYPE이 달라 충돌 아님 |
| 14 | FLOAT | `REAL` | |
| 15 | DOUBLE | `DOUBLE` | |
| 16 | VARCHAR | `VARCHAR` | |
| 17 | STRING | `VARCHAR` | |
| 18 | DATE | `DATE` | |
| 19 | TIME | `TIME` | |
| 20 | TIMESTAMP | `TIMESTAMP` | |
| 21 | TIMESTAMPTZ | `TIMESTAMP_WITH_TIMEZONE` | ⚠️ 다른 지점은 `TIMESTAMP` |
| 22 | TIMESTAMPLTZ | `TIMESTAMP_WITH_TIMEZONE` | ⚠️ |
| 23 | DATETIME | `TIMESTAMP` | |
| 24 | DATETIMETZ | `TIMESTAMP_WITH_TIMEZONE` | ⚠️ |
| 25 | DATETIMELTZ | `TIMESTAMP_WITH_TIMEZONE` | ⚠️ |
| 26 | BLOB | `BLOB` | |
| 27 | CLOB | `CLOB` | |
| 28 | ENUM | `VARCHAR` | |
| 29 | MULTISET | `ARRAY` | ⚠️ 다른 지점은 `OTHER` |
| 30 | SET | `ARRAY` | ⚠️ |
| 31 | LIST | `ARRAY` | ⚠️ |
| 32 | SEQUENCE | `ARRAY` | ⚠️ |
| 33 | JSON | `VARCHAR` | |

### 관찰 — 지점 간 불일치

같은 CUBRID 타입이 지점마다 다른 `java.sql.Types`로 보고되는 곳:

- **NULL 처리 4갈래** — RSMD① `OTHER` / RSMD② `NULL` / getColumns 처리 없음 / getBestRowIdentifier `NULL`. RSMD①만 `OTHER`(그나마 `Types.NULL`이 주석 처리됨)로 이례적.
- **BIT 처리 5갈래** — RSMD① `BIT`(prec==8)·`BINARY`+클래스 `Boolean`/`byte[]` · RSMD② 항상 `BIT`+`byte[]`(prec 1) · getColumns 항상 `BINARY` · getBestRowIdentifier **case 없음→null** · getTypeInfo `BIT`·`BINARY` 두 행. 한 타입이 지점마다 전부 다르게 나온다.
- **TZ/LTZ 4종** — RSMD①·RSMD(해당없음)·getColumns·getBestRowIdentifier는 전부 `TIMESTAMP`, **getTypeInfo만** `TIMESTAMP_WITH_TIMEZONE`(JDBC 4.2, 값 2014). 타임존 정보 손실 + 내부 불일치. 가장 실질적인 오매핑 후보.
- **컬렉션 SET/MULTISET/SEQUENCE** — RSMD①·getColumns는 `OTHER`, getTypeInfo만 `ARRAY`. (getBestRowIdentifier는 case 없음)
- **부호 없는 정수 USHORT/UINT/UBIGINT** — 어느 지점에도 case가 없다. RSMD①/getColumns는 `Types.NULL(0)`/미설정. 반면 클래스명(`UColumnInfo.findFQDN`)만 Short/Integer/Long로 정상 → 비대칭. 서버가 이 타입을 컬럼 타입으로 방출하는지 도달성 확인 필요.
- **getBestRowIdentifier의 BIT/VARBIT 누락** — 유니크 키에 비트열이 들어가면 `DATA_TYPE`이 null.

### 관찰 — 정상인데 오해하기 쉬운 것

- **FLOAT → `REAL`은 정상.** JDBC 규약상 `REAL`=단정밀도(→Java `float`), `FLOAT`=배정밀도(→`double`). CUBRID FLOAT/REAL은 4바이트 단정밀도이므로 `Types.REAL`이 맞다. `Types.FLOAT`으로 바꾸면 오히려 틀린다.
- **`Types.FLOAT` ≡ `Types.DOUBLE`(둘 다 배정밀도).** getTypeInfo의 "DOUBLE→`Types.FLOAT`"(표 5의 13번)은 오류가 아니라 **정상**이다 — JDBC의 두 배정밀도 상수(FLOAT·DOUBLE)를 CUBRID `DOUBLE` 하나로 커버하는 것. `Types.REAL`(단정밀도)만 CUBRID `FLOAT`에 대응.
- **ENUM / JSON → `VARCHAR`** — 표준 매핑이 없는 타입이라 방어 가능한 선택.
- **역방향(`setObject`)에는 `Types → U_TYPE` 스위치가 없다.** 값의 런타임 Java 클래스로 `U_TYPE`을 정한다(`UUType.getObjectDBtype`). `targetSqlType`은 NUMERIC/DECIMAL 스케일 조정 외엔 무시.

## 결론

전수 대조 결과, **개별 스칼라 타입의 매핑 값 자체는 대체로 규약상 적절**하다(오해하기 쉬운 FLOAT→REAL도 정상). 진짜 문제는 **매핑이 최소 5개 지점에 흩어져 있고, 같은 타입이 지점마다 다르게 보고된다**는 점이다. 검토가 필요한 지점은 다음과 같다.

1. **TZ/LTZ 날짜·시간** — 대부분 `TIMESTAMP`, getTypeInfo만 `TIMESTAMP_WITH_TIMEZONE`.
2. **BIT** — 5개 지점이 전부 다르게 처리(BIT/BINARY/Boolean/byte[]/null). 특히 getBestRowIdentifier는 case 자체가 없음.
3. **NULL** — RSMD①만 `OTHER`, 나머지는 `NULL` 또는 미처리.
4. **부호 없는 정수** — 어느 지점에도 case 없음(도달성 확인 필요).
5. **getTypeInfo 카탈로그** — 미정렬(`sortTuples` 없음 → JDBC "DATA_TYPE 순" 위반) + 오매핑 행: `NUMERIC→TINYINT`(CUBRID에 없는 타입), `NUMERIC→BIGINT`(레거시 중복, 삭제 대상; 미정렬로 BIGINT를 NUMERIC으로 생성할 위험). `DOUBLE→FLOAT`은 정상.

`NCHAR`/`NCHAR VARYING`·`MONETARY` 매핑은 데드 경로(엔진에서 제거된 타입; 단 MONETARY는 11.4 매뉴얼 문구가 아직 "제거될 예정")이며, TIMETZ는 CUBRID 타입이 아니어서 드라이버도 `/* unused */`로 일관된다.

## 참고

- [CUBRID 11.4 매뉴얼 – 데이터 타입](https://www.cubrid.org/manual/ko/11.4/sql/datatype.html)
- CUBRID JDBC 드라이버 소스: `UUType.java`(내부 타입 상수), `CUBRIDResultSetMetaData.java`(RSMD 생성자 ①·②), `CUBRIDDatabaseMetaData.java`(getColumns / getBestRowIdentifier / getTypeInfo), `UColumnInfo.java`(클래스명 매핑)

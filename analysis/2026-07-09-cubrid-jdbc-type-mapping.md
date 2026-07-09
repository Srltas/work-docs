# CUBRID 11.4 데이터 타입 → `java.sql.Types` 매핑 전수 분석

- 분류: analysis
- 날짜: 2026-07-09
- 관련: [CUBRID 11.4 매뉴얼 – 데이터 타입](https://www.cubrid.org/manual/ko/11.4/sql/datatype.html), CUBRID JDBC 드라이버 소스

## 목적

CUBRID JDBC 드라이버가 CUBRID의 각 데이터 타입을 `java.sql.Types`로 올바르게 매핑하는지 전수 조사한다. (1) CUBRID 11.4가 제공하는 데이터 타입 목록을 매뉴얼에서 확보하고, (2) 드라이버가 실제로 반환하는 `java.sql.Types` 값을 소스에서 확인하여 대조한다.

## 배경

JDBC 메타데이터 검증 과정에서 타입 매핑이 부정확해 보인다는 의심이 있었다. 드라이버는 컬럼 타입을 **세 경로**에서 각각 결정한다.

- `ResultSetMetaData.getColumnType()` — 결과셋 컬럼 타입 (앱이 가장 흔히 보는 값)
- `DatabaseMetaData.getColumns()` 의 `DATA_TYPE` — 카탈로그의 컬럼 목록
- `DatabaseMetaData.getTypeInfo()` — 드라이버가 광고하는 지원 타입 카탈로그

세 경로가 서로 일치하는지가 핵심 점검 대상이었다.

## 범위 / 방법

- **CUBRID 타입 정의**: 11.4 매뉴얼 데이터 타입 페이지에서 숫자/문자/비트/날짜·시간/컬렉션/LOB/ENUM/JSON/OID 전수 확보. deprecation·삭제 상태도 원문으로 재확인.
- **드라이버 매핑**: 소스 직접 확인. 아래 line 번호는 확인 시점 기준(버전에 따라 이동 가능).
  - 내부 타입 상수: `cubrid/jdbc/jci/UUType.java` (`U_TYPE_*`, 0~34)
  - RSMD: `cubrid/jdbc/driver/CUBRIDResultSetMetaData.java` 생성자 `switch`
  - DBMD getColumns: `cubrid/jdbc/driver/CUBRIDDatabaseMetaData.java` if/else 체인
  - DBMD getTypeInfo: 같은 파일의 병렬 배열 카탈로그
  - 클래스명: `cubrid/jdbc/jci/UColumnInfo.java` `findFQDN`
- **제외(엔진에서 제거된 타입)**: `NCHAR`, `NCHAR VARYING`, `MONETARY`.
  - `NCHAR` / `NCHAR VARYING` — 매뉴얼상 과거형("9.0 버전부터 더 이상 지원하지 않으며, 대신 CHAR, VARCHAR 타입을 사용").
  - `MONETARY` — 엔진에서 제거됨. 단 11.4 매뉴얼 문구는 미래형("MONETARY 타입은 제거될 예정이며(deprecated)")으로 남아 있어 표기가 다름.
  - 세 타입 모두 드라이버 코드에는 매핑이 잔존(예: `U_TYPE_MONETARY(10)→DOUBLE`)하나 서버가 방출하지 않는 데드 경로.
- **판정 범례**: ✅ 적절 · ⚠️ 검토 필요 · 그 외 비고 참조.

## 발견 / 관찰

### 표 A. RSMD 기준 — `ResultSetMetaData.getColumnType()`

앱이 결과셋 컬럼 타입을 조회할 때 실제로 받는 값. (`CUBRIDResultSetMetaData.java`)

| # | CUBRID 타입 (별칭) | `U_TYPE` (값) | `java.sql.Types` | line | 판정 |
|---|---|---|---|---|---|
| 1 | SHORT / SMALLINT | SHORT(9) | `SMALLINT` | :151 | ✅ |
| 2 | INTEGER / INT | INT(8) | `INTEGER` | :160 | ✅ |
| 3 | BIGINT | BIGINT(21) | `BIGINT` | :171 | ✅ |
| 4 | NUMERIC / DECIMAL / DEC | NUMERIC(7) | `NUMERIC` *(mysql 모드 `DECIMAL`)* | :189 | ✅ |
| 5 | FLOAT / REAL | FLOAT(11) | `REAL` | :177 | ✅ *(규약상 정상, 비고)* |
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

`switch`에 case 없음 → `default`(line 447) → `Types.NULL(0)` 반환: 부호 없는 정수 `USHORT(26)`/`UINT(27)`/`UBIGINT(28)`, `RESULTSET(20)`.

### 표 B. DBMD 기준 ① — `DatabaseMetaData.getColumns()` 의 `DATA_TYPE`

카탈로그로 컬럼 목록을 조회할 때의 타입. (`CUBRIDDatabaseMetaData.java`, `value[4]`)

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
| 9 | BIT | BIT(5) | **항상 `BINARY`** (정밀도 무관) | :1160 | ⚠️ **RSMD와 불일치** |
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
| 27 | (NULL 타입 컬럼) | NULL(0) | **처리 없음** (`DATA_TYPE` 미설정) | — | ⚠️ **RSMD와 불일치** |

if/else 체인에 항목 없음: `NULL(0)`, 부호 없는 정수 `USHORT`/`UINT`/`UBIGINT`, `RESULTSET(20)`. 미매칭 시 `value[4]`가 null인 채 반환.

### 표 C. DBMD 기준 ② — `DatabaseMetaData.getTypeInfo()` 카탈로그

드라이버가 "지원한다"고 광고하는 타입 목록. per-column 조회가 아니라 정적 카탈로그이며, 위 두 표와 여러 곳에서 어긋난다. (`CUBRIDDatabaseMetaData.java`, 병렬 배열 `column1`=TYPE_NAME / `column2`=DATA_TYPE)

| # | TYPE_NAME | `java.sql.Types` (DATA_TYPE) | 비고 |
|---|---|---|---|
| 1 | BIT | `BIT` | |
| 2 | NUMERIC | `TINYINT` | ⚠️ CUBRID에 TINYINT 없음 — NUMERIC 이름을 정수 타입에 부여한 비표준 행 |
| 3 | NUMERIC | `BIGINT` | 11번(BIGINT→BIGINT)과 중복 — NUMERIC 이름으로도 BIGINT를 광고 |
| 4 | BIT VARYING | `LONGVARBINARY` | |
| 5 | BIT VARYING | `VARBINARY` | |
| 6 | BIT | `BINARY` | |
| 7 | VARCHAR | `LONGVARCHAR` | |
| 8 | CHAR | `CHAR` | |
| 9 | NUMERIC | `NUMERIC` | |
| 10 | INTEGER | `INTEGER` | |
| 11 | BIGINT | `BIGINT` | |
| 12 | SMALLINT | `SMALLINT` | |
| 13 | DOUBLE | `FLOAT` | 15번(DOUBLE→DOUBLE)과 중복 행. `Types.FLOAT`은 JDBC상 배정밀도라 의미는 무방 |
| 14 | FLOAT | `REAL` | |
| 15 | DOUBLE | `DOUBLE` | |
| 16 | VARCHAR | `VARCHAR` | |
| 17 | STRING | `VARCHAR` | |
| 18 | DATE | `DATE` | |
| 19 | TIME | `TIME` | |
| 20 | TIMESTAMP | `TIMESTAMP` | |
| 21 | TIMESTAMPTZ | `TIMESTAMP_WITH_TIMEZONE` | ⚠️ 표 A·B는 `TIMESTAMP` |
| 22 | TIMESTAMPLTZ | `TIMESTAMP_WITH_TIMEZONE` | ⚠️ |
| 23 | DATETIME | `TIMESTAMP` | |
| 24 | DATETIMETZ | `TIMESTAMP_WITH_TIMEZONE` | ⚠️ |
| 25 | DATETIMELTZ | `TIMESTAMP_WITH_TIMEZONE` | ⚠️ |
| 26 | BLOB | `BLOB` | |
| 27 | CLOB | `CLOB` | |
| 28 | ENUM | `VARCHAR` | |
| 29 | MULTISET | `ARRAY` | ⚠️ 표 A·B는 `OTHER` |
| 30 | SET | `ARRAY` | ⚠️ |
| 31 | LIST | `ARRAY` | ⚠️ |
| 32 | SEQUENCE | `ARRAY` | ⚠️ |
| 33 | JSON | `VARCHAR` | |

### 표 A(RSMD) vs 표 B(getColumns) 차이 요약

동일 컬럼을 두 API로 조회했을 때 값이 달라지는 지점.

| CUBRID 타입 | RSMD `getColumnType` | DBMD `getColumns` | 비고 |
|---|---|---|---|
| **BIT** | `BIT`(prec==8) / `BINARY` | 항상 `BINARY` | RSMD만 정밀도 8을 `BIT`으로 |
| **NUMERIC** | mysql 모드에서 `DECIMAL` | 항상 `NUMERIC` | getColumns엔 mysql 분기 없음 |
| **(NULL 타입)** | `OTHER` | 처리 없음(미설정) | getColumns는 항목 자체가 없음 |

그 외 타입은 두 API가 동일한 `java.sql.Types`를 반환한다. TZ 계열·컬렉션은 A·B가 서로 일치하지만 **표 C(getTypeInfo)와는 불일치**한다.

### 개별 이슈

1. **TZ/LTZ 날짜·시간 4종(TIMESTAMPTZ·TIMESTAMPLTZ·DATETIMETZ·DATETIMELTZ)** — 표 A·B는 전부 `TIMESTAMP`, 표 C는 `TIMESTAMP_WITH_TIMEZONE`(JDBC 4.2, 값 2014). 타임존 정보 손실 + 내부 불일치. 가장 실질적인 오매핑 후보.
2. **BIT(8)** — RSMD가 `Types.BIT` + 클래스 `java.lang.Boolean`(`UColumnInfo.findFQDN`)로 보고. 1바이트 비트열이 Boolean으로 표현됨. 게다가 getColumns는 같은 컬럼을 `BINARY`로 봐 두 API가 갈린다.
3. **부호 없는 정수(USHORT/UINT/UBIGINT)** — 표 A·B 어디에도 case가 없어 각각 `Types.NULL(0)`(A) 또는 미설정(B). 반면 클래스명은 Short/Integer/Long로 정상 → 비대칭. 서버가 이 타입을 컬럼 타입으로 실제 방출하는지 도달성 확인 필요.
4. **getTypeInfo 카탈로그 이상 행** — TYPE_NAME "NUMERIC"이 `TINYINT`(CUBRID에 없는 타입)에 매핑되고, `BIGINT`·`DOUBLE`은 서로 다른 행에 중복 등장하는 등 손으로 유지한 카탈로그가 정돈되어 있지 않음.

### 정상인데 오해하기 쉬운 것

- **FLOAT → `REAL`은 정상.** JDBC 규약상 `REAL`=단정밀도(→Java `float`), `FLOAT`=배정밀도(→`double`). CUBRID FLOAT/REAL은 4바이트 단정밀도이므로 `Types.REAL`이 맞다. `Types.FLOAT`으로 바꾸면 오히려 틀린다.
- **`Types.FLOAT` ≡ `Types.DOUBLE`(둘 다 배정밀도).** 그래서 getTypeInfo의 "DOUBLE→`Types.FLOAT`" 행은 이름-타입 불일치가 아니라 단순 중복이다(위 표 C 13번).
- **ENUM / JSON → `VARCHAR`** — 표준 매핑이 없는 타입이라 방어 가능한 선택.
- **역방향(`setObject`)에는 `Types → U_TYPE` 스위치가 없다.** 값의 런타임 Java 클래스로 `U_TYPE`을 정한다(`UUType.getObjectDBtype`). `targetSqlType`은 NUMERIC/DECIMAL 스케일 조정 외엔 무시.

## 결론

전수 대조 결과, **대부분의 스칼라 타입 매핑은 규약상 적절**하다(오해하기 쉬운 FLOAT→REAL도 정상). 세 경로의 per-column 차이는 사실상 BIT 한 곳이고, TZ·컬렉션은 **getTypeInfo만** 이탈한다. 실제로 검토가 필요한 지점은 다음 4가지로 좁혀진다.

1. TZ/LTZ 날짜·시간의 `TIMESTAMP` 매핑 및 세 경로 불일치
2. BIT(8)의 Boolean 취급 및 RSMD↔getColumns 불일치
3. 부호 없는 정수의 `Types.NULL` 갭 (도달성 확인 필요)
4. getTypeInfo 카탈로그의 이상·중복 행

`NCHAR`/`NCHAR VARYING`·`MONETARY` 매핑은 데드 경로(엔진에서 제거된 타입; 단 MONETARY는 11.4 매뉴얼 문구가 아직 "제거될 예정")이며, TIMETZ는 CUBRID 타입이 아니어서 드라이버도 `/* unused */`로 일관된다.

## 참고

- [CUBRID 11.4 매뉴얼 – 데이터 타입](https://www.cubrid.org/manual/ko/11.4/sql/datatype.html)
- CUBRID JDBC 드라이버 소스: `UUType.java`(내부 타입 상수), `CUBRIDResultSetMetaData.java`(RSMD switch), `CUBRIDDatabaseMetaData.java`(getColumns / getTypeInfo), `UColumnInfo.java`(클래스명 매핑)

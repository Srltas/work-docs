# CUBRID JDBC 타입 보고 5개 지점의 데이터 타입별 반환값 전수표

- 분류: analysis
- 날짜: 2026-07-20
- 관련: [CUBRID 11.4 매뉴얼, 데이터 타입](https://www.cubrid.org/manual/ko/11.4/sql/datatype.html), CUBRID JDBC 드라이버 소스([CUBRID/cubrid-jdbc](https://github.com/CUBRID/cubrid-jdbc))

## 요약

CUBRID JDBC 드라이버가 컬럼 타입을 보고하는 5개 지점(ResultSetMetaData 생성자 2개, getColumns, getBestRowIdentifier, getTypeInfo)에 대해, CUBRID 11.4가 제공하는 전 데이터 타입의 반환값(java.sql.Types, 타입 이름, 클래스명, 크기류 컬럼)을 소스 기준 표로 정리했다.

## 목적

드라이버의 U_TYPE → JDBC 매핑은 한 곳이 아니라 최소 5개 지점에 각각 하드코딩되어 있다. 어느 API로 조회하느냐에 따라 같은 CUBRID 타입이 다른 값으로 보고될 수 있으므로, 지점별로 "모든 데이터 타입이 실제로 어떤 값을 반환하는가"를 한 문서에서 찾을 수 있는 전수 참조표를 만든다.

## 배경

드라이버에서 타입이 보고되는 경로는 두 갈래다. 일반 SQL 쿼리 결과셋은 서버가 준 `UColumnInfo[]`로 RSMD ①을 만들고, `DatabaseMetaData` 계열 메서드는 드라이버가 서버 쿼리 없이 조립한 합성 결과셋(`CUBRIDResultSetWithoutQuery`)을 돌려주며 그 결과셋의 메타데이터는 RSMD ②가 담당한다. 여기에 `getColumns`, `getBestRowIdentifier`가 튜플에 직접 써넣는 DATA_TYPE/TYPE_NAME 분기, `getTypeInfo`의 병렬 배열 카탈로그가 더해져 총 5개 지점이 된다.

```mermaid
flowchart TB
    APP[애플리케이션] --> Q[일반 SQL 쿼리 ResultSet]
    APP --> D[DatabaseMetaData 호출]
    Q --> R1["① RSMD(UColumnInfo[]) 생성자: switch"]
    D --> GC["③ getColumns: if-else 체인"]
    D --> GB["④ getBestRowIdentifier: switch"]
    D --> GT["⑤ getTypeInfo: 병렬 배열 카탈로그"]
    GC & GB & GT --> W[CUBRIDResultSetWithoutQuery 합성 결과셋]
    W --> R2["② RSMD(WithoutQuery) 생성자: if 체인"]
```

## 범위 / 방법

- **대상 소스**: `CUBRID/cubrid-jdbc` 저장소, `v11.3.2.0053` 기반.
  - RSMD ①: `CUBRIDResultSetMetaData.java`의 생성자 `CUBRIDResultSetMetaData(UColumnInfo[])` (타입 `switch`)
  - RSMD ②: 같은 파일의 생성자 `CUBRIDResultSetMetaData(CUBRIDResultSetWithoutQuery)` (`if` 체인)
  - getColumns / getBestRowIdentifier / getTypeInfo: `CUBRIDDatabaseMetaData.java`
  - 클래스명 매핑: `UColumnInfo.java`의 `findFQDN`, 내부 타입 상수: `UUType.java`(`U_TYPE_*`)
- **타입 범위**: CUBRID 11.4가 제공하는 데이터 타입 전부(수치/문자/비트/날짜·시간/컬렉션/LOB/ENUM/JSON/OID)와, 실행 결과셋에서 실제 발생하는 NULL 타입 컬럼(예: `SELECT NULL`). 다음은 11.4 기준 제공되지 않으므로 표에서 제외했다(드라이버 코드에는 매핑이 잔존).
  - `NCHAR` / `NCHAR VARYING`: 9.0부터 미지원(매뉴얼: "대신 CHAR, VARCHAR 타입을 사용").
  - `MONETARY`: deprecated(매뉴얼: "제거될 예정이며 더 이상 사용을 권장하지 않는다").
  - `USHORT`/`UINT`/`UBIGINT`, `RESULTSET`, `TIMETZ`: SQL 데이터 타입이 아닌 프로토콜 내부·미사용 상수.
- **표기 규칙**:
  - `java.sql.Types` 상수는 `Types.` 접두어 생략. 문자열 반환값은 `"따옴표"`.
  - `미분기` = 그 지점에 해당 타입 분기가 없음. RSMD 두 생성자는 배열 기본값이 남아 `getColumnType()=0`(상수값이 `Types.NULL`과 동일), `getColumnTypeName()=null`이 되고, getColumns/getBestRowIdentifier는 재사용하는 `value[]` 배열의 해당 칸을 덮어쓰지 않아 **이전 행 값이 잔존**한다(첫 행이면 null).
  - getTypeInfo 열의 `행 없음` = 카탈로그 33행에 그 타입 행이 아예 없음.

## 발견 / 관찰

### 표 1. RSMD ① `CUBRIDResultSetMetaData(UColumnInfo[] col_info)`

일반 SQL 쿼리 결과셋의 `ResultSetMetaData`가 반환하는 값.

**표시 크기**는 `getColumnDisplaySize()`의 반환값으로, 컬럼 값을 문자열로 출력할 때 필요한 최대 문자 수(폭)다. 예를 들어 INT는 11(`-2147483648`이 11자), BIGINT는 20, DATE는 10(`YYYY-MM-DD`), TIMESTAMPTZ는 82(TIMESTAMP 19자 + 타임존 문자열 여유 63자)다. 타입별 기본값이 드라이버에 고정되어 있고, 문자(CHAR/VARCHAR/ENUM/JSON)·비트(BIT/BIT VARYING) 계열은 컬럼 정밀도가 기본값보다 크면 정밀도로 올라간다(표의 `1*`). -1은 폭을 정할 수 없는 타입(컬렉션, LOB)이다.

| # | CUBRID 타입 (별칭) | U_TYPE (값) | getColumnType | getColumnTypeName | getColumnClassName | 표시 크기 |
|---|---|---|---|---|---|---|
| 1 | SHORT / SMALLINT | SHORT(9) | `SMALLINT` | "SMALLINT" | java.lang.Short | 6 |
| 2 | INTEGER / INT | INT(8) | `INTEGER` | "INTEGER" | java.lang.Integer | 11 |
| 3 | BIGINT | BIGINT(21) | `BIGINT` | "BIGINT" | java.lang.Long | 20 |
| 4 | NUMERIC / DECIMAL / DEC | NUMERIC(7) | `NUMERIC` | "NUMERIC" | java.math.BigDecimal | 40 |
| 5 | FLOAT / REAL | FLOAT(11) | `REAL` | "FLOAT" | java.lang.Float | 13 |
| 6 | DOUBLE / DOUBLE PRECISION | DOUBLE(12) | `DOUBLE` | "DOUBLE" | java.lang.Double | 23 |
| 7 | CHAR / CHARACTER | CHAR(1) | `CHAR` | "CHAR" | java.lang.String | 1* |
| 8 | VARCHAR / CHAR VARYING / STRING | VARCHAR(2) | `VARCHAR` | "VARCHAR" | java.lang.String | 1* |
| 9 | ENUM | ENUM(25) | `VARCHAR` | "ENUM" | java.lang.String | 1* |
| 10 | BIT (정밀도 8) | BIT(5) | `BIT` | "BIT" | java.lang.Boolean | 1* |
| 11 | BIT (그 외 정밀도) | BIT(5) | `BINARY` | "BIT" | byte[] | 1* |
| 12 | BIT VARYING | VARBIT(6) | `VARBINARY` | "BIT VARYING" | byte[] | 1* |
| 13 | DATE | DATE(13) | `DATE` | "DATE" | java.sql.Date | 10 |
| 14 | TIME | TIME(14) | `TIME` | "TIME" | java.sql.Time | 8 |
| 15 | TIMESTAMP | TIMESTAMP(15) | `TIMESTAMP` | "TIMESTAMP" | java.sql.Timestamp | 19 |
| 16 | TIMESTAMPTZ | TIMESTAMPTZ(29) | `TIMESTAMP` | "TIMESTAMPTZ" | java.sql.Timestamp | 82 |
| 17 | TIMESTAMPLTZ | TIMESTAMPLTZ(30) | `TIMESTAMP` | "TIMESTAMPLTZ" | java.sql.Timestamp | 82 |
| 18 | DATETIME | DATETIME(22) | `TIMESTAMP` | "DATETIME" | java.sql.Timestamp | 23 |
| 19 | DATETIMETZ | DATETIMETZ(31) | `TIMESTAMP` | "DATETIMETZ" | java.sql.Timestamp | 86 |
| 20 | DATETIMELTZ | DATETIMELTZ(32) | `TIMESTAMP` | "DATETIMELTZ" | java.sql.Timestamp | 86 |
| 21 | SET | SET(16) | `OTHER` | "SET" | 요소 배열(표 1-1) | -1 |
| 22 | MULTISET | MULTISET(17) | `OTHER` | "MULTISET" | 요소 배열(표 1-1) | -1 |
| 23 | LIST / SEQUENCE | SEQUENCE(18) | `OTHER` | "SEQUENCE" | 요소 배열(표 1-1) | -1 |
| 24 | BLOB | BLOB(23) | `BLOB` | "BLOB" | java.sql.Blob | -1 |
| 25 | CLOB | CLOB(24) | `CLOB` | "CLOB" | java.sql.Clob | -1 |
| 26 | JSON | JSON(34) | `VARCHAR` | "JSON" | java.lang.String | 1* |
| 27 | object (OID) | OBJECT(19) | `OTHER` | "CLASS" | cubrid.sql.CUBRIDOID | 256 |
| 28 | (NULL 타입 컬럼) | NULL(0) | `OTHER` | "" | "null" | 4 |

- NULL 타입은 소스에 `Types.NULL`이 주석 처리되어 있고 `OTHER`를 반환한다. 미분기의 0과 달리 명시적 `OTHER`다.
- 클래스명(getColumnClassName)은 `UColumnInfo.findFQDN`이 담당한다.
- **MySQL 호환 빌드 분기**(`UJCIUtil.isMysqlMode`, 패키지명 3번째 세그먼트가 `mysql`인 별도 빌드에서만 true): INT의 이름이 "INT", NUMERIC이 `DECIMAL`/"DECIMAL"로 바뀌고, SHORT/INT는 표시 크기가 정밀도로, NUMERIC은 정밀도+1(스케일>0이면 +2)로 바뀐다.
- Types 값에서 파생되는 boolean 메서드(두 생성자 공통): `isCaseSensitive()`는 CHAR/VARCHAR/LONGVARCHAR만 true, `isSigned()`는 SMALLINT/INTEGER/NUMERIC/DECIMAL/REAL/DOUBLE만 true라 **BIGINT 컬럼은 isSigned()=false**, `isCurrency()`는 DOUBLE/REAL/NUMERIC이 true(MySQL 모드는 항상 false).

#### 표 1-1. 컬렉션(SET/MULTISET/SEQUENCE) 요소 타입 반환값

컬렉션 컬럼에서 CUBRID 확장 API `getElementType()`/`getElementTypeName()`(비컬렉션 컬럼이면 예외)과 `getColumnClassName()`이 요소(base) 타입별로 반환하는 값. 11.4 기준 컬렉션 요소는 BLOB/CLOB을 제외한 타입이 허용된다(매뉴얼).

| 요소 U_TYPE (값) | getElementType | getElementTypeName | getColumnClassName |
|---|---|---|---|
| CHAR(1) | `CHAR` | "CHAR" | java.lang.String[] |
| VARCHAR(2) | `VARCHAR` | "VARCHAR" | java.lang.String[] |
| ENUM(25) | `VARCHAR` | "ENUM" | java.lang.String[] |
| BIT(5) (컬럼 정밀도 8) | `BIT` | "BIT" | java.lang.Boolean[] |
| BIT(5) (그 외) | `BINARY` | "BIT" | byte[][] |
| VARBIT(6) | `VARBINARY` | "BIT VARYING" | byte[][] |
| SHORT(9) | `SMALLINT` | "SMALLINT" | java.lang.Short[] |
| INT(8) | `INTEGER` | "INTEGER" | java.lang.Integer[] |
| BIGINT(21) | `BIGINT` | "BIGINT" | java.lang.Long[] |
| FLOAT(11) | `REAL` | "FLOAT" | java.lang.Float[] |
| DOUBLE(12) | `DOUBLE` | "DOUBLE" | java.lang.Double[] |
| NUMERIC(7) | `NUMERIC` | "NUMERIC" | java.lang.Double[] (BigDecimal[] 아님) |
| DATE(13) | `DATE` | "DATE" | java.sql.Date[] |
| TIME(14) | `TIME` | "TIME" | java.sql.Time[] |
| TIMESTAMP(15) | `TIMESTAMP` | "TIMESTAMP" | java.sql.Timestamp[] |
| TIMESTAMPTZ(29) | `TIMESTAMP` | "TIMESTAMPTZ" | java.sql.Timestamp[] |
| TIMESTAMPLTZ(30) | `TIMESTAMP` | "TIMESTAMPLTZ" | java.sql.Timestamp[] |
| DATETIME(22) | `TIMESTAMP` | "DATETIME" | java.sql.Timestamp[] |
| DATETIMETZ(31) | `TIMESTAMP` | "DATETIMETZ" | java.sql.Timestamp[] |
| DATETIMELTZ(32) | `TIMESTAMP` | "DATETIMELTZ" | java.sql.Timestamp[] |
| JSON(34) | `VARCHAR` | "JSON" | java.lang.String[] |
| OBJECT(19) | `OTHER` | "CLASS" | cubrid.sql.CUBRIDOID[] |
| SET(16)/MULTISET(17)/SEQUENCE(18) (중첩) | `OTHER` | "SET"/"MULTISET"/"SEQUENCE" | null |
| NULL(0) (예: `SELECT {NULL}`) | `NULL` | "" | "null" |

- BIT 요소의 Boolean[]/byte[][] 판정 기준은 요소 정밀도가 아니라 **컬럼 정밀도**(`getColumnPrecision()==8`)다.

### 표 2. RSMD ② `CUBRIDResultSetMetaData(CUBRIDResultSetWithoutQuery r)`

`DatabaseMetaData`가 돌려주는 합성 결과셋(getColumns, getTypeInfo, getTables 등 대부분의 DBMD 결과셋) 자체의 `ResultSetMetaData`. `switch`가 아니라 **독립 `if` 7개의 체인**이며, 합성 결과셋 컬럼 선언에 실제로 쓰이는 타입(VARCHAR/INT/SHORT/BIT/NULL 등)만 처리한다.

| # | 처리 U_TYPE (값) | getColumnType | getColumnTypeName | getPrecision | getColumnClassName |
|---|---|---|---|---|---|
| 1 | BIT(5) | `BIT` | "BIT" | 1 (강제) | "byte[]" |
| 2 | INT(8) | `INTEGER` | "INTEGER" | 10 (강제) | "java.lang.Integer" |
| 3 | SHORT(9) | `SMALLINT` | "SMALLINT" | 5 (강제) | "java.lang.Short" |
| 4 | VARCHAR(2) | `VARCHAR` | "VARCHAR" | 선언 정밀도 | "java.lang.String" |
| 5 | ENUM(25) | `VARCHAR` | "ENUM" | 선언 정밀도 | "java.lang.String" |
| 6 | JSON(34) | `VARCHAR` | "JSON" | 선언 정밀도 | "java.lang.String" |
| 7 | NULL(0) | `NULL` | "" | 0 | "" |
| - | 그 외 U_TYPE 전부 | 미분기: 0(=`NULL`) | 미분기: null | 0 | null |

- **생성자 ①과 다른 점**: BIT가 정밀도와 무관하게 항상 `BIT`이고 클래스명도 `byte[]`(①은 정밀도 8일 때만 `BIT`+`Boolean`). NULL 타입은 ①의 `OTHER`와 달리 명시적 `NULL`.
- 행 공통 고정값: `getScale()=0`, `getSchemaName()`/`getTableName()`="", `getColumnCharset()`=null, `isAutoIncrement()`=false. 표시 크기는 표 1과 같은 기본값 함수를 쓰며 VARCHAR/ENUM/JSON은 선언 정밀도가 크면 정밀도로 상승.
- 합성 결과셋의 컬럼 선언은 7종 안에 들도록 설계되어 있으나(예: getColumns의 BUFFER_LENGTH는 `U_TYPE_NULL`로 선언), 선언이 이를 벗어나면 조용히 0/null이 된다.

### 표 3. DBMD `getColumns()` 의 DATA_TYPE / TYPE_NAME

카탈로그 컬럼 조회 시 튜플의 `DATA_TYPE`(value[4])과 `TYPE_NAME`(value[5]).

| # | CUBRID 타입 (별칭) | U_TYPE (값) | DATA_TYPE | TYPE_NAME |
|---|---|---|---|---|
| 1 | SHORT / SMALLINT | SHORT(9) | `SMALLINT` | "SMALLINT" |
| 2 | INTEGER / INT | INT(8) | `INTEGER` | "INTEGER" |
| 3 | BIGINT | BIGINT(21) | `BIGINT` | "BIGINT" |
| 4 | NUMERIC / DECIMAL / DEC | NUMERIC(7) | `NUMERIC` | "NUMERIC" |
| 5 | FLOAT / REAL | FLOAT(11) | `REAL` | "FLOAT" |
| 6 | DOUBLE / DOUBLE PRECISION | DOUBLE(12) | `DOUBLE` | "DOUBLE PRECISION" |
| 7 | CHAR / CHARACTER | CHAR(1) | `CHAR` | "CHAR" |
| 8 | VARCHAR / CHAR VARYING / STRING | VARCHAR(2) | `VARCHAR` | "VARCHAR" |
| 9 | ENUM | ENUM(25) | `VARCHAR` | "ENUM" |
| 10 | BIT (정밀도 무관) | BIT(5) | `BINARY` | "BIT" |
| 11 | BIT VARYING | VARBIT(6) | `VARBINARY` | "BIT VARYING" |
| 12 | DATE | DATE(13) | `DATE` | "DATE" |
| 13 | TIME | TIME(14) | `TIME` | "TIME" |
| 14 | TIMESTAMP | TIMESTAMP(15) | `TIMESTAMP` | "TIMESTAMP" |
| 15 | TIMESTAMPTZ | TIMESTAMPTZ(29) | `TIMESTAMP` | "TIMESTAMPTZ" |
| 16 | TIMESTAMPLTZ | TIMESTAMPLTZ(30) | `TIMESTAMP` | "TIMESTAMPLTZ" |
| 17 | DATETIME | DATETIME(22) | `TIMESTAMP` | "DATETIME" |
| 18 | DATETIMETZ | DATETIMETZ(31) | `TIMESTAMP` | "DATETIMETZ" |
| 19 | DATETIMELTZ | DATETIMELTZ(32) | `TIMESTAMP` | "DATETIMELTZ" |
| 20 | SET | SET(16) | `OTHER` | "SET" |
| 21 | MULTISET | MULTISET(17) | `OTHER` | "MULTISET" |
| 22 | LIST / SEQUENCE | SEQUENCE(18) | `OTHER` | "SEQUENCE" |
| 23 | BLOB | BLOB(23) | `BLOB` | "BLOB" |
| 24 | CLOB | CLOB(24) | `CLOB` | "CLOB" |
| 25 | JSON | JSON(34) | `VARCHAR` | "JSON" |
| 26 | object (OID) | OBJECT(19) | `OTHER` | "CLASS" |
| - | 그 외 U_TYPE(NULL 타입 포함) | 미분기: 이전 행 값 잔존(첫 행 null) | 좌동 | 좌동 |

- **RSMD ①과 다른 점**: BIT가 정밀도 무관 항상 `BINARY`(①은 정밀도 8이면 `BIT`), DOUBLE의 TYPE_NAME이 "DOUBLE PRECISION"(①은 "DOUBLE"), NUMERIC에 MySQL 모드 분기 없음, NULL 타입 분기 자체가 없음.
- 타입 무관 공통 컬럼: TABLE_CAT=null, TABLE_SCHEM/TABLE_NAME은 `소유자.클래스` 분리, COLUMN_SIZE=CHAR_OCTET_LENGTH=정밀도, DECIMAL_DIGITS=스케일, NUM_PREC_RADIX=10, BUFFER_LENGTH=SQL_DATA_TYPE=SQL_DATETIME_SUB=null, NULLABLE/IS_NULLABLE는 non-null 플래그, COLUMN_DEF=컬럼 기본값, ORDINAL_POSITION=속성 순서, REMARKS는 브로커가 주는 경우만.
- 반환 전 `sortTuples("getColumns")`로 정렬한다.

### 표 4. DBMD `getBestRowIdentifier()` 의 DATA_TYPE / TYPE_NAME / COLUMN_SIZE

행 식별자 후보 컬럼의 타입 보고. COLUMN_SIZE(value[4])는 수치 6종만 정밀도를 넣고 나머지는 0이다.

| # | CUBRID 타입 | U_TYPE (값) | DATA_TYPE | TYPE_NAME | COLUMN_SIZE |
|---|---|---|---|---|---|
| 1 | SHORT / SMALLINT | SHORT(9) | `SMALLINT` | "SMALLINT" | 정밀도 |
| 2 | INTEGER / INT | INT(8) | `INTEGER` | "INTEGER" | 정밀도 |
| 3 | BIGINT | BIGINT(21) | `BIGINT` | "BIGINT" | 정밀도 |
| 4 | NUMERIC / DECIMAL / DEC | NUMERIC(7) | `NUMERIC` | "NUMERIC" | 정밀도 |
| 5 | FLOAT / REAL | FLOAT(11) | `REAL` | "FLOAT" | 정밀도 |
| 6 | DOUBLE / DOUBLE PRECISION | DOUBLE(12) | `DOUBLE` | "DOUBLE" | 정밀도 |
| 7 | CHAR / CHARACTER | CHAR(1) | `CHAR` | "CHAR" | 0 |
| 8 | VARCHAR / CHAR VARYING / STRING | VARCHAR(2) | `VARCHAR` | "VARCHAR" | 0 |
| 9 | ENUM | ENUM(25) | `VARCHAR` | "ENUM" | 0 |
| 10 | DATE | DATE(13) | `DATE` | "DATE" | 0 |
| 11 | TIME | TIME(14) | `TIME` | "TIME" | 0 |
| 12 | TIMESTAMP | TIMESTAMP(15) | `TIMESTAMP` | "TIMESTAMP" | 0 |
| 13 | TIMESTAMPTZ | TIMESTAMPTZ(29) | `TIMESTAMP` | "TIMESTAMPTZ" | 0 |
| 14 | TIMESTAMPLTZ | TIMESTAMPLTZ(30) | `TIMESTAMP` | "TIMESTAMPLTZ" | 0 |
| 15 | DATETIME | DATETIME(22) | `TIMESTAMP` | "DATETIME" | 0 |
| 16 | DATETIMETZ | DATETIMETZ(31) | `TIMESTAMP` | "DATETIMETZ" | 0 |
| 17 | DATETIMELTZ | DATETIMELTZ(32) | `TIMESTAMP` | "DATETIMELTZ" | 0 |
| 18 | BLOB | BLOB(23) | `BLOB` | "BLOB" | 0 |
| 19 | CLOB | CLOB(24) | `CLOB` | "CLOB" | 0 |
| 20 | JSON | JSON(34) | `VARCHAR` | "JSON" | 0 |
| 21 | (NULL 타입) | NULL(0) | `NULL` | "" | 0 |
| - | BIT(5), VARBIT(6), object(19), SET(16)/MULTISET(17)/SEQUENCE(18) | 미분기: 이전 행 값 잔존(첫 행 null) | 좌동 | 좌동 | 좌동 |

- **BIT / BIT VARYING에 case가 없다.** 5개 지점 중 유일하게 비트열을 전혀 처리하지 못하므로, 비트열 컬럼이 UNIQUE 키에 포함되면 DATA_TYPE/TYPE_NAME/COLUMN_SIZE가 미설정(잔존값)으로 나간다.
- 동작 특성: 제약 타입 0(UNIQUE)인 제약만 스캔하고 그중 컬럼 수가 가장 적은 것을 고른다. `scope`/`nullable` 파라미터는 무시된다. **SCOPE 컬럼(value[0])은 어디서도 설정하지 않아 전 행 null**이다. BUFFER_LENGTH=null, DECIMAL_DIGITS=스케일, PSEUDO_COLUMN=`bestRowNotPseudo` 고정. 반환 전 `sortTuples` 정렬.

### 표 5. DBMD `getTypeInfo()` 카탈로그 (33행 전체)

per-컬럼 조회가 아니라 역방향 카탈로그다: DATA_TYPE(`java.sql.Types`)을 키로 "그 JDBC 타입을 만들 때 쓸 CUBRID 네이티브 타입 이름(TYPE_NAME)"을 광고한다. 병렬 배열을 순서 그대로 내보내며, **5개 지점 중 유일하게 `sortTuples`를 호출하지 않는다**(JDBC 규약은 DATA_TYPE 순 + 근접순 정렬 요구).

행별로 값이 달라지는 컬럼 전부를 표기한다. SEARCHABLE의 `basic`=`typePredBasic`(2, WHERE 비교만), `searchable`=`typeSearchable`(3, LIKE 포함 전부).

| # | TYPE_NAME | DATA_TYPE | PRECISION | 리터럴 접두/접미 | CREATE_PARAMS | CASE_SENS | SEARCHABLE | UNSIGNED | FIXED_PREC | MAX_SCALE |
|---|---|---|---|---|---|---|---|---|---|---|
| 1 | BIT | `BIT` | 8 | B' / ' | (8) | false | basic | true | false | 0 |
| 2 | NUMERIC | `TINYINT` | 3 | null / null | (3) | false | basic | false | true | 0 |
| 3 | NUMERIC | `BIGINT` | 38 | null / null | null | false | basic | false | true | 38 |
| 4 | BIT VARYING | `LONGVARBINARY` | 1073741823 | X' / ' | null | false | basic | true | false | 0 |
| 5 | BIT VARYING | `VARBINARY` | 1073741823 | X' / ' | null | false | basic | true | false | 0 |
| 6 | BIT | `BINARY` | 1073741823 | X' / ' | null | false | basic | true | false | 0 |
| 7 | VARCHAR | `LONGVARCHAR` | 1073741823 | ' / ' | null | true | basic | true | false | 0 |
| 8 | CHAR | `CHAR` | 1073741823 | ' / ' | null | true | searchable | true | false | 0 |
| 9 | NUMERIC | `NUMERIC` | 38 | null / null | null | false | searchable | false | true | 38 |
| 10 | INTEGER | `INTEGER` | 10 | null / null | null | false | basic | false | false | 0 |
| 11 | BIGINT | `BIGINT` | 19 | null / null | null | false | basic | false | false | 0 |
| 12 | SMALLINT | `SMALLINT` | 5 | null / null | null | false | basic | false | false | 0 |
| 13 | DOUBLE | `FLOAT` | 38 | null / null | null | false | basic | false | true | 38 |
| 14 | FLOAT | `REAL` | 38 | null / null | null | false | basic | false | true | 38 |
| 15 | DOUBLE | `DOUBLE` | 38 | null / null | null | true | basic | false | true | 38 |
| 16 | VARCHAR | `VARCHAR` | 1073741823 | ' / ' | null | false | searchable | true | false | 0 |
| 17 | STRING | `VARCHAR` | 1073741823 | ' / ' | null | false | searchable | true | false | 0 |
| 18 | DATE | `DATE` | 10 | DATE' / ' | null | false | basic | true | false | 0 |
| 19 | TIME | `TIME` | 11 | TIME' / ' | null | false | basic | true | false | 0 |
| 20 | TIMESTAMP | `TIMESTAMP` | 22 | TIMESTAMP' / ' | null | false | basic | true | false | 0 |
| 21 | TIMESTAMPTZ | `TIMESTAMP_WITH_TIMEZONE` | 22 | TIMESTAMPTZ' / ' | null | false | basic | true | false | 0 |
| 22 | TIMESTAMPLTZ | `TIMESTAMP_WITH_TIMEZONE` | 22 | TIMESTAMPLTZ' / ' | null | false | basic | true | false | 0 |
| 23 | DATETIME | `TIMESTAMP` | 26 | DATETIME' / ' | null | false | basic | true | false | 0 |
| 24 | DATETIMETZ | `TIMESTAMP_WITH_TIMEZONE` | 26 | DATETIMETZ' / ' | null | false | basic | true | false | 0 |
| 25 | DATETIMELTZ | `TIMESTAMP_WITH_TIMEZONE` | 26 | DATETIMELTZ' / ' | null | false | basic | true | false | 0 |
| 26 | BLOB | `BLOB` | 1073741823 | null / null | null | false | basic | true | false | 0 |
| 27 | CLOB | `CLOB` | 1073741823 | null / null | null | false | basic | true | false | 0 |
| 28 | ENUM | `VARCHAR` | 1073741823 | ' / ' | null | true | basic | true | false | 0 |
| 29 | MULTISET | `ARRAY` | 1073741823 | MULTISET / null | null | true | basic | true | false | 0 |
| 30 | SET | `ARRAY` | 1073741823 | SET / null | null | true | basic | true | false | 0 |
| 31 | LIST | `ARRAY` | 1073741823 | LIST / null | null | true | basic | true | false | 0 |
| 32 | SEQUENCE | `ARRAY` | 1073741823 | SEQUENCE / null | null | true | basic | true | false | 0 |
| 33 | JSON | `VARCHAR` | 1073741823 | ' / ' | null | false | basic | true | false | 0 |

- 전 행 고정값 컬럼: NULLABLE=`typeNullable`(1), AUTO_INCREMENT=false, LOCAL_TYPE_NAME=TYPE_NAME과 동일 배열, MINIMUM_SCALE=0, SQL_DATA_TYPE=SQL_DATETIME_SUB=null, NUM_PREC_RADIX=10.
- OID(object) 타입의 행은 없다. 반대로 CUBRID에 없는 `TINYINT`용 NUMERIC(3) 행, 네이티브 BIGINT 이전의 레거시로 보이는 NUMERIC(38)→`BIGINT` 행이 있다.
- **값 배열이 한 칸 밀린 정황**: CASE_SENSITIVE는 문자 계열인 16·17행(VARCHAR/STRING)이 false인데 수치인 15행(DOUBLE)이 true이고, SEARCHABLE은 7행(LONGVARCHAR)이 basic인데 9행(NUMERIC)이 searchable이다. 문자 계열에 줄 값이 인접 행으로 밀려 들어간 모양새다.
- TZ 4종을 `TIMESTAMP_WITH_TIMEZONE`(JDBC 4.2)으로 광고하는 유일한 지점이다(표 1~4는 전부 `TIMESTAMP`).

### 표 6. 지점 간 한눈 대조 (CUBRID 타입 × 5지점의 java.sql.Types)

| CUBRID 타입 | ① RSMD 쿼리 | ② RSMD 합성 | ③ getColumns | ④ getBestRowId | ⑤ getTypeInfo |
|---|---|---|---|---|---|
| SHORT | `SMALLINT` | `SMALLINT` | `SMALLINT` | `SMALLINT` | `SMALLINT` |
| INTEGER | `INTEGER` | `INTEGER` | `INTEGER` | `INTEGER` | `INTEGER` |
| BIGINT | `BIGINT` | 미분기 | `BIGINT` | `BIGINT` | `BIGINT` (+NUMERIC 행도 BIGINT 광고) |
| NUMERIC | `NUMERIC` (MySQL 빌드 `DECIMAL`) | 미분기 | `NUMERIC` | `NUMERIC` | `NUMERIC` (+TINYINT/BIGINT 대체 행) |
| FLOAT | `REAL` | 미분기 | `REAL` | `REAL` | `REAL` |
| DOUBLE | `DOUBLE` | 미분기 | `DOUBLE` | `DOUBLE` | `DOUBLE`+`FLOAT` 2행 |
| CHAR | `CHAR` | 미분기 | `CHAR` | `CHAR` | `CHAR` |
| VARCHAR | `VARCHAR` | `VARCHAR` | `VARCHAR` | `VARCHAR` | `VARCHAR`+`LONGVARCHAR` 2행 (별칭 STRING 행도 `VARCHAR`) |
| ENUM | `VARCHAR` | `VARCHAR` | `VARCHAR` | `VARCHAR` | `VARCHAR` |
| BIT(8) | `BIT` | `BIT` | `BINARY` | 미분기 | `BIT` 행 |
| BIT(n≠8) | `BINARY` | `BIT` | `BINARY` | 미분기 | `BINARY` 행 |
| BIT VARYING | `VARBINARY` | 미분기 | `VARBINARY` | 미분기 | `VARBINARY`+`LONGVARBINARY` 2행 |
| DATE / TIME | `DATE` / `TIME` | 미분기 | `DATE` / `TIME` | `DATE` / `TIME` | `DATE` / `TIME` |
| TIMESTAMP / DATETIME | `TIMESTAMP` | 미분기 | `TIMESTAMP` | `TIMESTAMP` | `TIMESTAMP` |
| TZ/LTZ 4종 | `TIMESTAMP` | 미분기 | `TIMESTAMP` | `TIMESTAMP` | **`TIMESTAMP_WITH_TIMEZONE`** |
| SET/MULTISET/SEQUENCE | `OTHER` | 미분기 | `OTHER` | 미분기 | **`ARRAY`** |
| BLOB / CLOB | `BLOB` / `CLOB` | 미분기 | `BLOB` / `CLOB` | `BLOB` / `CLOB` | `BLOB` / `CLOB` |
| JSON | `VARCHAR` | `VARCHAR` | `VARCHAR` | `VARCHAR` | `VARCHAR` |
| object (OID) | `OTHER` | 미분기 | `OTHER` | 미분기 | 행 없음 |
| NULL 타입 | `OTHER` | `NULL` | 미분기 | `NULL` | 행 없음 |

## 결론

- 스칼라 타입의 매핑 값 자체는 5개 지점 모두 대체로 표준적이다(FLOAT→`REAL`은 JDBC 규약상 정답이고, getTypeInfo의 DOUBLE→`FLOAT` 행도 `Types.FLOAT`이 배정밀도라 정상).
- 지점 간 값이 갈리는 타입은 대조표(표 6) 굵은 곳들이다: **BIT**(①`BIT`/`BINARY` 정밀도 분기, ② 항상 `BIT`, ③ 항상 `BINARY`, ④ 미분기), **TZ/LTZ 4종**(⑤만 `TIMESTAMP_WITH_TIMEZONE`), **컬렉션**(⑤만 `ARRAY`), **NULL 타입**(①`OTHER`, ②④`NULL`, ③ 미분기), 그리고 TYPE_NAME 표기(③만 "DOUBLE PRECISION").
- 미분기 시 동작도 지점마다 다르다: RSMD 계열은 0(=`Types.NULL` 상수값)이 조용히 반환되고, getColumns/getBestRowIdentifier는 재사용 배열 탓에 **이전 행 값이 잔존**한다.
- getTypeInfo는 유일하게 미정렬이며, CASE_SENSITIVE/SEARCHABLE 배열이 한 칸 밀린 듯한 행(DOUBLE=true, VARCHAR=false 등)이 있다.

## 다음 단계

- 두 번째 노트로 지점 간 불일치와 JDBC 규약 위반(TZ 매핑, getTypeInfo 미정렬, 컬렉션 `ARRAY` 광고 대 `getArray()` 미구현, getBestRowIdentifier의 PK 미반환)을 분석 관점에서 재정리.
- 세 번째 노트로 Testcontainers 기반 5-DB(CUBRID/Oracle/MySQL/SQL Server/PostgreSQL) 실측 대조를 재정리.
- 매핑 로직 단일화(공유 매핑 함수) 개선은 별도 이슈로 논의.

## 참고

- CUBRID JDBC 드라이버 소스: [CUBRID/cubrid-jdbc](https://github.com/CUBRID/cubrid-jdbc) `v11.3.2.0053` 기반
  - `CUBRIDResultSetMetaData.java` (RSMD 생성자 ①·②, 기본 표시 크기)
  - `CUBRIDDatabaseMetaData.java` (getColumns / getBestRowIdentifier / getTypeInfo)
  - `UColumnInfo.java` (findFQDN 클래스명 매핑), `UUType.java` (U_TYPE 상수)
  - `CUBRIDResultSetWithoutQuery.java` (합성 결과셋, addTuple 복사 동작)
- [CUBRID 11.4 매뉴얼, 데이터 타입](https://www.cubrid.org/manual/ko/11.4/sql/datatype.html)
  - 컬렉션 요소: "BLOB, CLOB 타입을 제외한 나머지 타입들은 컬렉션 타입의 원소가 될 수 있다."
  - "MONETARY 타입은 제거될 예정이며(deprecated), 더 이상 사용을 권장하지 않는다."
  - "NCHAR, NCHAR VARYING 은 9.0 버전부터 더 이상 지원하지 않으며, 대신 CHAR, VARCHAR 타입을 사용하도록 한다."

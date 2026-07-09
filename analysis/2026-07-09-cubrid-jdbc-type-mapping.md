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

- **CUBRID 타입 정의**: 11.4 매뉴얼 데이터 타입 페이지에서 숫자/문자/비트/날짜·시간/컬렉션/LOB/ENUM/JSON/OID 전수 확보. deprecation·삭제 상태(NCHAR, MONETARY, TIMETZ 존재 여부)도 원문으로 재확인.
- **드라이버 매핑**: 소스 직접 확인.
  - 내부 타입 상수: `cubrid/jdbc/jci/UUType.java` (`U_TYPE_*`, 0~34)
  - RSMD: `cubrid/jdbc/driver/CUBRIDResultSetMetaData.java` 생성자 `switch`
  - DBMD getColumns: `cubrid/jdbc/driver/CUBRIDDatabaseMetaData.java` if/else 체인
  - DBMD getTypeInfo: 같은 파일의 병렬 배열 카탈로그
  - 클래스명: `cubrid/jdbc/jci/UColumnInfo.java` `findFQDN`
- **제외**: `NCHAR`, `NCHAR VARYING` — 매뉴얼상 9.0에서 엔진 제거("9.0 버전부터 더 이상 지원하지 않으며, 대신 CHAR, VARCHAR 타입을 사용"). 드라이버 코드에는 매핑이 잔존하나 서버가 방출하지 않는 데드 경로.

## 발견 / 관찰

### 매핑표 (RSMD `getColumnType` 기준, NCHAR 제외)

| CUBRID 타입 (별칭) | `U_TYPE` | `java.sql.Types` | 판정 |
|---|---|---|---|
| SHORT / SMALLINT | SHORT(9) | `SMALLINT` | ✅ |
| INTEGER / INT | INT(8) | `INTEGER` | ✅ |
| BIGINT | BIGINT(21) | `BIGINT` | ✅ |
| NUMERIC / DECIMAL / DEC | NUMERIC(7) | `NUMERIC` *(mysql 모드 `DECIMAL`)* | ✅ |
| FLOAT / REAL | FLOAT(11) | `REAL` | ✅ *(규약상 정상)* |
| DOUBLE / DOUBLE PRECISION | DOUBLE(12) | `DOUBLE` | ✅ |
| MONETARY | MONETARY(10) | `DOUBLE` | ⚠️ deprecated |
| CHAR / CHARACTER | CHAR(1) | `CHAR` | ✅ |
| VARCHAR / CHAR VARYING / STRING | VARCHAR(2) | `VARCHAR` | ✅ |
| BIT | BIT(5) | prec==8 → `BIT`, 그 외 → `BINARY` | ⚠️ |
| BIT VARYING | VARBIT(6) | `VARBINARY` | ✅ |
| DATE | DATE(13) | `DATE` | ✅ |
| TIME | TIME(14) | `TIME` | ✅ |
| TIMESTAMP | TIMESTAMP(15) | `TIMESTAMP` | ✅ |
| DATETIME | DATETIME(22) | `TIMESTAMP` | ✅ |
| TIMESTAMPTZ | TIMESTAMPTZ(29) | `TIMESTAMP` | ⚠️ |
| TIMESTAMPLTZ | TIMESTAMPLTZ(30) | `TIMESTAMP` | ⚠️ |
| DATETIMETZ | DATETIMETZ(31) | `TIMESTAMP` | ⚠️ |
| DATETIMELTZ | DATETIMELTZ(32) | `TIMESTAMP` | ⚠️ |
| SET | SET(16) | `OTHER` | ⚠️ |
| MULTISET | MULTISET(17) | `OTHER` | ⚠️ |
| LIST / SEQUENCE | SEQUENCE(18) | `OTHER` | ⚠️ |
| BLOB | BLOB(23) | `BLOB` | ✅ |
| CLOB | CLOB(24) | `CLOB` | ✅ |
| ENUM | ENUM(25) | `VARCHAR` | ✅ |
| JSON | JSON(34) | `VARCHAR` | ⚠️ 표준 부재 |
| object / OID | OBJECT(19) | `OTHER` | ✅ |
| (NULL 타입 컬럼) | NULL(0) | `OTHER` | ⚠️ `Types.NULL` 주석 처리됨 |

### 세 경로의 불일치

동일 타입인데 API마다 다른 값을 반환하는 지점:

| 타입 | RSMD `getColumnType` | DBMD `getColumns` | DBMD `getTypeInfo` |
|---|---|---|---|
| BIT | `BIT`(prec8) / `BINARY` | 항상 `BINARY` | `BIT`, `BINARY` 두 행 |
| SET / MULTISET / LIST / SEQUENCE | `OTHER` | `OTHER` | `ARRAY` |
| TIMESTAMPTZ·TIMESTAMPLTZ·DATETIMETZ·DATETIMELTZ | `TIMESTAMP` | `TIMESTAMP` | `TIMESTAMP_WITH_TIMEZONE` |

- RSMD ↔ getColumns의 per-column 실차이는 **BIT 한 곳**(+ NUMERIC의 mysql 모드 분기 유무, NULL 처리 여부)뿐. TZ·컬렉션은 두 경로가 일치한다.
- 반면 **getTypeInfo만** TZ를 `TIMESTAMP_WITH_TIMEZONE`(JDBC 4.2), 컬렉션을 `ARRAY`로 광고하여 이탈한다.
- getTypeInfo 자체 오류도 있음: TYPE_NAME "NUMERIC" 행이 `TINYINT`·`BIGINT`에 매핑되고, "DOUBLE" 행이 `FLOAT`에 매핑됨.

### 개별 이슈

1. **TZ/LTZ 날짜·시간 4종** — 세 경로가 `TIMESTAMP` vs `TIMESTAMP_WITH_TIMEZONE`로 갈림. 타임존 정보 손실 + 내부 불일치. 가장 실질적인 오매핑 후보.
2. **BIT(8)** — RSMD가 `Types.BIT` + 클래스 `java.lang.Boolean`로 보고. 1바이트 비트열이 Boolean으로 표현됨.
3. **부호 없는 정수(USHORT/UINT/UBIGINT)** — RSMD·getColumns 어느 switch/체인에도 case가 없어 각각 `Types.NULL(0)` 또는 미설정. 그런데 클래스명(`findFQDN`)은 Short/Integer/Long로 정상 → 비대칭. 서버가 이 타입을 컬럼 타입으로 실제 방출하는지 도달성 확인 필요.
4. **getTypeInfo 카탈로그 오류** — 위의 이름-타입 불일치 행들.

### 정상인데 오해하기 쉬운 것

- **FLOAT → `REAL`은 정상.** JDBC 규약상 `REAL`=단정밀도(→`float`), `FLOAT`=배정밀도(→`double`). CUBRID FLOAT/REAL은 4바이트 단정밀도이므로 `Types.REAL`이 맞다. `Types.FLOAT`으로 바꾸면 오히려 틀린다.
- **ENUM / JSON → `VARCHAR`** — 표준 매핑이 없는 타입이라 방어 가능한 선택.
- **역방향(`setObject`)에는 `Types → U_TYPE` 스위치가 없다.** 값의 런타임 Java 클래스로 `U_TYPE`을 정한다(`UUType.getObjectDBtype`). `targetSqlType`은 NUMERIC/DECIMAL 스케일 조정 외엔 무시.

## 결론

전수 대조 결과, **대부분의 스칼라 타입 매핑은 규약상 적절**하다(특히 오해하기 쉬운 FLOAT→REAL도 정상). 실제로 손봐야 할 후보는 다음 4가지로 좁혀진다.

1. TZ/LTZ 날짜·시간의 `TIMESTAMP` 매핑 및 세 경로 불일치
2. BIT(8)의 Boolean 취급
3. 부호 없는 정수의 `Types.NULL` 갭 (도달성 확인 후 판정)
4. getTypeInfo 카탈로그의 이름-타입 불일치 행

NCHAR/NCHAR VARYING 매핑은 데드 경로(엔진 9.0 제거)이며, TIMETZ는 CUBRID 타입이 아니어서 드라이버도 `/* unused */`로 일관된다.

## 다음 단계

- TZ 계열·부호 없는 정수의 **실측 검증**: 실제 컬럼을 만들어 세 경로의 반환값을 대조(도달성/재현).
- 확인되면 이슈화 후 수정 제안(패치) — 특히 세 경로 간 일관성 확보.

## 참고

- [CUBRID 11.4 매뉴얼 – 데이터 타입](https://www.cubrid.org/manual/ko/11.4/sql/datatype.html)
- CUBRID JDBC 드라이버 소스: `UUType.java`(내부 타입 상수), `CUBRIDResultSetMetaData.java`(RSMD switch), `CUBRIDDatabaseMetaData.java`(getColumns / getTypeInfo), `UColumnInfo.java`(클래스명 매핑)

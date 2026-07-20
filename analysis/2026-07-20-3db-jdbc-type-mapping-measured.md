# 3-DB JDBC 타입 매핑 실측 대조 (Oracle 26ai / MySQL 9.7 / PostgreSQL 18)

- 분류: analysis
- 날짜: 2026-07-20
- 관련: [타입 보고 5개 지점 반환값 전수표](2026-07-20-cubrid-jdbc-metadata-type-mapping.md)(1편), [지점 간 불일치·규약 위반 분석](2026-07-20-cubrid-jdbc-type-mapping-issues.md)(2편), [APIS-1086 작업 기록](../APIS-1086/APIS-1086-getbestrowidentifier-pk.md)

## 요약

Oracle Database 26ai Free, MySQL 9.7.1, PostgreSQL 18.4를 컨테이너로 기동해 같은 개념 타입 26종을 getColumnType / getColumns / getBestRowIdentifier(PK·UNIQUE) / getTypeInfo로 실측하고 CUBRID(1편 전수표)와 대조했다. TZ 타입을 `TIMESTAMP`로 접는 CUBRID의 선택은 PostgreSQL과 동류이고 측정 3사 중 표준 `TIMESTAMP_WITH_TIMEZONE` 상수를 쓰는 벤더는 없었으며, PK 행 식별자는 3사 모두 정상 반환이라 CUBRID의 PK 미반환(APIS-1086)이 업계 유일 결손임이 재확인됐다.

## 목적

CUBRID JDBC의 타입 매핑 선택(1·2편)이 업계 대비 어디에 있는지 맥락화한다. 같은 개념 타입을 3사 DB의 현행 최신 버전 드라이버가 각각 어떤 `java.sql.Types`로 보고하는지 4개 메서드 관점에서 실측해, 2편에서 제안한 교정(TZ 통일, 컬렉션 정책, BIT 정책)의 우선순위 판단에 쓸 비교 근거를 만든다.

## 배경

종전에 5-DB(SQL Server 포함) 실측을 수행한 적이 있으나, 이번에는 범위를 Oracle/MySQL/PostgreSQL 3사로 좁히고 각사 최신 버전으로 다시 측정했다. CUBRID 열은 이번 재실측 대상이 아니며 1편 전수표 값(종전 라이브 실측으로 검증됨)을 쓴다.

## 범위 / 방법

### 측정 대상 버전 (전부 실측 시점 최신)

| DBMS | 엔진(실측 버전 문자열) | 컨테이너 이미지 | JDBC 드라이버 |
|---|---|---|---|
| Oracle | Oracle AI Database 26ai Free Release 23.26.2.0.0 | `gvenzl/oracle-free:slim-faststart` | ojdbc17 `23.26.2.0.0` |
| MySQL | 9.7.1 | `mysql:latest` | mysql-connector-j `9.7.0` |
| PostgreSQL | 18.4 (Debian 18.4-1.pgdg13+1) | `postgres:latest` | pgjdbc `42.7.13` |

- **환경**: podman 컨테이너(호스트 네트워크) + JDK 21(Temurin 21.0.7) + 순수 JDBC 측정 프로그램(개념별 테이블 생성 후 4개 메서드 캡처). 버전 문자열·드라이버 버전은 `DatabaseMetaData`로 실측.
- Oracle 26ai Free는 컨테이너 기본 공유메모리(64MB)에서는 인스턴스가 기동 중 죽는다(ORA-03113). shm 2GB로 기동해야 한다.
- **getBestRowIdentifier**: `scope=bestRowSession`, `nullable=false`, PK 컬럼 테이블과 NOT NULL UNIQUE 컬럼 테이블 두 변형으로 측정.
- **표기**: `java.sql.Types` 상수는 접두어 생략. `<고유 N>`=표준 `java.sql.Types` 밖의 벤더 고유 코드. `N/A`=해당 DB에 개념이 없거나 DDL 자체가 불가. `없음`=메서드가 행을 반환하지 않음. `미설정`=CUBRID 드라이버 미분기(2편 B-5).

### 개념 타입 → DB별 DDL 매핑

| 개념 | Oracle | MySQL | PostgreSQL |
|---|---|---|---|
| int16 / int32 / int64 | SMALLINT / INT / NUMBER(19,0) | SMALLINT / INT / BIGINT | smallint / integer / bigint |
| decimal | NUMBER(10,2) | DECIMAL(10,2) | numeric(10,2) |
| real / double | BINARY_FLOAT / BINARY_DOUBLE | FLOAT / DOUBLE | real / double precision |
| char / varchar | CHAR(10) / VARCHAR2(100) | CHAR(10) / VARCHAR(100) | char(10) / varchar(100) |
| bit8 | N/A | BIT(8) | bit(8) |
| binary / varbinary | RAW(16) / RAW(100) | BINARY(16) / VARBINARY(100) | bytea / bytea |
| blob / clob | BLOB / CLOB | BLOB / TEXT | bytea / text |
| date / time | DATE / N/A | DATE / TIME | date / time |
| timestamp / datetime | TIMESTAMP / TIMESTAMP | TIMESTAMP / DATETIME | timestamp / timestamp |
| timestamptz / timestampltz | TIMESTAMP WITH TIME ZONE / TIMESTAMP WITH LOCAL TIME ZONE | N/A | timestamptz / N/A |
| json | JSON | JSON | json |
| enum | N/A | ENUM('red','green','blue') | N/A |
| set / multiset / sequence | N/A | N/A | N/A / N/A / int4[] |

datetimetz/datetimeltz는 CUBRID 고유 개념이라 3사 모두 N/A. CUBRID 열의 DDL은 1편과 동일(각 개념의 네이티브 타입). MySQL에는 네이티브 SET 타입(고정 문자열 리스트, getTypeInfo 표에 존재)이 있으나 CUBRID의 컬렉션 SET과 다른 개념이라 set 행은 측정에서 제외했다.

## 발견 / 관찰

### 표 1. getColumnType (실행 SELECT의 RSMD)

| 개념 | CUBRID | Oracle | MySQL | PostgreSQL |
|---|---|---|---|---|
| int16 | `SMALLINT` | `NUMERIC` | `SMALLINT` | `SMALLINT` |
| int32 | `INTEGER` | `NUMERIC` | `INTEGER` | `INTEGER` |
| int64 | `BIGINT` | `NUMERIC` | `BIGINT` | `BIGINT` |
| decimal | `NUMERIC` | `NUMERIC` | `DECIMAL` | `NUMERIC` |
| real | `REAL` | `<고유 100>` | `REAL` | `REAL` |
| double | `DOUBLE` | `<고유 101>` | `DOUBLE` | `DOUBLE` |
| char | `CHAR` | `CHAR` | `CHAR` | `CHAR` |
| varchar | `VARCHAR` | `VARCHAR` | `VARCHAR` | `VARCHAR` |
| bit8 | `BIT` | N/A | `BIT` | `BIT` |
| binary | `BINARY` | `VARBINARY` | `BINARY` | `BINARY` |
| varbinary | `VARBINARY` | `VARBINARY` | `VARBINARY` | `BINARY` |
| blob | `BLOB` | `BLOB` | `LONGVARBINARY` | `BINARY` |
| clob | `CLOB` | `CLOB` | `LONGVARCHAR` | `VARCHAR` |
| date | `DATE` | `TIMESTAMP` | `DATE` | `DATE` |
| time | `TIME` | N/A | `TIME` | `TIME` |
| timestamp | `TIMESTAMP` | `TIMESTAMP` | `TIMESTAMP` | `TIMESTAMP` |
| datetime | `TIMESTAMP` | `TIMESTAMP` | `TIMESTAMP` | `TIMESTAMP` |
| timestamptz | `TIMESTAMP` | `<고유 -101>` | N/A | `TIMESTAMP` |
| timestampltz | `TIMESTAMP` | `<고유 -102>` | N/A | N/A |
| datetimetz | `TIMESTAMP` | N/A | N/A | N/A |
| datetimeltz | `TIMESTAMP` | N/A | N/A | N/A |
| json | `VARCHAR` | `JSON(고유 2016)` | `LONGVARCHAR` | `OTHER` |
| enum | `VARCHAR` | N/A | `CHAR` | N/A |
| set | `OTHER` | N/A | N/A | N/A |
| multiset | `OTHER` | N/A | N/A | N/A |
| sequence | `OTHER` | N/A | N/A | `ARRAY` |

### 표 2. getColumns 의 DATA_TYPE

측정 3사는 전 개념에서 표 1과 동일한 값을 보고했다. **getColumnType과 getColumns가 갈리는 타입은 4개 DB 중 CUBRID bit8이 유일하다**(표 1 `BIT` 대 표 2 `BINARY`).

| 개념 | CUBRID | Oracle | MySQL | PostgreSQL |
|---|---|---|---|---|
| bit8 | **`BINARY`** | N/A | `BIT` | `BIT` |
| 그 외 전 개념 | 표 1과 동일 | 표 1과 동일 | 표 1과 동일 | 표 1과 동일 |

### 표 3. getBestRowIdentifier 의 DATA_TYPE (PRIMARY KEY 컬럼)

| 개념 | CUBRID | Oracle | MySQL | PostgreSQL |
|---|---|---|---|---|
| int16 | 없음* | `NUMERIC` | `SMALLINT` | `SMALLINT` |
| int32 | 없음* | `NUMERIC` | `INTEGER` | `INTEGER` |
| int64 | 없음* | `NUMERIC` | `BIGINT` | `BIGINT` |
| decimal | 없음* | `NUMERIC` | `DECIMAL` | `NUMERIC` |
| real | 없음* | `<고유 100>` | `REAL` | `REAL` |
| double | 없음* | `<고유 101>` | `DOUBLE` | `DOUBLE` |
| char | 없음* | `CHAR` | `CHAR` | `CHAR` |
| varchar | 없음* | `VARCHAR` | `VARCHAR` | `VARCHAR` |
| bit8 | 없음* | N/A | `BIT` | `BIT` |
| binary | 없음* | `VARBINARY` | `BINARY` | `BINARY` |
| varbinary | 없음* | `VARBINARY` | `VARBINARY` | `BINARY` |
| date | 없음* | `TIMESTAMP` | `DATE` | `DATE` |
| time | 없음* | N/A | `TIME` | `TIME` |
| timestamp | 없음* | `TIMESTAMP` | `TIMESTAMP` | `TIMESTAMP` |
| datetime | 없음* | `TIMESTAMP` | `TIMESTAMP` | `TIMESTAMP` |
| timestamptz | 없음* | N/A** | N/A | `TIMESTAMP` |
| timestampltz | 없음* | `<고유 -102>` | N/A | N/A |
| enum | 없음* | N/A | `CHAR` | N/A |

- `*` CUBRID는 PK만 있는 테이블에서 빈 결과를 반환한다(종전 실측 0/20). 근본원인은 브로커가 PK 제약을 제약 스키마 경로로 방출하지 않는 것으로, 2편 B-1과 APIS-1086에 정리했다. **측정 3사는 전부 PK를 정상 반환**하므로 이 결손은 4개 DB 중 CUBRID가 유일하다.
- `**` Oracle은 TIMESTAMP WITH TIME ZONE 컬럼에 PK/UNIQUE 제약 자체를 허용하지 않아(DDL 거부) N/A다. LOCAL TIME ZONE 쪽은 허용되어 `<고유 -102>`가 반환된다.

### 표 4. getBestRowIdentifier 의 DATA_TYPE (NOT NULL UNIQUE 컬럼)

| 개념 | CUBRID | Oracle | MySQL | PostgreSQL |
|---|---|---|---|---|
| int16 | `SMALLINT` | `NUMERIC` | `SMALLINT` | 없음 |
| int32 | `INTEGER` | `NUMERIC` | `INTEGER` | 없음 |
| int64 | `BIGINT` | `NUMERIC` | `BIGINT` | 없음 |
| decimal | `NUMERIC` | `NUMERIC` | `DECIMAL` | 없음 |
| real | `REAL` | `<고유 100>` | `REAL` | 없음 |
| double | `DOUBLE` | `<고유 101>` | `DOUBLE` | 없음 |
| char | `CHAR` | `CHAR` | `CHAR` | 없음 |
| varchar | `VARCHAR` | `VARCHAR` | `VARCHAR` | 없음 |
| bit8 | 미설정 | N/A | `BIT` | 없음 |
| binary | 미설정 | `VARBINARY` | `BINARY` | 없음 |
| varbinary | 미설정 | `VARBINARY` | `VARBINARY` | 없음 |
| date | `DATE` | `TIMESTAMP` | `DATE` | 없음 |
| time | `TIME` | N/A | `TIME` | 없음 |
| timestamp | `TIMESTAMP` | `TIMESTAMP` | `TIMESTAMP` | 없음 |
| datetime | `TIMESTAMP` | `TIMESTAMP` | `TIMESTAMP` | 없음 |
| timestamptz | `TIMESTAMP` | N/A** | N/A | 없음 |
| timestampltz | `TIMESTAMP` | `<고유 -102>` | N/A | N/A |
| enum | `VARCHAR` | N/A | `CHAR` | N/A |

- `**` 표 3과 동일: Oracle은 TIMESTAMP WITH TIME ZONE 컬럼에 UNIQUE 제약도 허용하지 않아 DDL이 거부됐다.
- **PostgreSQL은 UNIQUE-only 컬럼에 아무것도 반환하지 않는다**(명시적 PK 필요). 종전 실측과 동일하게 재현됐다. 즉 "UNIQUE만 인정"(CUBRID 현행)과 "PK만 인정"(pgjdbc)이 정반대 전략으로 공존하고, Oracle/MySQL은 둘 다 인정한다.
- CUBRID의 `미설정`은 비트열 계열에 드라이버 분기가 없어 이전 행 값이 잔존하는 문제(2편 B-5)다.
- **ojdbc는 `scope` 파라미터에 민감**한 것도 이번에 확인했다: `bestRowTemporary(0)`이면 빈 결과, `bestRowTransaction(1)`이면 ROWID 의사컬럼 행이 추가되고, `bestRowSession(2)`이면 키 컬럼만 반환한다. pgjdbc/Connector-J는 scope 값과 무관하게 동일한 결과를 반환한다. CUBRID 드라이버가 scope를 무시하는 것(2편 B-1)이 업계에서 특이한 일은 아니지만, Oracle처럼 scope별 의미를 구현한 벤더도 있다.

### 벤더 간 두드러진 차이 (이번 실측 기준)

- **Oracle은 정수·decimal이 전부 `NUMERIC`으로 수렴**한다(네이티브 정수 계열이 NUMBER 하나뿐). SMALLINT/INT로 선언해도 NUMBER로 흡수된다.
- **표준 밖 고유 코드 사용은 Oracle뿐**: BINARY_FLOAT(100), BINARY_DOUBLE(101), TZ(-101), LTZ(-102), INTERVALYM(-103), INTERVALDS(-104), VECTOR(-105), JSON(2016). 클래스도 `oracle.sql.TIMESTAMPTZ` 같은 독자 타입을 쓴다.
- **TZ 타입을 표준 `TIMESTAMP_WITH_TIMEZONE`(JDBC 4.2) 상수로 보고하는 벤더는 측정 3사 중 없다.** PostgreSQL은 timestamptz를 평범한 `TIMESTAMP`로 접고(CUBRID와 동류), Oracle은 고유 코드로 구분 노출, MySQL은 TZ 보존 타입 자체가 없다. CUBRID getTypeInfo만 표준 상수를 광고하는 상태(2편 A-3)는 오히려 업계보다 앞선 표기인 셈이라, 2편 B-4의 교정 방향(TZ 2종을 전 지점 `TIMESTAMP_WITH_TIMEZONE`으로)이 실현되면 측정 3사보다 규약 적합성이 높아진다.
- **LOB을 `Types.BLOB`/`CLOB`으로 보고하는 것은 CUBRID와 Oracle뿐**. MySQL은 `LONGVARBINARY`/`LONGVARCHAR`, PostgreSQL은 bytea/text가 각각 `BINARY`/`VARCHAR`다.
- **json은 벤더마다 전부 다르다**: Oracle 고유 JSON(2016), MySQL `LONGVARCHAR`, PostgreSQL `OTHER`, CUBRID `VARCHAR`. 표준 상수가 없는 영역이라 CUBRID의 `VARCHAR`는 방어 가능한 선택임이 재확인된다(2편 C).
- **MySQL 9.7이 BIT(8)을 `BIT`+`java.lang.Boolean`으로 보고**한다. CUBRID RSMD ①의 BIT(8) 취급(2편 B-7)과 같은 계열의 선택이 업계에도 존재한다는 뜻이다. 다만 MySQL은 getColumnType과 getColumns가 같은 값(`BIT`)이라 CUBRID 같은 지점 간 불일치는 없다.
- **PostgreSQL 배열(int4[])만 `ARRAY` 광고와 `java.sql.Array` 이행이 일치**한다(클래스도 `java.sql.Array`). CUBRID getTypeInfo의 `ARRAY` 광고는 `getArray()` 미지원이라 거짓 약속(2편 B-3)인 점과 대비된다.
- 기타: Oracle DATE는 시각 성분을 포함해 `TIMESTAMP`로 보고되고 순수 TIME 타입이 없다. MySQL enum은 getColumnType이 `CHAR`인데 getColumns의 TYPE_NAME은 "ENUM"이다. MySQL DATETIME의 컬럼 클래스는 `java.time.LocalDateTime`(JSR-310)으로, 시간 계열을 `java.sql.Date`/`Time`/`Timestamp`로 반환하는 CUBRID·pgjdbc와 다르다.

### getTypeInfo 카탈로그

CUBRID의 33행 전체는 1편 표 5에 있다. 아래는 이번 실측 전량(Oracle 31행, MySQL 44행)과 PostgreSQL 대표 발췌(전량 463행)다.

#### Oracle (전량, 31행)

| TYPE_NAME | DATA_TYPE | PRECISION |
|---|---|---|
| VECTOR | `<고유 -105>` | 0 |
| INTERVALDS | `<고유 -104>` | 4 |
| INTERVALYM | `<고유 -103>` | 5 |
| TIMESTAMP WITH LOCAL TIME ZONE | `<고유 -102>` | 11 |
| TIMESTAMP WITH TIME ZONE | `<고유 -101>` | 13 |
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
| JSON | `JSON(고유 2016)` | -1 |

Oracle도 NUMBER 하나로 BIT/TINYINT/BIGINT/NUMERIC/INTEGER/SMALLINT 6개 JDBC 타입을 커버한다. CUBRID getTypeInfo의 NUMERIC 다중 행(1편 표 5)과 같은 구성이며, 하나의 네이티브가 여러 DATA_TYPE 행을 갖는 것 자체는 업계 표준 관행임을 보여준다.

#### MySQL (전량, 44행)

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

MySQL 9.7에도 VECTOR 타입이 있고 `LONGVARBINARY`로 광고한다(Oracle은 고유 코드). MySQL의 FLOAT→`REAL`, DOUBLE/REAL(별칭)→`DOUBLE` 구성은 CUBRID의 FLOAT→`REAL` 매핑(2편 C)과 같은 논리다.

#### PostgreSQL (대표 발췌, 전량 463행)

생략한 행: 배열 타입 347행(전부 `ARRAY`, TYPE_NAME이 `_` 접두이며 대표로 int4 배열인 `_int4` 한 행만 남김), 지오메트리/네트워크/카탈로그 타입 등 `OTHER` 계열 다수, serial 계열(serial/smallserial/bigserial, 각각 `INTEGER`/`SMALLINT`/`BIGINT`), name(`VARCHAR`), information_schema 도메인 5행(`DISTINCT`), 복합 타입 1행(`STRUCT`). `ARRAY` 행은 전부 `_` 접두 타입이지만 그 역은 아니다(`_pg_user_mappings`는 `STRUCT`).

| TYPE_NAME | DATA_TYPE | PRECISION |
|---|---|---|
| bool | BIT | 0 |
| bit | BIT | 83886080 |
| int2 | SMALLINT | 0 |
| int4 | INTEGER | 0 |
| int8 | BIGINT | 0 |
| numeric | NUMERIC | 1000 |
| float4 | REAL | 0 |
| float8 | DOUBLE | 0 |
| money | DOUBLE | 0 |
| bytea | BINARY | 0 |
| char | CHAR | 0 |
| bpchar | CHAR | 10485760 |
| varchar | VARCHAR | 10485760 |
| text | VARCHAR | 0 |
| varbit | OTHER | 83886080 |
| date | DATE | 0 |
| time | TIME | 6 |
| timetz | TIME | 6 |
| timestamp | TIMESTAMP | 6 |
| timestamptz | TIMESTAMP | 6 |
| interval | OTHER | 6 |
| uuid | OTHER | 0 |
| json | OTHER | 0 |
| jsonb | OTHER | 0 |
| xml | SQLXML | 0 |
| _int4 | ARRAY | 0 |
| refcursor | REF_CURSOR | 0 |

## 결론

- **CUBRID의 스칼라 매핑 값은 업계 주류와 같은 자리에 있다.** 정수/decimal/문자/날짜 계열은 MySQL·PostgreSQL과 사실상 동일하고(MySQL은 decimal을 동의어 상수인 `DECIMAL`로 보고), TZ를 `TIMESTAMP`로 접는 것도 PostgreSQL과 동류다. 측정 3사 중 표준 `TIMESTAMP_WITH_TIMEZONE`을 쓰는 벤더는 없었다.
- **PK 행 식별자 결손은 CUBRID만의 문제로 재확인됐다.** 3사 모두 PK 컬럼을 정상 반환한다. UNIQUE-only 미반환(PostgreSQL)처럼 구현 전략이 갈리는 영역은 있으나, PK가 안 되는 벤더는 없다. APIS-1086의 우선순위(2편 결론에서 '높음')를 실측이 뒷받침한다.
- **getColumnType과 getColumns가 갈리는 타입(bit8)은 CUBRID가 유일**하다. 3사는 두 지점이 전 개념에서 일치한다. 2편 A-1(BIT 불일치) 교정의 근거가 된다.
- **BIT(8)→Boolean 취급 자체는 MySQL에도 있는 선택**이라, 2편 B-7의 교정은 "Boolean 취급 폐지"보다 "지점 간 통일" 쪽이 업계 관행에 부합한다.
- getTypeInfo에서 하나의 네이티브 타입이 여러 DATA_TYPE 행을 갖는 구성(CUBRID NUMERIC 3행)은 Oracle NUMBER 6행에서 보듯 표준 관행이다. 문제는 행 구성이 아니라 정렬 부재(2편 B-2)라는 판단도 유지된다.

## 다음 단계

- 이번 측정 스크립트(개념 카탈로그 × 4개 메서드)를 매핑 통일 작업(2편 근본 원인)의 회귀 기준으로 재사용. CUBRID를 포함한 재실측은 통일 구현 시점에 수행.
- Oracle의 scope 민감성(bestRowTemporary=빈 결과)은 CUBRID 드라이버의 scope 처리 설계 시 참고.

## 참고

- 1편(전수표): [2026-07-20-cubrid-jdbc-metadata-type-mapping.md](2026-07-20-cubrid-jdbc-metadata-type-mapping.md), 2편(분석): [2026-07-20-cubrid-jdbc-type-mapping-issues.md](2026-07-20-cubrid-jdbc-type-mapping-issues.md)
- getBestRowIdentifier PK 결손: [APIS-1086 작업 기록](../APIS-1086/APIS-1086-getbestrowidentifier-pk.md)
- 컨테이너 이미지: `gvenzl/oracle-free:slim-faststart`(Oracle Database 26ai Free), `mysql:latest`(9.7.1), `postgres:latest`(18.4)
- JDBC 드라이버(Maven Central 최신): ojdbc17 23.26.2.0.0, mysql-connector-j 9.7.0, postgresql(pgjdbc) 42.7.13
- JDBC 규약: Java SE javadoc `java.sql.DatabaseMetaData#getBestRowIdentifier`(scope 상수), `java.sql.Types`

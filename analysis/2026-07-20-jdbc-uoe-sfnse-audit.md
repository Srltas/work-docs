# CUBRID JDBC: UnsupportedOperationException → SQLFeatureNotSupportedException 감사

- 분류: analysis
- 날짜: 2026-07-20
- 관련: CUBRID JDBC 드라이버(`cubrid-jdbc/src/jdbc/cubrid/jdbc/driver`), JDBC 4.x 확장 로드맵

## 요약
CUBRID JDBC 드라이버 15개 파일에서 미지원 메서드가 `SQLException(new UnsupportedOperationException())` 등으로 예외를 던지는 곳이 총 296건이며, 이 중 278건은 `SQLFeatureNotSupportedException`(SFNSE)으로 전환 가능하고 18건(wrapper 14, client-info 2, statement-event-listener 2)은 계약상 전환이 부적합하거나 불가능하다.

## 목적
미지원 JDBC 메서드가 던지는 예외를 JDBC 4.0 표준인 `java.sql.SQLFeatureNotSupportedException`으로 개선하기 위해, 어떤 메서드가 문제이고 각각 어떻게 고쳐야 하는지 메서드 단위로 전수 조사한다.

## 배경
현재 드라이버는 미지원 기능을 `throw new SQLException(new java.lang.UnsupportedOperationException())` 패턴으로 신호한다. 이 경우 예외의 원인(cause)이 `RuntimeException`이라 SQLState가 비어 있고, 호출자(HikariCP, Hibernate, Spring 등)가 "미지원 기능"과 "실제 DB 오류"를 구분할 수 없다. `SQLFeatureNotSupportedException`은 이 의미를 표준적으로 전달하며 SQLState를 `0A000`(feature not supported) 계열로 세팅할 수 있다. 참고로 현재 코드베이스에서 `SQLFeatureNotSupportedException`은 단 한 번도 사용되지 않았다(0건).

## 범위 / 방법
- 대상: `cubrid-jdbc/src/jdbc/cubrid/jdbc/driver` 하위 드라이버 소스(테스트케이스·build output 제외).
- 방법: `UnsupportedOperationException` 전 출현(296건)의 라인 목록을 확보한 뒤, 파일별로 각 메서드의 시그니처와 `throws` 절, 구현 인터페이스, JDBC 버전을 읽어 분류하고 개선안을 도출.
- 검증: 파일별 추출과 독립 재검증을 병렬로 수행하고, 별도로 라인→메서드 시그니처 매핑을 생성해 296건 완전 일치(누락·초과·중복 0)를 교차 확인.

### 현재 던지는 형태 4종
| 형태 | 코드 패턴 | 건수 | 문제점 |
|------|-----------|------|--------|
| A | `throw new SQLException(new java.lang.UnsupportedOperationException());` | 203 | 원인이 RuntimeException이라 SQLState=null, 미지원 기능을 표준적으로 식별 불가 |
| B | `throw new SQLException(new UnsupportedOperationException());` | 84 | A와 동일(import만 다름) |
| C | `throw new java.lang.UnsupportedOperationException(...);` (raw, 미래핑) | 7 | checked 계약을 우회해 unchecked가 유출되는 실제 버그 |
| D | `SQLClientInfoException e; e.initCause(new UOE); throw e;` | 2 | 예외 타입 자체는 맞음(원인만 불필요) |

> 이하 표에서 약어: `A/B/C/D`는 위 형태, `UOE`=`UnsupportedOperationException`, `SFNSE`=`throw new SQLFeatureNotSupportedException();`.

## 발견 / 관찰

### 핵심 제약
`SQLFeatureNotSupportedException`은 checked 예외(`SQLFeatureNotSupportedException` → `SQLNonTransientException` → `SQLException`)이다. 따라서 메서드의 `throws` 절이 `SQLException`을 허용해야만 던질 수 있고, 이 제약 때문에 296건이 균일하지 않고 아래 7분류로 갈린다.

### 분류별 처리 방침
| 분류 | 건수 | SFNSE 전환 | 권장 개선 |
|------|------|:---:|-----------|
| 표준 (throws SQLException) | 264 | 가능 | `throw new SQLFeatureNotSupportedException();` (+ import 추가) |
| deprecated(위임가능) | 7 | 가능 | SFNSE 또는 지원 오버로드로 위임(예: `return getBigDecimal(col);`) |
| 기타/주의 (setTimestamptz, getCharacterStream) | 3 | 가능 | SFNSE (단 getCharacterStream은 현재 raw 버그라 반드시 수정) |
| parent-logger (getParentLogger ×4) | 4 | 가능 | `throw new SFNSE();` + 시그니처에 `throws SQLFeatureNotSupportedException` 추가(현재 raw throw는 계약 위반 버그) |
| wrapper (unwrap/isWrapperFor ×14) | 14 | 부적합 | 구현 권장: `isWrapperFor→return iface.isInstance(this);`, `unwrap→return iface.cast(this);` 실패 시 plain SQLException |
| client-info (setClientInfo ×2) | 2 | 불가 | `throws SQLClientInfoException`만 선언되어 SFNSE 컴파일 불가, 제대로 된 `SQLClientInfoException(failuresMap)`로 던질 것 |
| statement-event-listener (add/remove ×2) | 2 | 불가 | `throws` 절 자체가 없어 checked 예외 불가, no-op 구현 권장(또는 unchecked 유지) |

정리하면 SFNSE 전환 가능 278건, 전환 부적합·불가 18건(wrapper 14 + client-info 2 + listener 2).

### 실제 버그로 분류되는 raw throw (form C, 7건)
raw unchecked 예외가 그대로 유출된다. `getParentLogger` 4개는 스펙이 `SQLFeatureNotSupportedException`을 요구하는데 raw로 던져 계약 위반이고, `getCharacterStream(423)`은 `throws SQLException`을 선언해 놓고 unchecked를 던지는 버그이며, `add/removeStatementEventListener`는 애초에 throws 절이 없다.

### 특수 케이스 상세 (개별 판단 필요한 32건)
| 파일:라인 | 메서드 | 분류 | 현재 | 개선 |
|-----------|--------|------|:---:|------|
| CUBRIDConnection:1048 | `setClientInfo(Properties)` | client-info(불가) | D | SQLClientInfoException 유지 |
| CUBRIDConnection:1055 | `setClientInfo(String,String)` | client-info(불가) | D | SQLClientInfoException 유지 |
| CUBRIDConnection:1061 / :1066 | `isWrapperFor` / `unwrap` | wrapper(부적합) | A | 구현(isInstance/cast) |
| CUBRIDStatement:770 / :775 | `isWrapperFor` / `unwrap` | wrapper(부적합) | A | 구현 |
| CUBRIDResultSet:2099 / :2104 | `isWrapperFor` / `unwrap` | wrapper(부적합) | A | 구현 |
| CUBRIDResultSetWithoutQuery:1216 / :1221 | `isWrapperFor` / `unwrap` | wrapper(부적합) | A | 구현 |
| CUBRIDResultSetMetaData:784 / :789 | `isWrapperFor` / `unwrap` | wrapper(부적합) | A | 구현 |
| CUBRIDDatabaseMetaData:2921 / :2926 | `isWrapperFor` / `unwrap` | wrapper(부적합) | A | 구현 |
| CUBRIDDataSource:133 / :138 | `isWrapperFor` / `unwrap` | wrapper(부적합) | A | 구현 |
| CUBRIDDriver:343 | `getParentLogger()` | parent-logger | C(raw버그) | SFNSE + throws 추가 |
| CUBRIDDataSource:143 | `getParentLogger()` | parent-logger | C(raw버그) | SFNSE + throws 추가 |
| CUBRIDConnectionPoolDataSource:121 | `getParentLogger()` | parent-logger | C(raw버그) | SFNSE + throws 추가 |
| CUBRIDXADataSource:92 | `getParentLogger()` | parent-logger | C(raw버그) | SFNSE + throws 추가 |
| CUBRIDPooledConnection:150 / :155 | `add/removeStatementEventListener` | listener(불가) | C(raw) | no-op 구현 |
| CUBRIDResultSetWithoutQuery:423 | `getCharacterStream(int)` | 기타 | C(raw버그, 메시지有) | SFNSE (unchecked 유출 제거) |
| CUBRIDCallableStatement:412 / :454 | `setTimestamptz(...)` (CUBRID 확장) | 기타 | B | SFNSE(가능) |
| CUBRIDCallableStatement:296 · CUBRIDResultSet:458,635 · CUBRIDResultSetWithoutQuery:240,301,346,370 | `getBigDecimal(_,scale)` / `getUnicodeStream` | deprecated | A/B | SFNSE 또는 위임 |

### 파일별 전체 메서드 표 (296건 전부)

#### CUBRIDConnection.java (24건, 특수 4)
| # | 메서드 | 라인 | 현재 | 분류 | 개선안 |
|---|--------|-----|:---:|------|--------|
| 1 | `nativeSQL(String sql)` | 176 | B | 표준 | SFNSE |
| 2 | `getTypeMap()` | 451 | A | 표준 | SFNSE |
| 3 | `setTypeMap(Map)` | 455 | A | 표준 | SFNSE |
| 4 | `createStatement(int,int,int)` | 464 | A | 표준 | SFNSE |
| 5 | `prepareStatement(String,int,int,int)` | 506 | A | 표준 | SFNSE |
| 6 | `releaseSavepoint(Savepoint)` | 525 | A | 표준 | SFNSE |
| 7 | `rollback(Savepoint)` | 539 | A | 표준 | SFNSE |
| 8 | `setSavepoint()` | 563 | A | 표준 | SFNSE |
| 9 | `setSavepoint(String)` | 584 | A | 표준 | SFNSE |
| 10 | `createNClob()` | 1006 | A | 표준 | SFNSE |
| 11 | `createArrayOf(String,Object[])` | 1011 | A | 표준 | SFNSE |
| 12 | `createSQLXML()` | 1016 | A | 표준 | SFNSE |
| 13 | `createStruct(String,Object[])` | 1021 | A | 표준 | SFNSE |
| 14 | `getClientInfo()` | 1026 | A | 표준 | SFNSE |
| 15 | `getClientInfo(String)` | 1031 | A | 표준 | SFNSE |
| 16 | `setClientInfo(Properties)` | 1048 | D | client-info(불가) | SQLClientInfoException 유지 |
| 17 | `setClientInfo(String,String)` | 1055 | D | client-info(불가) | SQLClientInfoException 유지 |
| 18 | `isWrapperFor(Class)` | 1061 | A | wrapper(부적합) | `return iface.isInstance(this);` |
| 19 | `unwrap(Class)` | 1066 | A | wrapper(부적합) | `iface.cast(this)` / plain SQLException |
| 20 | `setSchema(String)` | 1071 | A | 표준 | SFNSE |
| 21 | `getSchema()` | 1076 | A | 표준 | SFNSE |
| 22 | `abort(Executor)` | 1081 | A | 표준 | SFNSE |
| 23 | `setNetworkTimeout(Executor,int)` | 1086 | A | 표준 | SFNSE |
| 24 | `getNetworkTimeout()` | 1091 | A | 표준 | SFNSE |

#### CUBRIDStatement.java (7건, 특수 2)
| # | 메서드 | 라인 | 현재 | 분류 | 개선안 |
|---|--------|-----|:---:|------|--------|
| 1 | `getMoreResults(int)` | 637 | A | 표준 | SFNSE |
| 2 | `isPoolable()` | 760 | A | 표준 | SFNSE |
| 3 | `setPoolable(boolean)` | 765 | A | 표준 | SFNSE |
| 4 | `isWrapperFor(Class)` | 770 | A | wrapper(부적합) | `return iface.isInstance(this);` |
| 5 | `unwrap(Class)` | 775 | A | wrapper(부적합) | `iface.cast(this)` / plain SQLException |
| 6 | `closeOnCompletion()` | 1011 | A | 표준 | SFNSE |
| 7 | `isCloseOnCompletion()` | 1016 | A | 표준 | SFNSE |

#### CUBRIDPreparedStatement.java (19건)
| # | 메서드 | 라인 | 현재 | 분류 | 개선안 |
|---|--------|-----|:---:|------|--------|
| 1 | `setUnicodeStream(int,InputStream,int)` | 350 | B | 표준(deprecated) | SFNSE |
| 2 | `setRef(int,Ref)` | 579 | B | 표준 | SFNSE |
| 3 | `setArray(int,Array)` | 679 | B | 표준 | SFNSE |
| 4 | `getParameterMetaData()` | 813 | B | 표준 | SFNSE |
| 5 | `setURL(int,URL)` | 830 | B | 표준 | SFNSE |
| 6 | `setBinaryStream(int,InputStream)` | 962 | A | 표준 | SFNSE |
| 7 | `setBinaryStream(int,InputStream,long)` | 970 | A | 표준 | SFNSE |
| 8 | `setAsciiStream(int,InputStream)` | 977 | A | 표준 | SFNSE |
| 9 | `setAsciiStream(int,InputStream,long)` | 984 | A | 표준 | SFNSE |
| 10 | `setCharacterStream(int,Reader)` | 991 | A | 표준 | SFNSE |
| 11 | `setCharacterStream(int,Reader,long)` | 999 | A | 표준 | SFNSE |
| 12 | `setNCharacterStream(int,Reader)` | 1004 | A | 표준 | SFNSE |
| 13 | `setNCharacterStream(int,Reader,long)` | 1010 | A | 표준 | SFNSE |
| 14 | `setNClob(int,NClob)` | 1015 | A | 표준 | SFNSE |
| 15 | `setNClob(int,Reader)` | 1020 | A | 표준 | SFNSE |
| 16 | `setNClob(int,Reader,long)` | 1025 | A | 표준 | SFNSE |
| 17 | `setNString(int,String)` | 1030 | A | 표준 | SFNSE |
| 18 | `setRowId(int,RowId)` | 1035 | A | 표준 | SFNSE |
| 19 | `setSQLXML(int,SQLXML)` | 1040 | A | 표준 | SFNSE |

#### CUBRIDCallableStatement.java (96건, 특수 3)
| # | 메서드 | 라인 | 현재 | 분류 | 개선안 |
|---|--------|-----|:---:|------|--------|
| 1 | `getBigDecimal(int,int scale)` | 296 | A | deprecated | SFNSE(또는 위임) |
| 2 | `getRef(int)` | 300 | A | 표준 | SFNSE |
| 3 | `getArray(int)` | 332 | A | 표준 | SFNSE |
| 4 | `getURL(int)` | 348 | A | 표준 | SFNSE |
| 5 | `setURL(String,URL)` | 352 | B | 표준 | SFNSE |
| 6 | `setNull(String,int)` | 356 | B | 표준 | SFNSE |
| 7 | `setBoolean(String,boolean)` | 360 | B | 표준 | SFNSE |
| 8 | `setByte(String,byte)` | 364 | B | 표준 | SFNSE |
| 9 | `setShort(String,short)` | 368 | B | 표준 | SFNSE |
| 10 | `setInt(String,int)` | 372 | B | 표준 | SFNSE |
| 11 | `setLong(String,long)` | 376 | B | 표준 | SFNSE |
| 12 | `setFloat(String,float)` | 380 | B | 표준 | SFNSE |
| 13 | `setDouble(String,double)` | 384 | B | 표준 | SFNSE |
| 14 | `setBigDecimal(String,BigDecimal)` | 388 | B | 표준 | SFNSE |
| 15 | `setString(String,String)` | 392 | B | 표준 | SFNSE |
| 16 | `setBytes(String,byte[])` | 396 | B | 표준 | SFNSE |
| 17 | `setDate(String,Date)` | 400 | B | 표준 | SFNSE |
| 18 | `setTime(String,Time)` | 404 | B | 표준 | SFNSE |
| 19 | `setTimestamp(String,Timestamp)` | 408 | B | 표준 | SFNSE |
| 20 | `setTimestamptz(String,CUBRIDTimestamptz)` | 412 | B | 기타(확장) | SFNSE |
| 21 | `setAsciiStream(String,InputStream,int)` | 416 | B | 표준 | SFNSE |
| 22 | `setBinaryStream(String,InputStream,int)` | 420 | B | 표준 | SFNSE |
| 23 | `setObject(String,Object,int,int)` | 425 | B | 표준 | SFNSE |
| 24 | `setObject(String,Object,int)` | 429 | B | 표준 | SFNSE |
| 25 | `setObject(String,Object)` | 433 | B | 표준 | SFNSE |
| 26 | `setCharacterStream(String,Reader,int)` | 437 | B | 표준 | SFNSE |
| 27 | `setDate(String,Date,Calendar)` | 441 | B | 표준 | SFNSE |
| 28 | `setTime(String,Time,Calendar)` | 445 | B | 표준 | SFNSE |
| 29 | `setTimestamp(String,Timestamp,Calendar)` | 449 | B | 표준 | SFNSE |
| 30 | `setTimestamptz(String,CUBRIDTimestamptz,Calendar)` | 454 | B | 기타(확장) | SFNSE |
| 31 | `setNull(String,int,String)` | 458 | B | 표준 | SFNSE |
| 32 | `getString(String)` | 462 | B | 표준 | SFNSE |
| 33 | `getBoolean(String)` | 466 | B | 표준 | SFNSE |
| 34 | `getByte(String)` | 470 | B | 표준 | SFNSE |
| 35 | `getShort(String)` | 474 | B | 표준 | SFNSE |
| 36 | `getInt(String)` | 478 | B | 표준 | SFNSE |
| 37 | `getLong(String)` | 482 | B | 표준 | SFNSE |
| 38 | `getFloat(String)` | 486 | B | 표준 | SFNSE |
| 39 | `getDouble(String)` | 490 | B | 표준 | SFNSE |
| 40 | `getBytes(String)` | 494 | B | 표준 | SFNSE |
| 41 | `getDate(String)` | 498 | B | 표준 | SFNSE |
| 42 | `getTime(String)` | 502 | B | 표준 | SFNSE |
| 43 | `getTimestamp(String)` | 506 | B | 표준 | SFNSE |
| 44 | `getObject(String)` | 510 | B | 표준 | SFNSE |
| 45 | `getObject(int,Map)` | 514 | B | 표준 | SFNSE |
| 46 | `getObject(String,Map)` | 518 | B | 표준 | SFNSE |
| 47 | `getBigDecimal(String)` | 522 | B | 표준 | SFNSE |
| 48 | `getRef(String)` | 526 | B | 표준 | SFNSE |
| 49 | `getBlob(String)` | 530 | B | 표준 | SFNSE |
| 50 | `getClob(String)` | 534 | B | 표준 | SFNSE |
| 51 | `getArray(String)` | 538 | B | 표준 | SFNSE |
| 52 | `getDate(String,Calendar)` | 542 | B | 표준 | SFNSE |
| 53 | `getTime(String,Calendar)` | 546 | B | 표준 | SFNSE |
| 54 | `getTimestamp(String,Calendar)` | 550 | B | 표준 | SFNSE |
| 55 | `getURL(String)` | 554 | B | 표준 | SFNSE |
| 56 | `registerOutParameter(String,int)` | 570 | B | 표준 | SFNSE |
| 57 | `registerOutParameter(String,int,int)` | 574 | B | 표준 | SFNSE |
| 58 | `registerOutParameter(String,int,String)` | 579 | B | 표준 | SFNSE |
| 59 | `executeBatch()` | 583 | B | 표준 | SFNSE |
| 60 | `addBatch()` | 587 | B | 표준 | SFNSE |
| 61 | `getCharacterStream(int)` | 639 | A | 표준 | SFNSE |
| 62 | `getCharacterStream(String)` | 644 | A | 표준 | SFNSE |
| 63 | `getNCharacterStream(int)` | 649 | A | 표준 | SFNSE |
| 64 | `getNCharacterStream(String)` | 654 | A | 표준 | SFNSE |
| 65 | `getNClob(int)` | 659 | A | 표준 | SFNSE |
| 66 | `getNClob(String)` | 664 | A | 표준 | SFNSE |
| 67 | `getNString(int)` | 669 | A | 표준 | SFNSE |
| 68 | `getNString(String)` | 674 | A | 표준 | SFNSE |
| 69 | `getRowId(int)` | 679 | A | 표준 | SFNSE |
| 70 | `getRowId(String)` | 684 | A | 표준 | SFNSE |
| 71 | `getSQLXML(int)` | 689 | A | 표준 | SFNSE |
| 72 | `getSQLXML(String)` | 694 | A | 표준 | SFNSE |
| 73 | `setAsciiStream(String,InputStream)` | 699 | A | 표준 | SFNSE |
| 74 | `setAsciiStream(String,InputStream,long)` | 705 | A | 표준 | SFNSE |
| 75 | `setBinaryStream(String,InputStream)` | 710 | A | 표준 | SFNSE |
| 76 | `setBinaryStream(String,InputStream,long)` | 716 | A | 표준 | SFNSE |
| 77 | `setBlob(String,Blob)` | 721 | A | 표준 | SFNSE |
| 78 | `setBlob(String,InputStream)` | 726 | A | 표준 | SFNSE |
| 79 | `setBlob(String,InputStream,long)` | 732 | A | 표준 | SFNSE |
| 80 | `setCharacterStream(String,Reader)` | 737 | A | 표준 | SFNSE |
| 81 | `setCharacterStream(String,Reader,long)` | 743 | A | 표준 | SFNSE |
| 82 | `setClob(String,Clob)` | 748 | A | 표준 | SFNSE |
| 83 | `setClob(String,Reader)` | 753 | A | 표준 | SFNSE |
| 84 | `setClob(String,Reader,long)` | 758 | A | 표준 | SFNSE |
| 85 | `setNCharacterStream(String,Reader)` | 763 | A | 표준 | SFNSE |
| 86 | `setNCharacterStream(String,Reader,long)` | 769 | A | 표준 | SFNSE |
| 87 | `setNClob(String,NClob)` | 774 | A | 표준 | SFNSE |
| 88 | `setNClob(String,Reader)` | 779 | A | 표준 | SFNSE |
| 89 | `setNClob(String,Reader,long)` | 784 | A | 표준 | SFNSE |
| 90 | `setNString(String,String)` | 789 | A | 표준 | SFNSE |
| 91 | `setRowId(String,RowId)` | 794 | A | 표준 | SFNSE |
| 92 | `setSQLXML(String,SQLXML)` | 799 | A | 표준 | SFNSE |
| 93 | `closeOnCompletion()` | 804 | B | 표준 | SFNSE |
| 94 | `isCloseOnCompletion()` | 809 | B | 표준 | SFNSE |
| 95 | `getObject(int,Class<T>)` | 814 | B | 표준 | SFNSE |
| 96 | `getObject(String,Class<T>)` | 819 | B | 표준 | SFNSE |

#### CUBRIDResultSet.java (66건, 특수 4)
| # | 메서드 | 라인 | 현재 | 분류 | 개선안 |
|---|--------|-----|:---:|------|--------|
| 1 | `getBigDecimal(int,int scale)` | 458 | A | deprecated | SFNSE(또는 위임) |
| 2 | `getUnicodeStream(int)` | 566 | A | 표준(deprecated) | SFNSE |
| 3 | `getBigDecimal(String,int scale)` | 635 | B | deprecated | SFNSE(또는 위임) |
| 4 | `getUnicodeStream(String)` | 659 | A | 표준(deprecated) | SFNSE |
| 5 | `getObject(int,Map)` | 1438 | B | 표준 | SFNSE |
| 6 | `getRef(int)` | 1442 | B | 표준 | SFNSE |
| 7 | `getArray(int)` | 1474 | B | 표준 | SFNSE |
| 8 | `getObject(String,Map)` | 1478 | B | 표준 | SFNSE |
| 9 | `getRef(String)` | 1482 | B | 표준 | SFNSE |
| 10 | `getArray(String)` | 1494 | B | 표준 | SFNSE |
| 11 | `getURL(int)` | 1524 | B | 표준 | SFNSE |
| 12 | `getURL(String)` | 1528 | B | 표준 | SFNSE |
| 13 | `updateArray(int,Array)` | 1532 | B | 표준 | SFNSE |
| 14 | `updateArray(String,Array)` | 1536 | B | 표준 | SFNSE |
| 15 | `updateRef(int,Ref)` | 1556 | B | 표준 | SFNSE |
| 16 | `updateRef(String,Ref)` | 1560 | B | 표준 | SFNSE |
| 17 | `getNCharacterStream(int)` | 1857 | A | 표준 | SFNSE |
| 18 | `getNCharacterStream(String)` | 1862 | A | 표준 | SFNSE |
| 19 | `getNClob(int)` | 1867 | A | 표준 | SFNSE |
| 20 | `getNClob(String)` | 1872 | A | 표준 | SFNSE |
| 21 | `getNString(int)` | 1877 | A | 표준 | SFNSE |
| 22 | `getNString(String)` | 1882 | A | 표준 | SFNSE |
| 23 | `getRowId(int)` | 1887 | A | 표준 | SFNSE |
| 24 | `getRowId(String)` | 1892 | A | 표준 | SFNSE |
| 25 | `getSQLXML(int)` | 1897 | A | 표준 | SFNSE |
| 26 | `getSQLXML(String)` | 1902 | A | 표준 | SFNSE |
| 27 | `updateAsciiStream(int,InputStream)` | 1912 | A | 표준 | SFNSE |
| 28 | `updateAsciiStream(String,InputStream)` | 1917 | A | 표준 | SFNSE |
| 29 | `updateAsciiStream(int,InputStream,long)` | 1922 | A | 표준 | SFNSE |
| 30 | `updateAsciiStream(String,InputStream,long)` | 1928 | A | 표준 | SFNSE |
| 31 | `updateBinaryStream(int,InputStream)` | 1933 | A | 표준 | SFNSE |
| 32 | `updateBinaryStream(String,InputStream)` | 1938 | A | 표준 | SFNSE |
| 33 | `updateBinaryStream(int,InputStream,long)` | 1944 | A | 표준 | SFNSE |
| 34 | `updateBinaryStream(String,InputStream,long)` | 1950 | A | 표준 | SFNSE |
| 35 | `updateBlob(int,InputStream)` | 1955 | A | 표준 | SFNSE |
| 36 | `updateBlob(String,InputStream)` | 1960 | A | 표준 | SFNSE |
| 37 | `updateBlob(int,InputStream,long)` | 1966 | A | 표준 | SFNSE |
| 38 | `updateBlob(String,InputStream,long)` | 1972 | A | 표준 | SFNSE |
| 39 | `updateCharacterStream(int,Reader)` | 1977 | A | 표준 | SFNSE |
| 40 | `updateCharacterStream(String,Reader)` | 1982 | A | 표준 | SFNSE |
| 41 | `updateCharacterStream(int,Reader,long)` | 1987 | A | 표준 | SFNSE |
| 42 | `updateCharacterStream(String,Reader,long)` | 1993 | A | 표준 | SFNSE |
| 43 | `updateClob(int,Reader)` | 1998 | A | 표준 | SFNSE |
| 44 | `updateClob(String,Reader)` | 2003 | A | 표준 | SFNSE |
| 45 | `updateClob(int,Reader,long)` | 2008 | A | 표준 | SFNSE |
| 46 | `updateClob(String,Reader,long)` | 2013 | A | 표준 | SFNSE |
| 47 | `updateNCharacterStream(int,Reader)` | 2018 | A | 표준 | SFNSE |
| 48 | `updateNCharacterStream(String,Reader)` | 2023 | A | 표준 | SFNSE |
| 49 | `updateNCharacterStream(int,Reader,long)` | 2028 | A | 표준 | SFNSE |
| 50 | `updateNCharacterStream(String,Reader,long)` | 2034 | A | 표준 | SFNSE |
| 51 | `updateNClob(int,NClob)` | 2039 | A | 표준 | SFNSE |
| 52 | `updateNClob(String,NClob)` | 2044 | A | 표준 | SFNSE |
| 53 | `updateNClob(int,Reader)` | 2049 | A | 표준 | SFNSE |
| 54 | `updateNClob(String,Reader)` | 2054 | A | 표준 | SFNSE |
| 55 | `updateNClob(int,Reader,long)` | 2059 | A | 표준 | SFNSE |
| 56 | `updateNClob(String,Reader,long)` | 2064 | A | 표준 | SFNSE |
| 57 | `updateNString(int,String)` | 2069 | A | 표준 | SFNSE |
| 58 | `updateNString(String,String)` | 2074 | A | 표준 | SFNSE |
| 59 | `updateRowId(int,RowId)` | 2079 | A | 표준 | SFNSE |
| 60 | `updateRowId(String,RowId)` | 2084 | A | 표준 | SFNSE |
| 61 | `updateSQLXML(int,SQLXML)` | 2089 | A | 표준 | SFNSE |
| 62 | `updateSQLXML(String,SQLXML)` | 2094 | A | 표준 | SFNSE |
| 63 | `isWrapperFor(Class)` | 2099 | A | wrapper(부적합) | `return iface.isInstance(this);` |
| 64 | `unwrap(Class)` | 2104 | A | wrapper(부적합) | `iface.cast(this)` / plain SQLException |
| 65 | `getObject(int,Class<T>)` | 2109 | A | 표준 | SFNSE |
| 66 | `getObject(String,Class<T>)` | 2114 | A | 표준 | SFNSE |

#### CUBRIDResultSetWithoutQuery.java (56건, 특수 7)
| # | 메서드 | 라인 | 현재 | 분류 | 개선안 |
|---|--------|-----|:---:|------|--------|
| 1 | `getBigDecimal(int,int scale)` | 240 | B | deprecated | SFNSE(또는 위임) |
| 2 | `getUnicodeStream(int)` | 301 | B | deprecated | SFNSE |
| 3 | `getBigDecimal(String,int scale)` | 346 | B | deprecated | SFNSE(또는 위임) |
| 4 | `getUnicodeStream(String)` | 370 | B | deprecated | SFNSE |
| 5 | `getCharacterStream(int)` | 423 | C(raw버그) | 기타 | SFNSE (unchecked 유출 제거) |
| 6 | `getHoldability()` | 969 | A | 표준 | SFNSE |
| 7 | `getNCharacterStream(int)` | 974 | A | 표준 | SFNSE |
| 8 | `getNCharacterStream(String)` | 979 | A | 표준 | SFNSE |
| 9 | `getNClob(int)` | 984 | A | 표준 | SFNSE |
| 10 | `getNClob(String)` | 989 | A | 표준 | SFNSE |
| 11 | `getNString(int)` | 994 | A | 표준 | SFNSE |
| 12 | `getNString(String)` | 999 | A | 표준 | SFNSE |
| 13 | `getRowId(int)` | 1004 | A | 표준 | SFNSE |
| 14 | `getRowId(String)` | 1009 | A | 표준 | SFNSE |
| 15 | `getSQLXML(int)` | 1014 | A | 표준 | SFNSE |
| 16 | `getSQLXML(String)` | 1019 | A | 표준 | SFNSE |
| 17 | `updateAsciiStream(int,InputStream)` | 1029 | A | 표준 | SFNSE |
| 18 | `updateAsciiStream(String,InputStream)` | 1034 | A | 표준 | SFNSE |
| 19 | `updateAsciiStream(int,InputStream,long)` | 1039 | A | 표준 | SFNSE |
| 20 | `updateAsciiStream(String,InputStream,long)` | 1045 | A | 표준 | SFNSE |
| 21 | `updateBinaryStream(int,InputStream)` | 1050 | A | 표준 | SFNSE |
| 22 | `updateBinaryStream(String,InputStream)` | 1055 | A | 표준 | SFNSE |
| 23 | `updateBinaryStream(int,InputStream,long)` | 1061 | A | 표준 | SFNSE |
| 24 | `updateBinaryStream(String,InputStream,long)` | 1067 | A | 표준 | SFNSE |
| 25 | `updateBlob(int,InputStream)` | 1072 | A | 표준 | SFNSE |
| 26 | `updateBlob(String,InputStream)` | 1077 | A | 표준 | SFNSE |
| 27 | `updateBlob(int,InputStream,long)` | 1083 | A | 표준 | SFNSE |
| 28 | `updateBlob(String,InputStream,long)` | 1089 | A | 표준 | SFNSE |
| 29 | `updateCharacterStream(int,Reader)` | 1094 | A | 표준 | SFNSE |
| 30 | `updateCharacterStream(String,Reader)` | 1099 | A | 표준 | SFNSE |
| 31 | `updateCharacterStream(int,Reader,long)` | 1104 | A | 표준 | SFNSE |
| 32 | `updateCharacterStream(String,Reader,long)` | 1110 | A | 표준 | SFNSE |
| 33 | `updateClob(int,Reader)` | 1115 | A | 표준 | SFNSE |
| 34 | `updateClob(String,Reader)` | 1120 | A | 표준 | SFNSE |
| 35 | `updateClob(int,Reader,long)` | 1125 | A | 표준 | SFNSE |
| 36 | `updateClob(String,Reader,long)` | 1130 | A | 표준 | SFNSE |
| 37 | `updateNCharacterStream(int,Reader)` | 1135 | A | 표준 | SFNSE |
| 38 | `updateNCharacterStream(String,Reader)` | 1140 | A | 표준 | SFNSE |
| 39 | `updateNCharacterStream(int,Reader,long)` | 1145 | A | 표준 | SFNSE |
| 40 | `updateNCharacterStream(String,Reader,long)` | 1151 | A | 표준 | SFNSE |
| 41 | `updateNClob(int,NClob)` | 1156 | A | 표준 | SFNSE |
| 42 | `updateNClob(String,NClob)` | 1161 | A | 표준 | SFNSE |
| 43 | `updateNClob(int,Reader)` | 1166 | A | 표준 | SFNSE |
| 44 | `updateNClob(String,Reader)` | 1171 | A | 표준 | SFNSE |
| 45 | `updateNClob(int,Reader,long)` | 1176 | A | 표준 | SFNSE |
| 46 | `updateNClob(String,Reader,long)` | 1181 | A | 표준 | SFNSE |
| 47 | `updateNString(int,String)` | 1186 | A | 표준 | SFNSE |
| 48 | `updateNString(String,String)` | 1191 | A | 표준 | SFNSE |
| 49 | `updateRowId(int,RowId)` | 1196 | A | 표준 | SFNSE |
| 50 | `updateRowId(String,RowId)` | 1201 | A | 표준 | SFNSE |
| 51 | `updateSQLXML(int,SQLXML)` | 1206 | A | 표준 | SFNSE |
| 52 | `updateSQLXML(String,SQLXML)` | 1211 | A | 표준 | SFNSE |
| 53 | `isWrapperFor(Class)` | 1216 | A | wrapper(부적합) | `return iface.isInstance(this);` |
| 54 | `unwrap(Class)` | 1221 | A | wrapper(부적합) | `iface.cast(this)` / plain SQLException |
| 55 | `getObject(int,Class<T>)` | 1226 | A | 표준 | SFNSE |
| 56 | `getObject(String,Class<T>)` | 1231 | A | 표준 | SFNSE |

#### CUBRIDResultSetMetaData.java (2건, 특수 2)
| # | 메서드 | 라인 | 현재 | 분류 | 개선안 |
|---|--------|-----|:---:|------|--------|
| 1 | `isWrapperFor(Class)` | 784 | A | wrapper(부적합) | `return iface.isInstance(this);` |
| 2 | `unwrap(Class)` | 789 | A | wrapper(부적합) | `iface.cast(this)` / plain SQLException |

#### CUBRIDDatabaseMetaData.java (12건, 특수 2)
| # | 메서드 | 라인 | 현재 | 분류 | 개선안 |
|---|--------|-----|:---:|------|--------|
| 1 | `getUDTs(String,String,String,int[])` | 2550 | B | 표준 | SFNSE |
| 2 | `autoCommitFailureClosesAllResultSets()` | 2880 | A | 표준 | SFNSE |
| 3 | `getClientInfoProperties()` | 2885 | A | 표준 | SFNSE |
| 4 | `getFunctionColumns(String,String,String,String)` | 2895 | A | 표준 | SFNSE |
| 5 | `getFunctions(String,String,String)` | 2901 | A | 표준 | SFNSE |
| 6 | `getRowIdLifetime()` | 2906 | A | 표준 | SFNSE |
| 7 | `getSchemas(String,String)` | 2911 | A | 표준 | SFNSE |
| 8 | `supportsStoredFunctionsUsingCallSyntax()` | 2916 | A | 표준 | SFNSE |
| 9 | `isWrapperFor(Class)` | 2921 | A | wrapper(부적합) | `return iface.isInstance(this);` |
| 10 | `unwrap(Class)` | 2926 | A | wrapper(부적합) | `iface.cast(this)` / plain SQLException |
| 11 | `getPseudoColumns(String,String,String,String)` | 2933 | A | 표준 | SFNSE |
| 12 | `generatedKeyAlwaysReturned()` | 2938 | A | 표준 | SFNSE |

#### CUBRIDBlob.java (3건)
| # | 메서드 | 라인 | 현재 | 분류 | 개선안 |
|---|--------|-----|:---:|------|--------|
| 1 | `position(byte[],long)` | 180 | A | 표준 | SFNSE |
| 2 | `position(Blob,long)` | 184 | A | 표준 | SFNSE |
| 3 | `truncate(long)` | 257 | A | 표준 | SFNSE |

#### CUBRIDClob.java (3건)
| # | 메서드 | 라인 | 현재 | 분류 | 개선안 |
|---|--------|-----|:---:|------|--------|
| 1 | `position(String,long)` | 186 | A | 표준 | SFNSE |
| 2 | `position(Clob,long)` | 190 | A | 표준 | SFNSE |
| 3 | `truncate(long)` | 300 | A | 표준 | SFNSE |

#### DataSource / Driver 계열 (6건, 전부 특수)
| 파일 | 메서드 | 라인 | 현재 | 분류 | 개선안 |
|------|--------|-----|:---:|------|--------|
| CUBRIDDriver | `getParentLogger()` | 343 | C(raw버그) | parent-logger | SFNSE + `throws SFNSE` 추가 |
| CUBRIDDataSource | `isWrapperFor(Class)` | 133 | A | wrapper(부적합) | 구현 |
| CUBRIDDataSource | `unwrap(Class)` | 138 | A | wrapper(부적합) | 구현 |
| CUBRIDDataSource | `getParentLogger()` | 143 | C(raw버그) | parent-logger | SFNSE + `throws SFNSE` 추가 |
| CUBRIDConnectionPoolDataSource | `getParentLogger()` | 121 | C(raw버그) | parent-logger | SFNSE + `throws SFNSE` 추가 |
| CUBRIDXADataSource | `getParentLogger()` | 92 | C(raw버그) | parent-logger | SFNSE + `throws SFNSE` 추가 |

#### CUBRIDPooledConnection.java (2건, 전부 특수)
| # | 메서드 | 라인 | 현재 | 분류 | 개선안 |
|---|--------|-----|:---:|------|--------|
| 1 | `addStatementEventListener(StatementEventListener)` | 150 | C(raw) | listener(불가) | no-op 구현 (SFNSE 불가) |
| 2 | `removeStatementEventListener(StatementEventListener)` | 155 | C(raw) | listener(불가) | no-op 구현 (SFNSE 불가) |

## 결론
296건 전부가 개선 대상이며, 그 중 278건은 표준 패턴대로 `SQLFeatureNotSupportedException`으로 안전하게 전환할 수 있다. 나머지 18건은 JDBC 인터페이스 계약(throws 절) 때문에 SFNSE로 바꾸면 안 되거나 컴파일이 불가하며, wrapper는 실제 구현으로, client-info는 `SQLClientInfoException`으로, statement-event-listener는 no-op로 처리해야 한다. 또한 form C 7건은 unchecked 예외가 유출되는 별도의 실제 버그로 함께 정리하는 것이 좋다.

## 다음 단계
- 표준 278건: `throw new SQLFeatureNotSupportedException();`로 일괄 치환 + 파일별 `import java.sql.SQLFeatureNotSupportedException;` 추가(선택: 진단성 위해 `new SQLFeatureNotSupportedException("<method> not supported", "0A000")`).
- parent-logger 4건: SFNSE로 변경하되 시그니처에 `throws SQLFeatureNotSupportedException` 추가(현재 raw throw 버그 해소).
- wrapper 14건: SFNSE로 얼버무리지 말고 실제 구현(`isWrapperFor→isInstance`, `unwrap→cast`, 실패 시 plain SQLException).
- client-info 2건: `SQLClientInfoException(failures, cause)`로 정리(SFNSE 컴파일 불가).
- listener 2건: no-op 구현(throws 절이 없어 checked 예외 불가).
- 이슈화 여부: 로드맵의 JDBC 4.x 확장 트랙과 연계해 별도 JIRA 이슈로 등록 검토.

## 참고
- 대상 소스: `cubrid-jdbc/src/jdbc/cubrid/jdbc/driver/*.java` (15개 파일)
- `java.sql.SQLFeatureNotSupportedException` (JDBC 4.0, Java 6+): `SQLNonTransientException` → `SQLException` 상속, SQLState 관례 `0A000`
- `java.sql.Wrapper`, `javax.sql.CommonDataSource#getParentLogger`, `javax.sql.PooledConnection#add/removeStatementEventListener`, `java.sql.Connection#setClientInfo`의 계약(throws 절)

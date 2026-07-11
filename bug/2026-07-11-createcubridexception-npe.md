# CUBRID JDBC 드라이버 createCUBRIDException NullPointerException (getGeneratedKeys 등 con 없이 생성되는 ResultSet의 널-미방어 결함)

- 분류: bug
- 날짜: 2026-07-11
- 관련: Hibernate ORM CUBRID CI 잔여 실패 분석, JDBC 드라이버 수정 트랙

## 목적
Hibernate ORM을 CUBRID(10.2~11.4)에서 돌릴 때 가장 많이 발생하는 비결정적 테스트 실패인 `createCUBRIDException` NullPointerException의 근본 원인을 드라이버 소스로 규명하고, 이것이 방언(Dialect)이나 테스트로는 고칠 수 없는 JDBC 드라이버 결함임을 확정한다.

> 정정 이력(2026-07-11): 초판은 원인을 "`close()`가 `con`을 null로 만든다"로 봤으나, 드라이버 소스를 직접 재검증한 결과 **이는 틀렸다**. `close()`는 `con`을 건드리지 않는다. 실제 원인은 `getGeneratedKeys()` 등이 사용하는 `CUBRIDResultSet(UStatement)` 생성자가 `con`(및 일부 경로에서 `u_stmt`)을 애초에 null로 만들고, 접근자/가드가 이를 방어 없이 역참조하는 것이다. 아래 본문은 재검증된 내용으로 교체했다.

## 배경
CUBRID CI 5버전 매트릭스에서 잔여 실패의 최대 군집이 이 NPE다. 증상은 항상 아래 형태다.

```
java.lang.NullPointerException: Cannot invoke
  "cubrid.jdbc.driver.CUBRIDConnection.createCUBRIDException(int, ...)" because <참조>가 null
```

특징은 (1) 버전/실행마다 실패 대상이 요동치고(같은 코드인데 어떤 버전은 극소수, 다른 버전은 수십 개), (2) 서로 무관한 테스트(merge, persist, lock 등)가 무작위로 걸린다는 것이다. 로직 버그라면 항상 같은 지점이 터져야 하는데 그렇지 않아, 특정 타이밍 조건에서만 발현하는 결함으로 의심되었다. (재검증 결과 이 "비결정성"의 실체는 §발견에서 규명한다.)

## 범위 / 방법
- CI JUnit 결과에서 실패 시그니처와 버전별 분포를 수집.
- CUBRID JDBC 드라이버 소스(`cubrid.jdbc.driver` 패키지: `CUBRIDStatement`, `CUBRIDResultSet`, `CUBRIDResultSetWithoutQuery`, `CUBRIDConnection`)를 정독해 `createCUBRIDException`의 정의, `con`/`u_stmt` 필드의 대입 지점(생성자·close·accessor), 호출 경로를 라인 단위로 추적·대조.

## 발견 / 관찰
1. `createCUBRIDException(...)`는 `CUBRIDConnection`의 **인스턴스** 메서드로, 오류 코드를 표준 `SQLException`(CUBRID 계열 예외)으로 변환한다. 드라이버 전반은 `con.createCUBRIDException(...)` 형태로 예외를 만든다(호출 지점 150여 곳). 인스턴스 메서드지만 예외 객체 자체는 `new CUBRIDException(errCode)`로 **연결 없이** 만들 수 있고(메시지는 코드에서 정적 조회), `con`은 부가적 로깅(`u_con.logException`)에만 쓰인다.

2. **핵심 정정 — `con`은 `close()`가 아니라 "생성자"에서 null이 된다.** `CUBRIDResultSet.close()`(`CUBRIDResultSet.java:282`)가 null로 만드는 필드는 `stmt`, `u_stmt`, `column_info`, `col_name_to_index`, `error`이며 **`con`은 건드리지 않는다**. `CUBRIDResultSet`에서 `con = null`은 오직 생성자 `CUBRIDResultSet(UStatement s)`(`:159`)뿐이다. 이 생성자를 쓰는 ResultSet은 처음부터 `con`이 없다:
   - **auto-generated-keys**: `getGeneratedKeys()`가 `new CUBRIDResultSet(null)`(`CUBRIDStatement.java:599`) 또는 `new CUBRIDResultSet(auto_generatedkeys_stmt)`(`:968`)로 생성 — 둘 다 `con = null`.
   - stored procedure OUT 파라미터: `CUBRIDOutResultSet`(`super(null)`).
   - OID: `CUBRIDOIDImpl.java:86`.

   반면 **일반 쿼리 ResultSet**은 `new CUBRIDResultSet(con, this, ...)`(`CUBRIDStatement.java:341`)로 **진짜 `con`을 받는다.** 그래서 일반 ResultSet은 닫은 뒤 접근해도 `con`이 살아 있어 아래 가드가 정상적으로 `result_set_closed` SQLException을 던진다(NPE 아님).

   ```java
   // CUBRIDResultSet.java:1628
   private void checkIsOpen() throws SQLException {
       if (is_closed) {
           throw con.createCUBRIDException(CUBRIDJDBCErrorCode.result_set_closed, null); // con이 null이면 여기서 NPE
       }
   }
   ```

3. **CI 스택으로 확정된 경로 = getGeneratedKeys.** 실패 스택 최상단 드라이버 프레임은 `CUBRIDResultSet.findColumn`이고 Hibernate `MutationExecutorSingleNonBatched.performNonBatchedOperations`(**INSERT + 생성값 조회**)에서 호출된다. "INSERT + 생성값 조회"는 곧 `getGeneratedKeys()`이며, 그 반환 ResultSet은 위 2번대로 `con = null`이다. 즉 **CI 증거 자체가 "con은 생성자에서 null"이라는 재검증 결과를 뒷받침한다** — "close()가 null로 만들어서"가 아니다.

4. **관측된 두 "얼굴"은 모두 getGeneratedKeys의 con-없는(그리고 u_stmt-없는) ResultSet에서 나온다.** `close()`가 필드를 null로 만드는 것은 red herring이다(정상 ResultSet은 `con`이 살아 있어 가드가 깔끔한 SQLException을 던지므로 NPE가 나지 않는다):
   - **얼굴 A — `this.con` null → `findColumn`/`getXxx` → `checkIsOpen()` → `con.createCUBRIDException`**: generated-keys ResultSet(`con=null`)이 **닫힌 상태**(`is_closed=true`)에서 접근될 때. 가장 흔한 시그니처.
   - **얼굴 B — `this.u_stmt` null → `getMetaData` → `u_stmt.getColumnInfo()`**: `getGeneratedKeys()`가 캡처된 키가 없을 때 반환하는 `new CUBRIDResultSet(null)`은 `u_stmt=null`, `is_closed=false`, `con=null`이다. `getMetaData()`(`:680`)는 `checkIsOpen()`을 부르지만 `is_closed=false`라 **통과**하고, 곧이어 `u_stmt.getColumnInfo()`(`:684`)에서 NPE. (일부는 기대 예외 `IllegalStateException` 대신 이 NPE가 올라와 테스트가 깨진다.) 이 얼굴은 "닫힘"과 무관하다.

5. **비결정성의 실체.** 드라이버 결함 자체는 **결정적**이다(generated-keys ResultSet은 항상 `con=null` → 닫힌 뒤 접근하면 항상 NPE). 비결정적인 것은 "Hibernate가 언제 그 ResultSet을 닫힌 뒤/키 없이 접근하느냐"로, INSERT+생성값 경로의 정리 순서·풀 반환·선행 오류 타이밍에 의존한다. 그래서 버전/실행마다 걸리는 테스트가 달라진다. 초판이 의심한 "GC/finalizer" 타이밍보다, **ResultSet 종류(generated-keys 여부)와 그 수명 주기**가 본질이다.

6. **선례: 같은 코드베이스에 이미 올바른 구현이 있다.** 자매 클래스 `CUBRIDResultSetWithoutQuery.checkIsOpen()`(`:933`)은 `throw new CUBRIDException(CUBRIDJDBCErrorCode.result_set_closed)`로 **연결 없이** 예외를 만들어 NPE에 면역이다. 또 `CUBRIDStatement`/`CUBRIDPreparedStatement`/`CUBRIDDatabaseMetaData`의 `checkIsOpen()`은 `if (con != null) … else new CUBRIDException(…)` 방어 패턴을 이미 갖고 있다. `CUBRIDResultSet`(얼굴 A)과 `CUBRIDShardMetaData`의 `checkIsOpen`만 이 방어가 빠져 있다.

## 결론
- 이 NPE는 CUBRID JDBC 드라이버 내부 결함이다. 근본 원인은 **`getGeneratedKeys()` 등이 쓰는 `CUBRIDResultSet(UStatement)` 생성자가 `con`(및 `new CUBRIDResultSet(null)` 경로에선 `u_stmt`)을 null로 둔 채**, 접근자·가드가 이를 방어 없이 역참조하는 것이다. `close()`가 필드를 null로 만드는 것은 이 NPE의 원인이 아니다(정상 ResultSet은 `con`이 살아 있어 깔끔한 SQLException을 던진다).
- **방언(CUBRIDDialect)이나 Hibernate 테스트로는 고칠 수 없다.** 비결정적으로 보이지만 특정 테스트를 skip해도 generated-keys를 쓰는 다른 테스트로 옮겨가므로 skip 대상도 아니다.
- Hibernate CUBRID CI에서 다른 결정적 실패(엔진 한계, core 렌더링 등)를 모두 처분한 뒤에도 남는, **안정적 그린 CI의 사실상 유일한 걸림돌**이다.

## 재현 (최소 케이스)
Hibernate 없이 드라이버 단독으로 재현된다. 핵심은 **`getGeneratedKeys()`가 반환하는 con-없는 ResultSet**을 건드리는 것이다(실행에는 기동 중인 CUBRID 필요). 초판의 `select 1` → 일반 ResultSet 케이스는 **재현되지 않는다**(일반 ResultSet은 `con`이 살아 있어 닫힌 뒤 접근해도 `SQLException`이 뜬다).

```java
Class.forName("cubrid.jdbc.driver.CUBRIDDriver");
try (Connection c = DriverManager.getConnection("jdbc:cubrid:localhost:33000:demodb:::", "public", "")) {

    // 얼굴 A: 닫힌 generated-keys ResultSet (con=null) 접근 -> checkIsOpen -> con.createCUBRIDException NPE
    PreparedStatement ps = c.prepareStatement(
        "insert into t(v) values (1)", Statement.RETURN_GENERATED_KEYS);
    ps.executeUpdate();
    ResultSet gk = ps.getGeneratedKeys();  // new CUBRIDResultSet(...): con = null
    gk.close();                            // is_closed = true (con은 여전히 null)
    try {
        gk.getString(1);                   // 또는 findColumn("id")
    } catch (SQLException e) {
        System.out.println("기대(정상): " + e.getMessage());   // "...closed ResultSet."
    } catch (NullPointerException npe) {
        System.out.println("버그 A: " + npe.getMessage());      // createCUBRIDException(...) because this.con is null
    }

    // 얼굴 B: 캡처된 키 없는 getGeneratedKeys -> new CUBRIDResultSet(null) (u_stmt=null, is_closed=false)
    Statement st = c.createStatement();
    st.executeUpdate("insert into t(v) values (2)");  // RETURN_GENERATED_KEYS 없음
    ResultSet gk2 = st.getGeneratedKeys();            // new CUBRIDResultSet(null): u_stmt = null
    try {
        gk2.getMetaData();                             // checkIsOpen 통과(is_closed=false) 후 u_stmt.getColumnInfo()
    } catch (SQLException e) {
        System.out.println("기대(정상): " + e.getMessage());
    } catch (NullPointerException npe) {
        System.out.println("버그 B: " + npe.getMessage());      // getColumnInfo() because this.u_stmt is null
    }
}
```

- **기대(정상 드라이버)**: 두 경우 모두 명확한 `SQLException`.
- **실제(버그)**: `NullPointerException`(예외를 만들거나 메타데이터를 읽으려다 `con`/`u_stmt`가 null이라 실패).
- **Hibernate에서 "랜덤"으로 보이는 이유**: use-after-close/키-없음 상황이 INSERT+생성값 경로에서 정리 순서·풀 반환·선행 오류 타이밍에 따라 생겼다 안 생겼다 하기 때문. **드라이버 결함은 결정적**(generated-keys ResultSet → 항상 con/u_stmt null)이고, **비결정적인 것은 그 접근이 언제 일어나느냐**다.

## 다음 단계
드라이버 수정 이슈로 등록(JDBC 드라이버 트랙). 초판이 나열한 3개 후보를 재검증된 원인 기준으로 평가하면:

1. **예외 변환 경로에 널 가드 — 채택(핵심).** 가드가 `con`이 null이면 경유하지 않고 표준 "closed" `SQLException`을 직접 생성. 얼굴 A를 정확히 해결하며 코드베이스에 이미 선례 4곳(§발견 6). 대상: `CUBRIDResultSet.checkIsOpen`(+ 같은 파일 `checkIsUpdatable`/`checkIsSensitive`), `CUBRIDShardMetaData.checkIsOpen`. `CUBRIDResultSetWithoutQuery`와 동일하게 `new CUBRIDException(code)` 형태.
   - 단, **얼굴 B(u_stmt null)는 이것만으로 안 고쳐진다.** `getMetaData`는 `checkIsOpen`을 통과한 뒤 `u_stmt`를 역참조하기 때문. 얼굴 B에는 (a) `getGeneratedKeys()`가 키 없을 때 `u_stmt=null` 대신 **빈 결과셋(빈 column_info)** 을 만들도록 생성 경로를 고치거나, (b) `u_stmt`를 역참조하는 접근자에 널 가드를 추가하는 별도 조치가 필요하다.
2. **`close()`에서 `con`을 유지 — 기각.** 전제가 틀렸다. `close()`는 `con`을 null로 만들지 않으며, 문제의 null은 *생성자*에서 온다. 이 방안은 가장 흔한 시그니처(얼굴 A)를 전혀 고치지 못한다.
3. **닫힌 객체 접근 조기 차단 — 기각(1로 수렴).** 진입점 감지는 이미 `checkIsOpen()`으로 존재하고(모든 getter 첫 줄), 이 경로엔 `catch(NullPointerException)` 변환도 없다. 실질을 벗기면 1과 같아지고, 얼굴 B도 못 고친다.

**권장 = 1 + 얼굴 B 처리.** 가드를 연결 없이 만들고(얼굴 A), 아울러 키-없는 generated-keys 결과셋의 `u_stmt=null` 접근(얼굴 B)을 처리한다. 회귀 테스트는 위 재현(§)의 두 경로를 JUnit으로 고정: 수정 전 NPE로 실패 → 수정 후 `SQLException`으로 통과. 드라이버 수정 후 CUBRID CI 재검증.

## 참고
- 드라이버 소스(재검증, 라인 확정):
  - `CUBRIDResultSet#checkIsOpen`(`:1628`, `if (is_closed) throw con.createCUBRIDException(result_set_closed, null)`); `findColumn`/`getXxx`가 진입 시 `checkIsOpen()` 호출.
  - `CUBRIDResultSet#close`(`:282`)는 `stmt`/`u_stmt`/`column_info`/`col_name_to_index`/`error`를 null로 하지만 **`con`은 아님**.
  - `CUBRIDResultSet(UStatement)`(`:158-187`)가 `con=null`(`:159`), `u_stmt=s`(`:161`, `s`가 null이면 `u_stmt`도 null); `getMetaData`(`:680`)는 `checkIsOpen` 후 `u_stmt.getColumnInfo()`(`:684`).
  - `CUBRIDStatement#getGeneratedKeys`(`:595`)와 생성 지점(`:599`, `:968`); 일반 쿼리 결과셋은 `new CUBRIDResultSet(con, ...)`(`:341`).
  - 올바른 선례: `CUBRIDResultSetWithoutQuery#checkIsOpen`(`:933`, 연결 없이 `new CUBRIDException(...)`); 방어형 `CUBRIDStatement`/`CUBRIDPreparedStatement`/`CUBRIDDatabaseMetaData#checkIsOpen`.
  - 예외 생성기 `CUBRIDConnection#createCUBRIDException`(`:984-1002`, 3개 오버로드; 예외 객체는 연결 없이 생성 가능, `con`은 `u_con.logException` 부가 로깅에만 사용).
- CI 실패 스택(확정, 둘 다 getGeneratedKeys의 con/u_stmt-없는 결과셋):
  - `... createCUBRIDException(int, Throwable) because "this.con" is null` at `CUBRIDResultSet.findColumn` <- Hibernate `MutationExecutorSingleNonBatched.performNonBatchedOperations`(INSERT+생성값 = getGeneratedKeys).
  - `... UStatement.getColumnInfo() because "this.u_stmt" is null` at `CUBRIDResultSet.getMetaData`(MultiPathCircleCascade* 계열; 일부는 기대 예외 `IllegalStateException` 대신 이 NPE).
- cubrid-manual은 엔진 SQL/함수/예약어를 다루므로 드라이버 내부 동작은 매뉴얼이 아니라 드라이버 소스로 확정했다.
- CI 실패 수치와 전체 처분 내역은 내부 통합 문서에 정리.

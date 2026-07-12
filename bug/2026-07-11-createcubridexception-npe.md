# CUBRID JDBC createCUBRIDException NullPointerException — con 없이 생성되는 ResultSet(getGeneratedKeys/OUT/OID)의 초기화 미비 결함

- 분류: bug
- 날짜: 2026-07-11
- 관련: Hibernate ORM CUBRID CI 잔여 실패 분석, JDBC 드라이버 수정 트랙

## 목적
Hibernate ORM을 CUBRID(10.2~11.4)에서 돌릴 때 가장 많이 발생하는 비결정적 테스트 실패인 `createCUBRIDException` NullPointerException의 근본 원인을 드라이버 소스로 규명하고, 방언(Dialect)·테스트로는 고칠 수 없는 JDBC 드라이버 결함임을 확정하며, 근본 해법을 제시한다.

> 정정 이력
> - **2026-07-11**: 초판은 원인을 "`close()`가 `con`을 null로 만든다"로 봤으나, 드라이버 소스 재검증 결과 **틀렸다**. `close()`는 `con`을 건드리지 않는다. 실제로 `con`은 `CUBRIDResultSet(UStatement)` 생성자에서 null이 된다.
> - **2026-07-12**: 추가 검증으로 세 가지를 더 정정·보강. (1) **방아쇠 정정** — NPE는 "닫힌 뒤 접근(use-after-close)"이 아니라 **`con`을 역참조하는 행/컬럼 검증 분기**에서 발생한다(대표적으로 `next()` 전 getter). generated-keys/OID의 base `close()`는 오히려 조용히 no-op이 된다. (2) **A/B는 한 뿌리** — 두 증상 모두 1-arg 생성자의 초기화 미비에서 나온다. (3) **근본 해법** — 생성자에 연결을 주입한다.

## 배경
CUBRID CI 5버전 매트릭스에서 잔여 실패의 최대 군집이 이 NPE다. 증상은 항상 아래 형태다.

```
java.lang.NullPointerException: Cannot invoke
  "cubrid.jdbc.driver.CUBRIDConnection.createCUBRIDException(int, ...)" because <참조>가 null
```

특징은 (1) 버전/실행마다 실패 대상이 요동치고, (2) 서로 무관한 테스트(merge, persist, lock 등)가 무작위로 걸린다는 것이다. 로직 버그라면 항상 같은 지점이 터져야 하는데 그렇지 않아, 특정 타이밍 조건에서만 발현하는 결함으로 의심되었다(이 "비결정성"의 실체는 §발견 5에서 규명).

## 범위 / 방법
- CI JUnit 결과에서 실패 시그니처와 버전별 분포를 수집.
- CUBRID JDBC 드라이버 소스를 라인 단위로 추적·대조: `CUBRIDResultSet`의 두 생성자와 `con`/`u_stmt` 대입 지점, `con`을 역참조하는 전 지점(가드·`synchronized(con)`·오류 분기), `close()`/`next()`의 실제 동작, 그리고 `getGeneratedKeys`/`CUBRIDOutResultSet`/`CUBRIDOIDImpl`의 생성 경로.

## 발견 / 관찰

### 1. 근본 원인: 1-arg 생성자 `CUBRIDResultSet(UStatement)`의 초기화 미비
`CUBRIDResultSet`에는 두 생성자가 있다.
- **5-arg** `CUBRIDResultSet(CUBRIDConnection c, CUBRIDStatement s, …)`(`:108`): `con = c`(`:111`), `u_stmt = s.u_stmt` — **정상 초기화**. 일반 SELECT 결과셋(`CUBRIDStatement:341`)이 사용.
- **1-arg** `CUBRIDResultSet(UStatement s)`(`:158`): `con = null`(`:159`), `u_stmt = s`(`:161`, `s`가 null이면 `u_stmt`도 null). **초기화 미비.**

`con`은 이 클래스에서 딱 두 곳(`:111 con=c`, `:159 con=null`)에서만 대입된다. 즉 **`con == null`은 1-arg 생성자로 만든 결과셋에서만** 발생하고, 그걸 쓰는 곳은:
- **generated-keys**: `getGeneratedKeys()`가 `new CUBRIDResultSet(null)`(`CUBRIDStatement:599`) 또는 `new CUBRIDResultSet(auto_generatedkeys_stmt)`(`:968`)
- **OUT 파라미터**: `CUBRIDOutResultSet`가 `super(null)`(`CUBRIDOutResultSet:50`)
- **OID**: `CUBRIDOIDImpl.getValues()`가 `new CUBRIDResultSet(u_stmt)`(`CUBRIDOIDImpl:86`)

### 2. 두 증상(얼굴 A·B)은 별개가 아니라 "한 뿌리"
1-arg 생성자가 필드를 덜 초기화한다는 **하나의 근본**에서, 어느 필드가 null이냐에 따라 두 얼굴로 나타난다.
- **얼굴 A — `con == null`**: 1-arg 생성자의 **모든** 사용처(위 3종 전부). `con`을 역참조하는 지점에서 NPE.
- **얼굴 B — `u_stmt == null`**: **오직 `new CUBRIDResultSet(null)`(`CUBRIDStatement:599`, 키 없는 getGeneratedKeys)** 에서만. `u_stmt`를 역참조하는 접근자(`getMetaData` 등)에서 NPE.

차이는 "고치는 법"뿐이다(§해법): `con`은 어느 지점에서든 실제 연결이 손에 있어 **채워 넣을 수 있고**, `u_stmt`는 `:599`에선 애초에 가리킬 statement가 없어 **빈 결과셋으로 별도 처리**가 필요하다.

### 3. 정확한 방아쇠: "닫힌 뒤 접근"이 아니라 "con 역참조 분기 도달"
초판(및 CI 최초 관찰)은 방아쇠를 use-after-close로 봤지만 **부정확하다.**
- `checkIsOpen()`(`:1628`)은 `is_closed==true`일 때만 `con`을 역참조한다. 그런데 generated-keys/OID의 **base `close()`(`:282`)는 첫 줄 `synchronized (con)`(`:284`)에서 `con==null`로 NPE → `catch(NullPointerException){}`(`:314`)가 삼킴 → `is_closed=true`(`:290`)에 도달 못 하고 조용히 no-op**이 된다. 따라서 이 결과셋들은 애초에 "닫힌 상태"가 되지 않고, `checkIsOpen` 경로로는 NPE가 나지 않는다.
- 실제 NPE는 **`con`을 (조건 없이) 역참조하는 행/컬럼 검증·오류 분기**에서 난다:
  - `next()` 전 getter, 또는 마지막 행 이후 getter → `beforeGetValue`(`:1676`) → `checkRowIsValidForGet`(`current_row==-1` 또는 `==number_of_rows`) → `con.createCUBRIDException`(`:1654`)
  - 없는 컬럼명 `findColumn` → `con.createCUBRIDException(invalid_column_name)`(`:713`)
  - 잘못된 컬럼 인덱스 → `checkColumnIsValid`(`:1666`); update 계열(이 RS는 읽기 전용) → `checkIsUpdatable`(`:1636`)
  - 서버/변환 오류 → `beforeGetValue`(`:1689`)·`checkGetXXXError`(`:1702`)의 오류 분기
- **예외: OUT 파라미터만 use-after-close가 통한다.** `CUBRIDOutResultSet`는 `close()`(`:71`)를 override해 `synchronized(con)` 없이 `is_closed=true`를 직접 세우므로, 닫은 뒤 `checkIsOpen`이 `con`을 역참조해 NPE가 난다.

### 4. 정상 순회는 왜 살아남나
`next()`(`:193`)도 `con==null`이라 `synchronized (con)`(`:198`)에서 NPE가 나지만, **그 NPE를 catch(`:220`)해 `synchronized(this)`로 재시도**한다. 유효 행에 올라간 뒤 유효 컬럼을 정상적으로 읽으면 `con`은 오류 분기에서만 쓰이므로 건드리지 않는다. 즉 **`while(next()){ 유효 getter }` 정상 경로는 NPE가 안 난다** — `con==null`은 클래스 전반에 깔린 문제이고, 정상 경로는 "con을 건드리는 분기를 우연히 안 지나가서" 통과하는 것에 가깝다.

### 5. 비결정성의 실체
드라이버 결함은 **결정적**이다(1-arg 결과셋은 항상 `con==null`). 비결정적인 것은 "Hibernate가 언제 그 결과셋의 con-역참조 분기(next 전 접근·없는 컬럼 조회 등)를 밟느냐"로, INSERT+생성값 경로의 정리 순서·풀 반환·선행 오류 타이밍에 달렸다. 그래서 버전/실행마다 걸리는 테스트가 달라진다.

### 6. CI 스택 재해석 & 올바른 선례
- `... createCUBRIDException(int, Throwable) because "this.con" is null` at `CUBRIDResultSet.findColumn` ← INSERT+생성값(=getGeneratedKeys). "닫힘"이 아니라 **`findColumn`의 없는-컬럼명 분기(`:713`)** 로 보는 게 코드와 일치(is_closed는 false).
- `... getColumnInfo() because "this.u_stmt" is null` at `CUBRIDResultSet.getMetaData` ← 키 없는 getGeneratedKeys(`:599`, `u_stmt=null`) → `getMetaData`(`:680`)가 `checkIsOpen` 통과 후 `u_stmt.getColumnInfo()`(`:684`).
- **선례**: 자매 `CUBRIDResultSetWithoutQuery.checkIsOpen()`(`:933`)은 `new CUBRIDException(result_set_closed)`로 연결 없이 예외를 만들어 면역. `CUBRIDStatement`/`CUBRIDPreparedStatement`/`CUBRIDDatabaseMetaData`의 `checkIsOpen`도 `if (con != null) … else …` 방어형.

## 결론
- 이 NPE는 CUBRID JDBC 드라이버 내부 결함이며, 근본 원인은 **1-arg 생성자 `CUBRIDResultSet(UStatement)`가 `con`(및 `:599`에선 `u_stmt`)을 채우지 않은 채 결과셋을 만들고**, 공유 접근자·가드가 이를 방어 없이 역참조하는 것이다. 얼굴 A·B는 이 한 뿌리의 두 증상이다.
- `close()`가 필드를 null로 만드는 것은 원인이 아니며(오히려 con=null에선 `close()`가 조용히 no-op이 되어 `u_stmt`/서버 커서 누수까지 유발), 방아쇠는 "con 역참조 분기 도달"이다.
- **방언·테스트로는 고칠 수 없다.** 비결정적으로 보이지만 generated-keys를 쓰는 다른 테스트로 옮겨가므로 skip 대상도 아니다. 안정적 그린 CI의 사실상 유일한 걸림돌.

## 재현 (최소 케이스)
기동 중인 CUBRID 필요. 초판의 `select 1`(일반 ResultSet)은 **재현되지 않는다**(진짜 con을 받으므로 정상 SQLException). 핵심은 **`getGeneratedKeys()` 등 1-arg 생성 결과셋의 con-역참조 분기**를 밟는 것.

```java
Class.forName("cubrid.jdbc.driver.CUBRIDDriver");
try (Connection c = DriverManager.getConnection(
        "jdbc:cubrid:localhost:33000:demodb:::", "public", "")) {
    Statement ddl = c.createStatement();
    ddl.executeUpdate("CREATE TABLE t_keys (id INT AUTO_INCREMENT, v VARCHAR(20), PRIMARY KEY(id))");

    // 얼굴 A (con NPE): generated-keys, next() 전 getter
    PreparedStatement ps = c.prepareStatement(
            "INSERT INTO t_keys(v) VALUES ('a')", Statement.RETURN_GENERATED_KEYS);
    ps.executeUpdate();
    ResultSet gk = ps.getGeneratedKeys();     // con=null
    try { gk.getInt(1); }                     // checkRowIsValidForGet(-1) -> con.createCUBRIDException -> NPE
    catch (SQLException e)          { System.out.println("정상: " + e.getMessage()); }
    catch (NullPointerException e)  { System.out.println("버그 con NPE: " + e); }
    // 참고: gk.close()는 con=null이라 조용히 no-op → "닫은 뒤 접근"은 방아쇠가 아님.
    //       while (gk.next()) gk.getInt(1);  // 정상 경로는 NPE 없음.

    // 얼굴 B (u_stmt NPE): 키 없는 getGeneratedKeys -> new CUBRIDResultSet(null)
    Statement st = c.createStatement();
    st.executeUpdate("INSERT INTO t_keys(v) VALUES ('b')");   // RETURN_GENERATED_KEYS 없음
    ResultSet gk2 = st.getGeneratedKeys();    // u_stmt=null, con=null
    try { gk2.getMetaData(); }                // checkIsOpen 통과 -> u_stmt.getColumnInfo() -> NPE
    catch (SQLException e)          { System.out.println("정상: " + e.getMessage()); }
    catch (NullPointerException e)  { System.out.println("버그 u_stmt NPE: " + e); }
}
```

- OUT 파라미터(`CUBRIDOutResultSet`, 커서 반환 Java SP 필요): `rs=(ResultSet)cstmt.getObject(idx)` 후 `rs.getString(1)`(next 전) 또는 **`rs.close(); rs.next()`**(이 타입만 use-after-close 성립).
- OID(`CUBRIDOIDImpl`): 단일 테이블 `SELECT *` → `((CUBRIDResultSet)rs).getOID().getValues(new String[]{...})` 후 `getString(1)`(next 전).

## 해법
### 근본 해법 (권장): 생성자에 연결을 주입
`con`이 null인 네 지점 모두 **생성 시점에 실제 `CUBRIDConnection`이 이미 손에 있다**. `con=null`은 연결이 없어서가 아니라 1-arg 생성자가 연결을 인자로 안 받기 때문이다.

| 생성 지점 | 쓸 수 있는 연결 |
|---|---|
| `CUBRIDStatement:599 / :968` | `this.con` (`CUBRIDStatement:59`) |
| `CUBRIDOutResultSet:50` | `ucon.getCUBRIDConnection()` (`UConnection:1731`, 이미 `:54`에서 호출) |
| `CUBRIDOIDImpl:86` | `cur_con` (`CUBRIDOIDImpl:55`) |

수정: 1-arg 생성자에 `CUBRIDConnection` 파라미터를 추가하고, 각 지점이 위 연결을 넘긴다.
```java
// 현재:  public CUBRIDResultSet(UStatement s)            { con = null; … }
// 수정:  public CUBRIDResultSet(CUBRIDConnection c, UStatement s) { con = c; … }
//   CUBRIDStatement:599 -> new CUBRIDResultSet(con, null)
//   CUBRIDStatement:968 -> new CUBRIDResultSet(con, auto_generatedkeys_stmt)
//   CUBRIDOutResultSet:50 -> super(ucon.getCUBRIDConnection(), null)
//   CUBRIDOIDImpl:86 -> new CUBRIDResultSet(cur_con, u_stmt)
```
- `con`을 역참조하는 모든 지점(가드 ~13곳 + `close()`/`next()`의 `synchronized(con)` + 오류 분기)이 **가드 수정 0으로 한꺼번에 정상 동작** → NPE 대신 의도된 `SQLException`.
- **보너스**: `close()`가 끝까지 실행되어(더는 no-op 아님) `u_stmt`/서버 커서 누수도 해결.
- **영향 범위**: 1-arg 생성자 호출은 **내부 4곳뿐**(일반 5-arg 경로 무관). `public`이지만 인자가 내부 클래스 `UStatement`라 외부 앱 호출 불가 → 공개 API 무영향, 안전.
- **안전성**: `con == null`을 분기 신호로 쓰는 코드는 없음(전수 확인) → 값 채워도 회귀 없음.
- **무변경 대안은 부적합**: 생성자 안에서 `s`로부터 con을 유도하는 방법은, OUT 파라미터(`super(null)`로 s=null, u_stmt는 나중에 세팅)와 키 없는 gk(`:599`, s=null)에서 con을 못 채운다(게다가 `con`이 `private`이라 하위 클래스가 super 이후 설정 불가). 그래서 "이미 손에 있는 연결을 명시적으로 넘기는" 파라미터 방식이 유일하게 완전하다.

### 얼굴 B 마무리
`:599`(키 없는 gk)는 con을 채워도 `u_stmt=null`이 남는다 → `getGeneratedKeys()`가 `new CUBRIDResultSet(null)` 대신 **제대로 된 빈 결과셋**을 반환하도록(또는 `u_stmt` 역참조 접근자에 널 가드) 처리.

### 원안 3개와의 관계
- **Option 1(예외 경로 널 가드)** = 증상 측 방어. `con` 역참조 ~13곳 + `synchronized(con)`까지 일일이 막아야 하는 두더지 잡기. **defense-in-depth 2차선**으로는 유효(새 con-없는 경로가 생겨도 SQLException 보장).
- **Option 2(`con` 유지)** = 방향은 옳았으나 위치가 틀림. `close()`가 아니라 **생성자**에서 con을 유지(=주입)하는 것이 올바른 형태 → 이번 근본 해법이 곧 Option 2의 정확한 구현.
- **Option 3(조기 감지)** = 감지는 이미 진입점에 있어 실질적으로 Option 1로 수렴.

**권장 조합**: ① 생성자 con 주입(근본) + ② `:599` 빈 결과셋 처리(얼굴 B) + (선택) ③ 가드 connection-free 화를 defense-in-depth로. 회귀 테스트는 §재현의 네 경로(gk 얼굴 A/B, OUT, OID)를 JUnit으로 고정: 수정 전 NPE → 수정 후 `SQLException`.

## 참고
- 소스(재검증, 라인 확정): `CUBRIDResultSet` — 1-arg 생성자 `:158`(`con=null :159`), 5-arg `:108`(`con=c :111`); `checkIsOpen :1628`(`con` 역참조 `:1630`), `checkRowIsValidForGet :1652`(`:1654`), `checkColumnIsValid :1664`(`:1666`), `findColumn :708`(`:713`), `beforeGetValue :1676`(`:1689`), `checkGetXXXError :1693`(`:1702`), `checkIsUpdatable :1634`(`:1636`); `getMetaData :680`(`u_stmt.getColumnInfo :684`); `next :193`(`synchronized(con) :198`, `catch NPE :220`); `close :282`(`synchronized(con) :284`, `catch NPE :314`).
- `CUBRIDStatement` — 일반 결과셋 `:341`, `getGeneratedKeys :595`(`:599`), `MakeAutoGeneratedKeysResultSet :950`(`:968`), `con` 필드 `:59`.
- `CUBRIDOutResultSet` — `extends :38`, `super(null) :50`, `ucon :47`, `close() override :71`(`u_stmt.close() :80`); `CUBRIDOIDImpl` — `cur_con :55`, `getValues :75`(`new CUBRIDResultSet(u_stmt) :86`); `UConnection.getCUBRIDConnection :1731`; `UStatement.relatedConnection :89`.
- 선례: `CUBRIDResultSetWithoutQuery.checkIsOpen :933`; 예외 생성기 `CUBRIDConnection.createCUBRIDException :984-1002`(예외 객체는 연결 없이 생성 가능, `con`은 `u_con.logException` 부가 로깅에만 사용).
- CI 실패 스택(확정): findColumn→con / getMetaData→u_stmt (둘 다 getGeneratedKeys의 con/u_stmt-없는 결과셋).
- cubrid-manual은 엔진 SQL/함수/예약어용이라, 드라이버 내부 동작은 매뉴얼이 아니라 드라이버 소스로 확정.
- CI 실패 수치와 전체 처분 내역은 내부 통합 문서에 정리.

# CUBRID JDBC 드라이버 createCUBRIDException NullPointerException (닫힌 Statement/ResultSet 예외 변환 경로 결함)

- 분류: bug
- 날짜: 2026-07-11
- 관련: Hibernate ORM CUBRID CI 잔여 실패 분석, JDBC 드라이버 수정 트랙

## 목적
Hibernate ORM을 CUBRID(10.2~11.4)에서 돌릴 때 가장 많이 발생하는 비결정적 테스트 실패인 `createCUBRIDException` NullPointerException의 근본 원인을 드라이버 소스로 규명하고, 이것이 방언(Dialect)이나 테스트로는 고칠 수 없는 JDBC 드라이버 결함임을 확정한다.

## 배경
CUBRID CI 5버전 매트릭스에서 잔여 실패의 최대 군집이 이 NPE다. 증상은 항상 아래 형태다.

```
java.lang.NullPointerException: Cannot invoke
  "cubrid.jdbc.driver.CUBRIDConnection.createCUBRIDException(int, ...)" because <참조>가 null
```

특징은 (1) 버전/실행마다 실패 대상이 요동치고(같은 코드인데 어떤 버전은 극소수, 다른 버전은 수십 개), (2) 서로 무관한 테스트(merge, persist, lock 등)가 무작위로 걸린다는 것이다. 로직 버그라면 항상 같은 지점이 터져야 하는데 그렇지 않아, 특정 타이밍 조건에서만 발현하는 결함으로 의심되었다.

## 범위 / 방법
- CI JUnit 결과에서 실패 시그니처와 버전별 분포를 수집.
- CUBRID JDBC 드라이버 소스(`cubrid.jdbc.driver` 패키지: `CUBRIDStatement`, `CUBRIDResultSet`, `CUBRIDConnection`)를 정독해 `createCUBRIDException`의 정의와 호출 경로를 추적.

## 발견 / 관찰
1. `createCUBRIDException(...)`는 `CUBRIDConnection`의 메서드로, 서버 오류 코드를 표준 `SQLException`(CUBRID 계열 예외)으로 변환한다. 드라이버 전반의 오류 처리는 소유 커넥션 참조를 통해 `con.createCUBRIDException(...)` 형태로 예외를 만든다(드라이버 내 호출 지점 100여 곳).
2. **CI 스택으로 확정된 경로**: 실패 메시지는 `Cannot invoke "cubrid.jdbc.driver.CUBRIDConnection.createCUBRIDException(int, java.lang.Throwable)" because "this.con" is null`, 최상단 드라이버 프레임은 `cubrid.jdbc.driver.CUBRIDResultSet.findColumn`이며 Hibernate `MutationExecutorSingleNonBatched.performNonBatchedOperations`(INSERT + 생성값 조회)에서 호출된다.
3. `CUBRIDResultSet.close()`는 정리 시 `is_closed = true`와 함께 소유 커넥션 참조 `con`을 **null로 설정**한다. 그런데 "닫힘"을 표준 예외로 알리는 가드가 바로 그 `con`을 역참조한다.

   ```java
   private void checkIsOpen() throws SQLException {
       if (is_closed) {
           throw con.createCUBRIDException(CUBRIDJDBCErrorCode.result_set_closed, null); // con은 close()가 null로 만듦
       }
   }
   ```

   `findColumn(String)`·`getXxx(...)` 등 대부분의 접근자는 진입 시 `checkIsOpen()`을 부른다. 따라서 **닫힌 ResultSet을 다시 건드리면**, "닫혔다(result_set_closed)"고 알리려는 바로 그 코드가 `con`(null)을 역참조해 NPE를 던진다. 닫힘을 보고하는 가드가 닫힘 상태에서 스스로 터지는(self-defeating) 구조다.
4. 결과적으로 "ResultSet이 닫혔습니다"라는 명확한 `SQLException` 대신 raw `NullPointerException`이 노출되어, 원래 상황(use-after-close)을 감춘다. **오류 보고 경로가 `con`의 non-null을 가정하는데 `close()`가 그 가정을 깨뜨린다.** 동일 패턴(`close()`가 `con`을 null로 만들고, 오류 경로가 `con.createCUBRIDException(...)`을 호출)은 `CUBRIDStatement`(`statement_closed`) 등 다른 클래스에도 존재한다.
5. 비결정성의 원인: "닫힌 뒤 접근(use-after-close)" 또는 close와 사용의 경합이 발생하는 타이밍이 GC/finalizer, 커넥션 풀 재활용, 테스트 정리(cleanup) 순서에 의존하기 때문이다. 그래서 버전/실행마다 걸리는 테스트가 달라진다.

## 결론
- 이 NPE는 CUBRID JDBC 드라이버 내부 결함이다. `close()`가 커넥션 참조를 null로 만드는데, 오류 변환 경로가 그 참조를 무조건 역참조하기 때문에, 닫힌 뒤 접근 시 원래 오류가 raw NPE로 둔갑한다.
- **방언(CUBRIDDialect)이나 Hibernate 테스트로는 고칠 수 없다.** 비결정적이라 특정 테스트를 skip해도 다른 테스트로 옮겨가므로 skip 대상도 아니다.
- Hibernate CUBRID CI에서 다른 결정적 실패(엔진 한계, core 렌더링 등)를 모두 처분한 뒤에도 남는, **안정적 그린 CI의 사실상 유일한 걸림돌**이다.

## 재현 (최소 케이스)
Hibernate 없이 드라이버 단독으로 재현된다. 핵심은 **닫힌 ResultSet의 접근자를 다시 호출**하는 것이다(실행에는 기동 중인 CUBRID가 필요).

```java
Class.forName("cubrid.jdbc.driver.CUBRIDDriver");
try (Connection c = DriverManager.getConnection("jdbc:cubrid:localhost:33000:demodb:::", "public", "")) {
    Statement st = c.createStatement();
    ResultSet rs = st.executeQuery("select 1");
    rs.next();
    rs.close();                 // close(): is_closed=true, con=null
    try {
        rs.getString(1);        // 또는 rs.findColumn("x") : 닫힌 뒤 접근 -> checkIsOpen()
        System.out.println("예외 없음(예상과 다름)");
    } catch (SQLException e) {
        System.out.println("기대(정상): " + e.getMessage());        // "ResultSet is closed"
    } catch (NullPointerException npe) {
        System.out.println("버그: " + npe.getMessage());            // ...createCUBRIDException(...) because this.con is null
    }
}
```

- **기대(정상 드라이버)**: `SQLException`("result set closed").
- **실제(버그)**: `NullPointerException`(닫힘을 알리려다 `con`이 null이라 예외 생성 자체가 실패).
- **Hibernate에서 "랜덤"으로 보이는 이유**: 위 use-after-close를 Hibernate가 의도적으로 하는 게 아니라, INSERT + 생성값 조회 경로(위 스택)에서 ResultSet이 연결 정리/풀 반환/선행 오류 등으로 이미 닫힌 뒤 `findColumn`으로 접근하는 상황이 GC/finalizer·커넥션 풀 재활용·정리 순서 타이밍에 따라 생겼다 안 생겼다 한다. **드라이버 결함 자체는 결정적**(닫힌 ResultSet 접근 -> 항상 NPE)이고, **비결정적인 것은 "언제 닫힌 뒤 접근이 일어나느냐"**다. 그래서 버전/실행마다 걸리는 테스트가 달라진다.

## 다음 단계
- 드라이버 수정 이슈로 등록(JDBC 드라이버 트랙). 수정 방향 후보:
  1. 예외 변환 경로에 null 가드: `con`이 null이면 `con`을 경유하지 않고 표준 "closed" `SQLException`을 직접 생성.
  2. `close()`에서 `con`을 null로 만들지 않고 `is_closed` 플래그로만 상태 관리(참조는 유지).
  3. 닫힌 객체 접근을 메서드 진입 시점에 명확한 `SQLException`으로 차단해, `catch (NullPointerException)`에 의존하는 변환 자체를 없앰.
- 드라이버 수정 후 CUBRID CI 재검증. 이 NPE가 사라지면 잔여 실패는 결정적 항목만 남아 그린 판정이 안정된다.

## 참고
- 드라이버 소스(확정): `cubrid.jdbc.driver.CUBRIDResultSet#checkIsOpen`(`if (is_closed) throw con.createCUBRIDException(result_set_closed, null)`), 동 클래스 `findColumn`/`getXxx`가 진입 시 `checkIsOpen()` 호출, `close()`가 `is_closed=true`+`con=null` 설정. 예외 생성기 `cubrid.jdbc.driver.CUBRIDConnection#createCUBRIDException`. 동일 패턴 `cubrid.jdbc.driver.CUBRIDStatement`(`statement_closed`).
- CI 실패 스택(확정): `NullPointerException ... createCUBRIDException(int, java.lang.Throwable) because "this.con" is null` at `CUBRIDResultSet.findColumn` <- Hibernate `MutationExecutorSingleNonBatched.performNonBatchedOperations`.
- cubrid-manual은 엔진 SQL/함수/예약어를 다루므로 드라이버 내부 동작은 매뉴얼이 아니라 드라이버 소스로 확정했다.
- CI 실패 수치와 전체 처분 내역은 내부 통합 문서에 정리.

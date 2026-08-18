# CUBRID JDBC 에러 메시지 테이블 스레드 안전화 설계 (CUBRIDJDBCErrorCode)

- 분류: spec
- 날짜: 2026-08-19
- 관련: [APIS-1090이 드러낸 에러 메시지 테이블 레이스 분석](../bug/2026-08-19-apis1090-not-supported-message-race.md)

## 요약
`CUBRIDJDBCErrorCode.getMessage()`의 스레드 안전하지 않은 지연 초기화를, **지연 초기화 자체를 제거하고 불변 맵으로 전환**해 해결한다. 정적 초기화로 옮겨 JVM의 클래스 초기화 보장에 안전성을 위임하고, 더 이상 변하지 않는 맵이므로 `Hashtable`을 `HashMap` + `Collections.unmodifiableMap`으로 바꿔 조회 경로의 락을 없앤다. 동일 결함이 있는 `UErrorCode` 2건은 별도 이슈로 분리한다.

## 목적
QA 회귀 리포트에서 드러난 "동시 최초 호출 시 에러 메시지가 `null`" 결함을 드라이버에서 근본 해결하기 위한 설계를 확정한다. 원인 분석은 별도 문서에 있으므로, 이 문서는 **무엇을 어떻게 바꿀지**만 다룬다.

## 배경

현재 코드는 참조를 먼저 공개하고 나중에 채운다.

```java
private static Hashtable<Integer, String> messageString;      // volatile 아님

private static void setMessageHash() {
    messageString = new Hashtable<Integer, String>();          // (A) 참조 공개
    messageString.put(new Integer(unknown), "");               // (B) 이후 44건 채움
    ...
}

public static String getMessage(int code) {
    if (messageString == null) setMessageHash();               // (C) 존재 여부만 확인
    return (String) messageString.get(new Integer(code));      // (D) 조회
}
```

(A)와 (B) 사이에 다른 스레드가 (C)를 통과해 (D)를 실행하면 **비어 있는 테이블**에서 `null`을 받는다. (C)에 락이 없어 두 스레드가 동시에 `setMessageHash()`에 진입할 수 있고, 그 경우 이미 완성된 테이블이 새 빈 테이블로 교체되기도 한다.

- 도입 시점: 에러코드가 배열 인덱스에서 음수 코드로 바뀌며 `String[]`이 지연 초기화 `Hashtable`로 교체될 때(2012-08-23)
- 노출 계기: 미지원 메서드 예외 전환(APIS-1090)이 `getMessage`의 네 번째 호출자를 추가하면서, 다중 스레드가 동시에 첫 호출을 하게 됨
- 실측: 호출당 약 25% 확률로 `null` (20 스레드 동형 재현 12회 중 12회 발생)

## 범위 / 방법

- 대상: `cubrid/jdbc/driver/CUBRIDJDBCErrorCode.java` 1개 파일
- 제외: `cubrid/jdbc/jci/UErrorCode.java`의 `codeToMessage` / `codeToCASMessage` 2건 (동일 결함, 별도 이슈)
- 방법: 저장소의 기존 관용구 조사, 바이트코드 확인, 컴파일 타임 상수 인라인 실증, 재현 도구 기반 전후 대조

## 발견 / 관찰

### (1) 저장소의 기존 관용구는 이미 즉시 초기화다

같은 저장소의 다른 static `Hashtable`은 전부 정적 블록에서 즉시 초기화한다. 지연 초기화를 쓰는 3곳이 오히려 예외다.

| 클래스 | 초기화 방식 |
|---|---|
| `CUBRIDJdbcInfoTable` | `static { ... }` |
| `CUBRIDKeyTable` | `static { ... }` |
| `CUBRIDXidTable` | `static { ... }` |
| `CUBRIDConnectionPoolManager` | `static { ... }` |
| **`CUBRIDJDBCErrorCode`** | **지연 초기화 (결함)** |
| **`UErrorCode` (테이블 2개)** | **지연 초기화 (결함)** |

### (2) 지연시킬 실익이 애초에 없다

44개 코드 필드가 `static final`이 **아니므로**, 필드를 읽는 것만으로도 클래스 초기화(`<clinit>`)가 실행된다. 즉 클래스가 로드되는 시점이 이미 첫 에러 시점이고, 즉시 초기화로 바꿔도 **로드 시점은 달라지지 않는다.** 추가 비용은 `<clinit>`에 `put` 44건이 붙는 것뿐이다.

### (3) 코드 커버리지가 완전하다

정의된 에러코드 필드 44개와 등록 엔트리 44개가 1:1로 정확히 일치한다. 따라서 수정 후 `null`은 **정의되지 않은 코드**를 조회할 때만 나오며, 이는 기존 계약 그대로다. 값이 `null`인 엔트리도 없다(`unknown`만 빈 문자열).

### (4) 불변으로 만들면 `Hashtable`의 동기화는 순손실이다

정적 초기화 이후 이 맵은 변하지 않는다. 안전성은 두 겹으로 보장된다.

- 클래스 초기화 락(JLS 12.4.2): `<clinit>` 완료 전에는 다른 스레드가 클래스를 사용할 수 없다
- `final` 필드 의미론(JLS 17.5): 초기화를 본 스레드는 완성된 맵을 본다

이 상태에서 `Hashtable`의 메서드 단위 `synchronized`는 아무 안전성도 추가하지 못하면서 조회마다 모니터 획득 비용만 남긴다. 스레드 안전화의 결론은 락을 더 잘 거는 것이 아니라 **불변으로 만들어 락을 없애는 것**이다.

`final`은 참조만 지키고 내용은 지키지 못하므로, 불변을 관례가 아니라 강제로 만들려면 `Collections.unmodifiableMap`으로 감싼다. 빌드 타겟이 1.8이라 `Map.of`는 사용할 수 없다.

### (5) 에러코드 필드를 `static final`로 바꾸는 것은 별개 결정이다

`private static final Map messageString`(참조 final)과 `public static final int not_supported = -21105`(컴파일 타임 상수)는 성격이 다르다.

| | `final Map messageString` | `static final int not_supported` |
|---|---|---|
| 컴파일 결과 | 런타임에 `getstatic`으로 읽음 | 사용처 바이트코드에 **값이 인라인** |
| 라이브러리 교체 시 | 새 값 반영 | **옛 값 잔류** |

실증: 상수를 참조하는 소비자 클래스를 컴파일한 뒤 라이브러리만 새 값으로 교체하면, `static final` 쪽은 옛 값을 계속 출력하고 non-final 쪽만 새 값을 읽는다. 컴파일러는 문자열 연결까지 상수 하나로 접어버린다.

전례도 있다. 2012년 재번호 작업이 바로 이 코드들의 값을 전면 변경했다. 당시 `static final`이었다면 옛 드라이버로 컴파일한 애플리케이션은 라이브러리만 교체해서는 옛 값을 계속 사용했을 것이다.

반대 논거도 있다. 현재 `public static int`는 **외부에서 쓰기가 가능**해(`CUBRIDJDBCErrorCode.not_supported = 0;` 이 컴파일된다) 실질적인 구멍이다. 따라서 전환 자체는 방어할 가치가 있으나, 이는 **스레드 안전성이 아니라 공개 API와 바이너리 호환성에 관한 결정**이므로 별도 이슈에서 "값을 다시 변경하지 않는다"를 명문화한 뒤 다루는 것이 맞다. 이번 이슈에 섞으면 리뷰 축이 둘로 갈린다.

## 결론: 확정 설계

### 변경 후 형태

```java
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;                       // java.util.Hashtable 제거

public class CUBRIDJDBCErrorCode {

    public static int unknown = -21100;
    ...                                     // 코드 필드 44개 변경 없음
    public static int file_not_found_prop = -21143;

    // 주의: 아래 초기화는 위 코드 상수들이 대입된 뒤 실행된다(정적 초기화자는 텍스트 순서).
    // 새 에러 코드는 반드시 이 줄 위에 선언할 것.
    private static final Map<Integer, String> messageString = createMessageMap();

    private static Map<Integer, String> createMessageMap() {
        Map<Integer, String> m = new HashMap<Integer, String>();
        m.put(unknown, "");
        m.put(not_supported, "Not supported method");
        ...                                 // 44건, 메시지 문자열 변경 없음
        m.put(file_not_found_prop, "File not found - ");
        return Collections.unmodifiableMap(m);
    }

    public static String getMessage(int code) {
        return messageString.get(code);      // 분기 없음, 락 없음
    }
}
```

### 변경 항목

| 항목 | 이전 | 이후 |
|---|---|---|
| 초기화 시점 | 지연 (`getMessage` 첫 호출) | 즉시 (클래스 초기화) |
| 초기화 진입 보호 | 없음 | JVM 클래스 초기화 락 |
| 자료구조 | `Hashtable` (조회마다 락) | `HashMap` (락 없음) |
| 불변성 | 없음 | `final` 참조 + `unmodifiableMap` 내용 |
| 키 박싱 | `new Integer(code)` | 오토박싱 (`Integer.valueOf`) |
| `setMessageHash()` | 외부에서 호출 가능한 상태 변경 메서드 | **삭제** (순수 팩터리로 대체) |

핵심은 `setMessageHash()`가 사라지는 것이다. 결함은 "이 메서드를 아무 보호 없이 부를 수 있다"였고, 부를 수 있는 메서드 자체가 없어지면 결함의 전제가 사라진다.

### 관측 가능한 동작 변화

| 상황 | 이전 | 이후 |
|---|---|---|
| 등록된 코드 조회 | 정상 문자열 또는 **`null`(레이스)** | 항상 정상 문자열 |
| 등록되지 않은 코드 조회 | `null` | `null` (계약 유지) |
| 클래스 초기화 시점 | 첫 참조 시 | 첫 참조 시 (변화 없음) |
| 예외 메시지 문자열 | 변경 없음 | 변경 없음 |

### 위험 점검

| 위험 | 판단 |
|---|---|
| 정적 초기화자 실행 순서 | 코드 필드 44개가 모두 맵 초기화보다 텍스트상 위에 있어야 한다. 현재 배치가 이미 충족하며, 주석으로 못을 박는다 |
| 정적 초기화 중 예외 | `ExceptionInInitializerError`로 클래스가 영구 사용 불가가 된다. 여기서는 `Map.put`만 하고 null 키와 null 값이 없음을 확인했다 |
| `unmodifiableMap` 회귀 | 이후 누가 맵을 변경하려 하면 `UnsupportedOperationException`으로 즉시 드러난다. 의도된 동작이다 |
| 커버리지 누락 | 44/44 일치를 확인했다 |

### 대안 검토와 기각 사유

| 대안 | 기각 사유 |
|---|---|
| `getMessage()`에 `synchronized` 추가 | diff는 가장 작지만, 유지할 이유가 없는 지연 초기화를 살린 채 조회마다 락 비용만 남는다 |
| `volatile` + 안전 공개 (로컬에서 채운 뒤 대입) | 정석이지만 코드가 늘고, 마찬가지로 불필요한 지연을 유지한다 |
| `Map` 제거 후 `switch` 문 | 44개 분기로 전면 재작성이 필요해 변경 표면이 과도하다 |
| 정적 블록을 클래스 맨 아래 배치 | 필드 선언 순서 함정에는 더 강하지만, 위에 선언된 필드를 아래 블록에서 대입하는 형태라 가독성이 떨어진다. 주석으로 대체 |

## 검증 계획

레이스는 "수정 후 통과"가 증명이 되지 못하므로, 수정 전에 재현되던 것이 수정 후 사라지는 **전후 대조**로 검증한다.

| 단계 | 방법 | 기대 |
|---|---|---|
| 1 | 동시 최초 호출 재현 도구, 20회 x 50 스레드 | 수정 전 111/1000 → **0/1000** |
| 2 | TC 동형 재현 (20 스레드가 실제 접속 후 미지원 메서드 호출) | 수정 전 12/12 실행에서 발생 → **0** |
| 3 | 바이트코드 확인 | `<clinit>`에 `put` 44건, `getMessage`에서 분기 소멸, 조회 경로에 `monitorenter` 없음 |
| 4 | 불변성 확인 | 리플렉션으로 `put` 시도 시 `UnsupportedOperationException` |
| 5 | CTP JDBC 스위트 | 회귀 없음 |

### 단위 테스트를 추가하지 않는 이유

이 레이스는 "클래스가 아직 로드되지 않았을 것"이 전제라 **JVM당 단 한 번만 유효**하다. JUnit 스위트에 넣으면 다른 테스트가 먼저 클래스를 로드하는 순간 무력화되어, 통과해도 아무것도 보장하지 못하는 테스트가 된다. 재현 프로그램을 이슈에 첨부해 수동 대조 절차로 남기는 편이 정직하다.

## 다음 단계

- 이슈 1: 본 설계 구현 (`CUBRIDJDBCErrorCode`). 브랜치 전략(독립 브랜치 선행 머지 vs 미지원 메서드 전환 브랜치 포함)은 미확정
- 이슈 2: `UErrorCode.codeToMessage` / `codeToCASMessage` 2건에 동일 수정 적용. `UError.getErrorMsg`가 서버 에러마다 호출하고 외부 호출처가 29곳이라 노출은 오히려 더 크다
- 이슈 3(선택): 에러코드 필드의 `public static final` 전환. 바이너리 호환성 정책 확정이 선행되어야 한다
- 이슈 1 반영 후 관련 TC의 기대 결과 파일 갱신 (기대 결과 갱신은 반드시 본 수정 이후에 해야 한다. 그러지 않으면 영구 flaky TC가 된다)

## 참고
- JLS 12.4.2 (클래스 초기화 절차와 락), JLS 17.5 (final 필드 의미론), JLS 13.4.9 (상수 필드의 바이너리 호환성)
- 원인 분석 노트: [APIS-1090이 드러낸 에러 메시지 테이블 레이스](../bug/2026-08-19-apis1090-not-supported-message-race.md)

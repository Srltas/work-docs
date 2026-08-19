# CUBRID JDBC 에러 메시지 테이블 스레드 안전화 및 에러코드 상수화 설계 (CUBRIDJDBCErrorCode)

- 분류: spec
- 날짜: 2026-08-19
- 관련: [APIS-1090이 드러낸 에러 메시지 테이블 레이스 분석](../bug/2026-08-19-apis1090-not-supported-message-race.md)

## 요약
두 단계로 처리한다. **1부(이슈 1, 구현 완료)**: `CUBRIDJDBCErrorCode.getMessage()`의 스레드 안전하지 않은 지연 초기화를, 지연 초기화 자체를 제거하고 불변 맵으로 전환해 해결한다. 정적 초기화로 옮겨 JVM의 클래스 초기화 보장에 안전성을 위임하고, 더 이상 변하지 않는 맵이므로 `Hashtable`을 `HashMap` + `Collections.unmodifiableMap`으로 바꿔 조회 경로의 락을 없앤다. **2부(이슈 3)**: 46개 에러코드 필드를 `public static final int`로 전환해 공개 필드 쓰기 구멍을 막고, 1부가 남긴 선언 순서 제약을 제거한다. 동일 결함이 있는 `UErrorCode` 2건은 별도 이슈로 분리한다.

## 목적
QA 회귀 리포트에서 드러난 "동시 최초 호출 시 에러 메시지가 `null`" 결함을 드라이버에서 근본 해결하기 위한 설계를 확정한다. 원인 분석은 별도 문서에 있으므로, 이 문서는 **무엇을 어떻게 바꿀지**만 다룬다.

## 배경

현재 코드는 참조를 먼저 공개하고 나중에 채운다.

```java
private static Hashtable<Integer, String> messageString;      // volatile 아님

private static void setMessageHash() {
    messageString = new Hashtable<Integer, String>();          // (A) 참조 공개
    messageString.put(new Integer(unknown), "");               // (B) 이후 46건 채움
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
- 방법: 저장소의 기존 관용구 조사, 바이트코드 확인, 자료구조 교체 실험, 컴파일 타임 상수 인라인 실증, 재현 도구 기반 전후 대조

## 발견 / 관찰

### (1) 결함은 자료구조가 아니라 초기화 프로토콜에 있다

`Hashtable`이 스레드 안전하다는 것은 **개별 연산**이 원자적이라는 뜻이지, 그 연산들을 엮은 로직이 옳다는 뜻이 아니다. 이번에 깨진 불변식은 **"참조가 non-null이면 46건이 다 들어 있다"** 이며, 이는 맵이 알 수도 지킬 수도 없는 약속이다. 빈 테이블에 조회가 들어오면 맵은 정확하게 "없음"을 반환하며 제 역할을 다한다.

자료구조를 JDK에서 가장 강한 동시성 맵으로 바꿔도 결함이 남는 것으로 이를 확인했다. 나머지 구조를 그대로 두고 `Hashtable`을 `ConcurrentHashMap`으로 교체한 뒤 50 스레드 동시 최초 호출을 5회 반복한 결과다.

```
ConcurrentHashMap: null 6/50
ConcurrentHashMap: null 5/50
ConcurrentHashMap: null 2/50
ConcurrentHashMap: null 6/50
ConcurrentHashMap: null 4/50
```

따라서 해결책은 더 강한 자료구조가 아니라 **초기화 방식의 교체**여야 한다. 동시에 이것은 뒤에서 `Hashtable`을 `HashMap`으로 바꾸는 것이 안전성을 낮추지 않는다는 근거이기도 하다. `Hashtable`의 락은 이 결함을 막아준 적이 없다.

### (2) 저장소의 기존 관용구는 이미 즉시 초기화다

같은 저장소의 다른 static `Hashtable`은 전부 정적 블록에서 즉시 초기화한다. 지연 초기화를 쓰는 3곳이 오히려 예외다.

| 클래스 | 초기화 방식 |
|---|---|
| `CUBRIDJdbcInfoTable` | `static { ... }` |
| `CUBRIDKeyTable` | `static { ... }` |
| `CUBRIDXidTable` | `static { ... }` |
| `CUBRIDConnectionPoolManager` | `static { ... }` |
| **`CUBRIDJDBCErrorCode`** | **지연 초기화 (결함)** |
| **`UErrorCode` (테이블 2개)** | **지연 초기화 (결함)** |

### (3) 지연시킬 실익이 애초에 없다

46개 코드 필드가 `static final`이 **아니므로**, 필드를 읽는 것만으로도 클래스 초기화(`<clinit>`)가 실행된다. 즉 클래스가 로드되는 시점이 이미 첫 에러 시점이고, 즉시 초기화로 바꿔도 **로드 시점은 달라지지 않는다.** 추가 비용은 `<clinit>`에 `put` 46건이 붙는 것뿐이다.

2부(상수화) 적용 후에는 근거가 달라진다. 46개가 컴파일 타임 상수가 되면 필드를 읽어도 클래스 초기화가 트리거되지 않고, `<clinit>` 시점이 `getMessage()` 호출 시점으로 밀린다. 결과는 동일한데, `<clinit>`의 유일한 효과가 private 필드 `messageString` 대입이고 그 필드에 닿는 경로가 `getMessage(int)` 하나뿐이며 그 호출 자체가 초기화를 트리거하기 때문이다. 이 근거는 이 클래스에 두 번째 정적 필드나 부수효과 있는 정적 블록이 생기면 다시 검토해야 한다.

### (4) 코드 커버리지가 완전하다

정의된 에러코드 필드 46개와 등록 엔트리 46개가 1:1로 정확히 일치한다. 따라서 수정 후 `null`은 **정의되지 않은 코드**를 조회할 때만 나오며, 이는 기존 계약 그대로다. 값이 `null`인 엔트리도 없다(`unknown`만 빈 문자열).

### (5) 불변으로 만들면 `Hashtable`의 동기화는 순손실이다

정적 초기화 이후 이 맵은 변하지 않는다. 안전성은 두 겹으로 보장된다.

- 클래스 초기화 락(JLS 12.4.2): `<clinit>` 완료 전에는 다른 스레드가 클래스를 사용할 수 없다
- `final` 필드 의미론(JLS 17.5): 초기화를 본 스레드는 완성된 맵을 본다

이 상태에서 `Hashtable`의 메서드 단위 `synchronized`는 아무 안전성도 추가하지 못하면서 조회마다 모니터 획득 비용만 남긴다. 스레드 안전화의 결론은 락을 더 잘 거는 것이 아니라 **불변으로 만들어 락을 없애는 것**이다.

`final`은 참조만 지키고 내용은 지키지 못하므로, 불변을 관례가 아니라 강제로 만들려면 `Collections.unmodifiableMap`으로 감싼다. 빌드 타겟이 1.8이라 `Map.of`는 사용할 수 없다.

### (6) 에러코드 상수화: 동결 약속은 이미 존재하며, 비용은 값을 바꿀 때만 발생한다

`private static final Map messageString`(참조 final)과 `public static final int not_supported = -21105`(컴파일 타임 상수)는 성격이 다르다.

| | `final Map messageString` | `static final int not_supported` |
|---|---|---|
| 컴파일 결과 | 런타임에 `getstatic`으로 읽음 | 소비자 바이트코드에 **값이 인라인** |
| 라이브러리 교체 시 | 새 값 반영 | 재컴파일 전까지 옛 값 유지 |

JVM의 캐싱이 아니라 컴파일러 동작이다(JLS 4.12.4, 13.1). 상수를 참조하는 소비자를 컴파일하면 정의 클래스에 대한 참조 자체가 사라진다. 실측: 소비자 바이트코드가 `ldc "CODE = -21105"` 하나로 접히고, 정의 클래스를 클래스패스에서 제거해도 실행된다.

#### 값을 바꾸면 실제로 무슨 일이 일어나는가

소비자는 세 부류이고, 상수화가 영향을 주는 건 셋째뿐이다.

| 부류 | 예 | 값 변경 시 |
|---|---|---|
| 같이 컴파일되는 소비자 | 드라이버 자신, CTP 시나리오(매 실행 `javac` 재컴파일), 엔진 빌드 | 새 값을 그대로 따라감. 상수화와 무관 |
| 생값을 박아둔 소비자 | `.answer` 파일의 `-21104` 190건, `assertEquals(-21107, ...)`, 엔진의 손복사 7개 | 상수화 여부와 무관하게 동일하게 깨짐 |
| **미리 컴파일된 외부 앱** | Maven Central `org.cubrid:cubrid-jdbc` 사용자 | **여기서만 차이가 난다** |

셋째 부류를 실측했다. 드라이버가 `not_supported`를 -21105에서 -21999로 바꾸고, 앱은 재컴파일 없이 라이브러리만 교체한 상황이다.

```
상수화 안 함 : 미지원 메서드로 인식 -> 폴백 실행    (앱이 믿는 값 = -21999)
상수화 함    : 인식 실패 -> 예외가 그대로 전파      (앱이 믿는 값 = -21105)
```

`final`은 "업그레이드 후에도 동작하던 앱"을 **"조용히 오작동하는 앱"** 으로 바꾼다. 예외도 경고도 없다.

반대로 필드를 **삭제**하는 경우엔 `final` 쪽이 더 안전하다. 값이 이미 박혀 있어 앱이 계속 동작하는 반면, non-final은 `NoSuchFieldError`로 죽는다. 즉 위험한 조작은 삭제가 아니라 **값 변경**과 **삭제 후 번호 재사용**이다.

값을 바꿔야 할 상황이 오면 선택지는 셋이다. ① 바꾸지 않고 새 코드를 추가한다(14년간의 실제 방식, 값 공간이 -21100~-21145로 연속이라 -21146부터 이어붙이면 된다) ② 옛 코드를 남긴 채 새 코드를 추가하고 옛 것을 deprecated 처리한다 ③ 정말로 변경하고 메이저 버전을 올려 릴리스 노트에 재컴파일 필수를 명시한다.

#### 동결 약속은 이미 세 채널로 존재한다

| 채널 | 내용 |
|---|---|
| 공식 매뉴얼 | `www.cubrid.org/manual` 의 "JDBC Error Codes and Error Messages" 가 **-21101~-21141(46개 중 41개)을 메시지 문자열까지 그대로 게시** 중. 값 변경은 상수화와 무관하게 이미 문서 위반이다 |
| `SQLException.getErrorCode()` | `CUBRIDException`이 코드를 vendorCode 슬롯에 그대로 넣는다. 사용자가 생값으로 비교해도 동일하게 깨지므로 **번호는 이미 행위적으로 동결**돼 있다 |
| 엔진의 손복사 | `cubrid/pl_engine/pl_server/.../CUBRIDServerSideJDBCErrorCode.java`가 46개 중 7개(-21109, -21110, -21115, -21116, -21117, -21121, -21128)를 `public static final int` 리터럴로 손복사해 두고 있다. "The following codes are ported from CUBRIDJDBCErrorCode.java" 주석 외에 강제하는 장치가 없다 |

즉 `final`은 새 약속을 만드는 것이 아니라, **이미 지키고 있던 약속을 어겼을 때의 벌칙을 추가**하는 것이다.

#### 이력이 뒷받침한다

16개 리비전 전부에서 name=value 집합을 추출해 기계적으로 확인했다. 2012-08-23 재번호(`6688814`) 이후 **기존 값 변경 0건, 이름 변경 0건, 삭제 0건.** 유일한 편집 조작은 append이며, 실질 변경 이벤트는 14년간 2회(APIS-869가 2개, APIS-1089가 2개 추가)다. 값 공간은 -21100~-21145로 빈틈이 없다.

동일 제품 선례도 있다. 엔진이 1년 앞서 같은 종류의 정적 초기화 레이스를 겪고 같은 방식으로 고쳤다: `78b451584 [CBRD-26132] create error message table earlier than any threads that use it (#6269)` (2025-06-23). 그 테이블의 상수는 이미 `final`이다.

C/C++ 쪽은 완전한 무관계다. broker/CAS/CCI 어디에도 -211xx 대역이 없다(CAS는 -10000~-10200, 엔진 에러는 ~-1375). 번호가 와이어를 타지도 않는다.

#### 상수화를 해야 할 근거

1. 현재 `public static int`는 외부에서 **쓰기가 가능하다.** `CUBRIDJDBCErrorCode.not_supported = 0;` 이 컴파일된다
2. 1부가 남긴 **선언 순서 제약이 사라진다.** 실제 파일에 상수화를 적용하고 47번째 코드를 맵 초기화 줄 아래에 선언해 확인했다:

```
맵 크기 47, 키 0 존재 false, 새 코드 조회 "Brand new error", 메시지 없는 코드 0개
createMessageMap 바이트코드: getstatic 0, 인라인 적재 94
```

`getstatic`이 0이므로 필드를 런타임에 읽지 않고, 따라서 대입 순서라는 개념 자체가 사라진다. 1부에서 넣은 NOTE 주석은 근거가 사라지므로 삭제한다.

#### 남는 비용

| 항목 | 판단 |
|---|---|
| 값 변경 시 외부 앱 조용한 오작동 | 실재한다. 다만 값 변경 자체가 이미 매뉴얼 위반이고 14년간 0건이다 |
| 쓰기 쪽 바이너리 비호환 | 오늘 합법인 `CUBRIDJDBCErrorCode.not_supported = 0;`으로 미리 컴파일된 앱은 라이브러리 교체 후 런타임에 `IllegalAccessError`로 죽는다. 다만 드라이버 상수에 대입하는 코드는 애초에 버그이며, 조용히 틀리는 것보다 시끄럽게 죽는 편이 낫다 |
| 배포 채널에 변경 고지 수단 없음 | 이 프로젝트에는 CHANGELOG도 semver 선언도 deprecation 채널도 없다. 본 이슈에서는 문서화를 하지 않기로 했고, 필요해지면 별도로 다룬다 |
| 제품 내 컴파일-후-배포 사례 | `$CUBRID/vm/pl_server.jar`가 드라이버 전체를 번들한 shaded fat jar이며, 번들 사본(11.3.2.0058)과 설치 드라이버(11.3.3.0066)의 버전이 어긋나 있다. 현재는 클래스패스가 분리돼 있어 위험하지 않으나, 이 패턴의 첫 사례로 기록해 둔다 |

## 결론: 확정 설계

### 1부: 스레드 안전화 (이슈 1)

#### 변경 후 형태

```java
import java.util.Collections;
import java.util.HashMap;
import java.util.Map;                       // java.util.Hashtable 제거

public class CUBRIDJDBCErrorCode {

    public static int unknown = -21100;
    ...                                     // 코드 필드 46개 변경 없음
    public static int file_not_found_prop = -21143;

    // 주의: 아래 초기화는 위 코드 상수들이 대입된 뒤 실행된다(정적 초기화자는 텍스트 순서).
    // 새 에러 코드는 반드시 이 줄 위에 선언할 것.
    private static final Map<Integer, String> messageString = createMessageMap();

    private static Map<Integer, String> createMessageMap() {
        Map<Integer, String> m = new HashMap<Integer, String>();
        m.put(unknown, "");
        m.put(not_supported, "Not supported method");
        ...                                 // 46건, 메시지 문자열 변경 없음
        m.put(file_not_found_prop, "File not found - ");
        return Collections.unmodifiableMap(m);
    }

    public static String getMessage(int code) {
        return messageString.get(code);      // 분기 없음, 락 없음
    }
}
```

#### 변경 항목

| 항목 | 이전 | 이후 |
|---|---|---|
| 초기화 시점 | 지연 (`getMessage` 첫 호출) | 즉시 (클래스 초기화) |
| 초기화 진입 보호 | 없음 | JVM 클래스 초기화 락 |
| 자료구조 | `Hashtable` (조회마다 락) | `HashMap` (락 없음) |
| 불변성 | 없음 | `final` 참조 + `unmodifiableMap` 내용 |
| 키 박싱 | `new Integer(code)` | 오토박싱 (`Integer.valueOf`) |
| `setMessageHash()` | 외부에서 호출 가능한 상태 변경 메서드 | **삭제** (순수 팩터리로 대체) |

핵심은 `setMessageHash()`가 사라지는 것이다. 결함은 "이 메서드를 아무 보호 없이 부를 수 있다"였고, 부를 수 있는 메서드 자체가 없어지면 결함의 전제가 사라진다.

#### 관측 가능한 동작 변화

| 상황 | 이전 | 이후 |
|---|---|---|
| 등록된 코드 조회 | 정상 문자열 또는 **`null`(레이스)** | 항상 정상 문자열 |
| 등록되지 않은 코드 조회 | `null` | `null` (계약 유지) |
| 클래스 초기화 시점 | 첫 참조 시 | 첫 참조 시 (변화 없음) |
| 예외 메시지 문자열 | 변경 없음 | 변경 없음 |

### 2부: 에러코드 상수화 (이슈 3)

#### 변경 후 형태

46개 필드에 `final`을 붙이고, 1부에서 넣은 NOTE 주석 2줄을 삭제한다. 값과 이름은 그대로다.

```java
    public static final int unknown = -21100;
    ...                                     // 46개 전부, 값 변경 없음
    public static final int invalid_savepoint = -21145;

    private static final Map<Integer, String> messageString = createMessageMap();
    // NOTE 주석 2줄 삭제: 선언 순서 제약이 사라졌으므로 남겨두면 틀린 주석이 된다
```

#### 변경 항목

| 항목 | 1부 이후 | 2부 이후 |
|---|---|---|
| 코드 필드 | `public static int` (외부에서 쓰기 가능) | `public static final int` (쓰기 불가) |
| `createMessageMap()`의 필드 접근 | `getstatic` 46회 | 인라인 상수 적재, `getstatic` 0회 |
| `<clinit>` | `putstatic` 47회 | `putstatic` 1회 (`messageString`만) |
| 선언 순서 제약 | 있음 (NOTE 주석이 보호) | **없음** |
| NOTE 주석 | 있음 | **삭제** |

#### 관측 가능한 동작 변화

드라이버 내부에는 없다. 소스 트리를 두 벌 만들어 46개 `(이름, 값, 메시지)` 삼중쌍, `messageString` 맵 전수(46 엔트리), 맵 크기와 클래스와 해시코드, 불변성, 경계값 8종, 46개 코드 전부에 대한 `CUBRIDException` 생성 결과까지 153줄을 전후 비교해 완전 일치를 확인했다. 드라이버 112개 클래스 전부가 `final` 상태로 경고 없이 컴파일된다(이 자체가 대입하는 코드가 없다는 증거다).

외부에는 두 가지 변화가 있으며, 둘 다 의도된 것이다. 리플렉션 쓰기(`setAccessible(true)` + `Field.setInt`)가 `IllegalAccessException`이 되고, 미리 컴파일된 소비자의 읽기가 인라인 값으로 고정된다.

#### 하지 않기로 한 것

| 항목 | 사유 |
|---|---|
| `containsKey(0)` 가드 | 상수화하면 필드를 런타임에 읽지 않으므로 함정이 발동하지 않는다. 실제 파일로 확인했다. 이론적으로는 blank final이나 비상수 초기화자가 남지만, 46줄이 모두 동일한 리터럴 형태인 파일에서 47번째만 그렇게 쓸 이유가 없어 현실적 위험이 아니다 |
| 맵 크기 검사 | 매직 넘버 46을 손으로 유지해야 하고, 코드 추가 시 갱신을 잊으면 가드 자체가 오작동한다 |
| 값 동결 정책 문서화 | 별도로 다룬다. 동결은 매뉴얼과 `getErrorCode()`로 이미 사실상 존재한다 |

### 위험 점검 (1부, 2부 공통)

| 위험 | 판단 |
|---|---|
| 정적 초기화자 실행 순서 | **2부로 해소됨.** 1부 단계에서는 코드 필드가 맵 초기화보다 텍스트상 위에 있어야 했고 NOTE 주석이 그것을 지켰다. 2부에서 46개가 컴파일 타임 상수가 되어 `getstatic`이 사라지면서 제약 자체가 없어졌고, NOTE도 함께 삭제한다 |
| 정적 초기화 중 예외 | `ExceptionInInitializerError`로 클래스가 영구 사용 불가가 된다. 여기서는 `Map.put`만 하고 null 키와 null 값이 없음을 확인했다 |
| `unmodifiableMap` 회귀 | 이후 누가 맵을 변경하려 하면 `UnsupportedOperationException`으로 즉시 드러난다. 의도된 동작이다 |
| 커버리지 누락 | 46/46 일치를 확인했다 |

### 대안 검토와 기각 사유 (1부)

| 대안 | 기각 사유 |
|---|---|
| `getMessage()`에 `synchronized` 추가 | diff는 가장 작지만, 유지할 이유가 없는 지연 초기화를 살린 채 조회마다 락 비용만 남는다 |
| `volatile` + 안전 공개 (로컬에서 채운 뒤 대입) | 정석이지만 코드가 늘고, 마찬가지로 불필요한 지연을 유지한다 |
| 자료구조만 `ConcurrentHashMap`으로 교체 | (1)절 실측대로 결함이 그대로 남는다. 원인이 자료구조가 아니기 때문이다 |
| `Map` 제거 후 `switch` 문 | 46개 분기로 전면 재작성이 필요해 변경 표면이 과도하다 |
| 정적 블록을 클래스 맨 아래 배치 | 필드 선언 순서 함정에는 더 강하지만, 위에 선언된 필드를 아래 블록에서 대입하는 형태라 가독성이 떨어진다. 주석으로 대체 |

## 검증 계획

레이스는 "수정 후 통과"가 증명이 되지 못하므로, 수정 전에 재현되던 것이 수정 후 사라지는 **전후 대조**로 검증한다.

| 단계 | 방법 | 기대 |
|---|---|---|
| 1 | 동시 최초 호출 재현 도구, 20회 x 50 스레드 | 수정 전 111/1000 → **0/1000** |
| 2 | TC 동형 재현 (20 스레드가 실제 접속 후 미지원 메서드 호출) | 수정 전 12/12 실행에서 발생 → **0** |
| 3 | 바이트코드 확인 | `<clinit>`에 `put` 46건, `getMessage`에서 분기 소멸, 조회 경로에 `monitorenter` 없음 |
| 4 | 불변성 확인 | 리플렉션으로 `put` 시도 시 `UnsupportedOperationException` |
| 5 | CTP JDBC 스위트 | 회귀 없음 |

2부(상수화) 추가 검증:

| 단계 | 방법 | 기대 |
|---|---|---|
| 6 | 46개 필드가 모두 `public static final int` + 리터럴인지 확인 | 46/46, 비상수 초기화자 0건 |
| 7 | `createMessageMap()` 바이트코드 | `getstatic` 0, 인라인 적재로 대체 |
| 8 | `<clinit>` 바이트코드 | `putstatic` 1회 (`messageString`만) |
| 9 | 46개 `(이름, 값, 메시지)` 삼중쌍 전후 비교 | 완전 일치 |
| 10 | 드라이버 전체 컴파일 | 112개 클래스 경고 없이 성공(대입 코드 부재 증명) |
| 11 | 선언 순서 무관 확인 | 47번째 코드를 맵 초기화 아래에 선언해도 정상 조회 |

### 단위 테스트를 추가하지 않는 이유

근거는 세 가지이며, 무게 순서대로다.

1. **드라이버 저장소에는 테스트 소스 트리도, `build.xml`의 junit 타깃도 없다.** 테스트를 넣을 곳 자체가 없으므로, 이 결함 하나를 위해 테스트 인프라를 신설하는 것은 균형이 맞지 않는다.
2. 이 레이스는 "클래스가 아직 로드되지 않았을 것"이 전제라 **JVM당 단 한 번만 유효**하다. 일반적인 JUnit 스위트에 넣으면 다른 테스트가 먼저 클래스를 로드하는 순간 무력화된다. 다만 이것이 "테스트 불가능"을 뜻하지는 않는다. 반복마다 새 `URLClassLoader`로 클래스를 다시 로드하면 전제를 재현할 수 있다. 즉 기술적 불가능이 아니라 비용 대비 효용의 문제다.
3. **앞으로 현실적인 회귀는 레이스가 아니라 선언 순서 함정이다.** 새 에러코드를 맵 초기화 줄 아래에 선언하면 키가 0으로 등록되어 조용히 `null`이 나온다. 이건 스레드 테스트로 잡히지 않으며, 상수화 이슈(이슈 3)가 근본에서 제거하거나 초기화 시 `m.containsKey(0)` 가드로 잡는 편이 맞다.

따라서 검증은 저장소 밖 재현 프로그램의 전후 대조로 하고, 그 프로그램을 이슈에 첨부해 절차로 남긴다.

## 다음 단계

- 이슈 1: 본 설계 구현 (`CUBRIDJDBCErrorCode`). **구현 완료**. 브랜치 `apis-errorcode-thread-safety`(upstream/develop 기준), 커밋 1건. 실측: 수정 전 92/1000 → 수정 후 0/1000, 실브로커 20스레드 TC 동형 재현 12/12 전부 정상. 전체 브랜치 리뷰 통과(Critical/Important 0건)
- 이슈 2: `UErrorCode.codeToMessage` / `codeToCASMessage` 2건에 동일 수정 적용. `UError.getErrorMsg`가 서버 에러마다 호출하고 외부 호출처가 29곳이라 노출은 오히려 더 크다
- 이슈 3: 에러코드 필드의 `public static final` 전환. **진행 확정**(위 2부). 값 동결 정책 문서화는 범위에서 제외했다. 이슈 1과 같은 브랜치에 별도 커밋으로 쌓는다. 이슈 3이 이슈 1의 NOTE 주석을 다시 삭제하는 관계라 한 브랜치로 묶는 편이 리뷰에 유리하고, 백포트 단위를 쪼개고 싶으면 커밋이 분리돼 있어 cherry-pick으로 가능하다
- 이슈 1 반영 후 관련 TC의 기대 결과 파일 갱신 (기대 결과 갱신은 반드시 본 수정 이후에 해야 한다. 그러지 않으면 영구 flaky TC가 된다)

## 참고
- JLS 12.4.2 (클래스 초기화 절차와 락), JLS 17.5 (final 필드 의미론), JLS 4.12.4 / 13.1 (컴파일 타임 상수와 인라인)
- 원인 분석 노트: [APIS-1090이 드러낸 에러 메시지 테이블 레이스](../bug/2026-08-19-apis1090-not-supported-message-race.md)

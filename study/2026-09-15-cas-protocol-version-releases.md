# CAS 프로토콜 버전과 릴리스 대응표

- 분류: study
- 날짜: 2026-09-15
- 관련: JDBC 메타데이터 값을 서버 버전에 따라 다르게 답해야 하는지 판단하기 위한 기준표

## 요약

CAS 프로토콜 버전은 릴리스 경계와 정확히 맞아떨어지므로, 드라이버가 서버 기능을 판별할 때 버전 문자열 대신 프로토콜 번호를 쓰면 왕복 없이 정확하다.

## 목적

JDBC 드라이버가 "이 서버가 이 기능을 지원하는가"를 판별할 때 쓸 기준을 정한다. `getDatabaseProductVersion()` 은 서버 왕복이 필요하지만 프로토콜 버전은 접속 시점에 이미 받아 둔 값이라 비용이 없다. 다만 그러려면 프로토콜 번호와 릴리스가 어떻게 대응하는지 확정돼 있어야 한다.

## 배경

CUBRID 11.2부터 `schema.table` 형태를 지원한다. 그런데 JDBC 드라이버의 `supportsSchemasInDataManipulation` 계열은 서버 버전과 무관하게 고정값을 답한다. 드라이버가 지원하는 서버 범위에는 11.2 미만도 들어 있으므로, 고정값을 어느 쪽으로 바꾸든 절반은 틀린 답이 된다.

드라이버에는 이미 프로토콜 버전으로 기능을 켜고 끄는 관례가 있다. 예를 들어 홀더블 결과셋은 `protocolVersion >= PROTOCOL_V12` 로 판별한다.

## 범위 / 방법

- 정의 위치: 엔진 저장소 `src/broker/cas_protocol.h` 의 `enum t_cas_protocol`
- 각 버전이 도입된 커밋을 `git log -S` 로 찾고, 그 커밋을 포함하는 최초 릴리스 태그를 조회
- 경계 검증: 직전 릴리스 계열 태그가 해당 커밋을 포함하지 않는지 `git merge-base --is-ancestor` 로 확인

## 발견 / 관찰

### 대응표

| 프로토콜 | 도입일 | 최초 릴리스 | 내용 |
|---|---|---|---|
| `PROTOCOL_V0` | 2012-07-04 | 10.0 | 구 프로토콜 |
| `PROTOCOL_V1` | 2012-07-04 | 10.0 | query timeout, query cancel |
| `PROTOCOL_V2` | 2012-07-04 | 10.0 | 실행 결과에 컬럼 메타데이터 동봉 |
| `PROTOCOL_V3` | 2012-11-02 | 10.0 | 서버 세션 키로 세션 정보 확장 |
| `PROTOCOL_V4` | 2012-12-17 | 10.0 | as_index 전달 |
| `PROTOCOL_V5` | 2013-03-26 | 10.0 | 샤드, fetch end 플래그 |
| `PROTOCOL_V6` | 2014-06-27 | 10.0 | unsigned integer 타입 |
| `PROTOCOL_V7` | 2015-02-09 | 10.0 | 타임존 타입, XASL 항목 고정 |
| `PROTOCOL_V8` | 2018-10-25 | **10.2** | JSON 타입 |
| `PROTOCOL_V9` | 2020-04-20 | **11.0** | CAS 헬스 체크 |
| `PROTOCOL_V10` | 2020-08-04 | **11.0** | SSL 브로커/CAS |
| `PROTOCOL_V11` | 2022-04-06 | **11.2** | out resultset |
| `PROTOCOL_V12` | 2023-08-30 | **11.3** | double/float 후행 0 제거 |

`CURRENT_PROTOCOL = PROTOCOL_V12` 이고, JDBC 드라이버도 `CAS_PROTOCOL_VERSION = PROTOCOL_V12` 를 보낸다.

### 경계는 정확하다

V11 이 11.2 경계와 어긋나지 않는지 확인했다. 11.0 과 11.1 계열 태그 중 V11 도입 커밋을 포함하는 것은 없다.

```
V11 커밋을 포함하는 최초 태그들: 11.2, 11.2.1, 11.2.2, 11.2.3, ...
v11.0 / v11.0.1 / v11.0.10 ... : 전부 미포함
```

따라서 `protocolVersion >= PROTOCOL_V11` 은 "서버가 11.2 이상"과 같은 뜻이다.

### 드라이버가 받는 하한

드라이버는 접속 시점에 V8 미만 서버를 거부한다.

```java
/* The driver only supports servers using PROTOCOL_V8 or later. */
if (protocolVersion < PROTOCOL_V8) { ... }
```

즉 드라이버가 실제로 마주치는 서버 범위는 **10.2 이상**이다. 10.2, 11.0, 11.1, 11.2, 11.3, 11.4 가 전부 지원 대상이므로 기능 판별이 필요하다.

### 두 판별 수단의 비용

| 수단 | 왕복 | 비고 |
|---|---|---|
| `protocolVersion` | 없음 | 접속 핸드셰이크에서 이미 받음 |
| `getDatabaseProductVersion()` | 1회 | `GET_DB_VERSION` 요청, 파싱 결과는 인스턴스에 캐시 |

프로토콜 번호가 릴리스 경계와 맞는 기능이라면 전자를 쓰는 편이 낫고, 이미 드라이버 안에 그 관례가 있다.

## 결론

`schema.table` 지원 판별은 `protocolVersion >= PROTOCOL_V11` 로 표현할 수 있고, 이는 "11.2 이상"과 정확히 같다. 왕복이 없고 기존 기능 게이트 관례와도 일치한다.

다만 모든 기능이 프로토콜 경계와 맞는 것은 아니다. 프로토콜을 올리지 않고 들어간 기능은 이 표로 판별할 수 없고, 그때는 버전 문자열을 봐야 한다. 기능마다 어느 쪽인지 먼저 확인하고 수단을 고르는 것이 맞다.

## 다음 단계

- `supportsSchemasIn*` 계열을 `PROTOCOL_V11` 게이트로 바꾸는 작업을 이슈로 등록한다
- 같은 스키마 계열인 `getSchemas()`, `getSchemaTerm()` 도 함께 본다. 값 하나씩 고치면 조합이 여전히 앞뒤가 안 맞는다
- 프로토콜을 올리지 않은 기능이 있는지, 즉 이 표로 판별 못 하는 경우가 무엇인지 별도로 정리한다

## 참고

- 엔진 정의: `src/broker/cas_protocol.h` 의 `enum t_cas_protocol`
- 드라이버 상수: `src/jdbc/cubrid/jdbc/jci/UConnection.java` 의 `PROTOCOL_V*`, `CAS_PROTOCOL_VERSION`

# CAS 프로토콜 버전과 릴리스 대응표

- 분류: study
- 날짜: 2026-09-15
- 관련: JDBC 메타데이터 값을 서버 버전에 따라 다르게 답해야 하는지 판단하기 위한 기준표

## 요약

프로토콜 번호는 기능 게이트로 쓸 수 있지만 릴리스를 역산하는 데는 쓸 수 없다. 유지보수 라인으로 백포트되기 때문에 10.2 서버도 V10 을 답한다. 다만 V11 경계는 11.2 와 정확히 일치하므로 `schema.table` 판별에는 그대로 쓸 수 있다.

## 목적

JDBC 드라이버가 "이 서버가 이 기능을 지원하는가"를 판별할 때 쓸 기준을 정한다. `getDatabaseProductVersion()` 은 서버 왕복이 필요하지만 프로토콜 버전은 접속 시점에 이미 받아 둔 값이라 비용이 없다. 다만 그러려면 프로토콜 번호와 릴리스가 어떻게 대응하는지 확정돼 있어야 한다.

## 배경

CUBRID 11.2부터 `schema.table` 형태를 지원한다. 그런데 JDBC 드라이버의 `supportsSchemasInDataManipulation` 계열은 서버 버전과 무관하게 고정값을 답한다. 드라이버가 지원하는 서버 범위에는 11.2 미만도 들어 있으므로, 고정값을 어느 쪽으로 바꾸든 절반은 틀린 답이 된다.

드라이버에는 이미 프로토콜 버전으로 기능을 켜고 끄는 관례가 있다. 예를 들어 OUT 결과셋 번호의 폭(4바이트/8바이트)은 `brokerProtocolVersion() < PROTOCOL_V11` 로 가른다. (홀더블 결과셋은 번호가 아니라 브로커 정보의 기능 비트 0x40 으로 판별한다.)

## 범위 / 방법

- 정의 위치: 엔진 저장소 `src/broker/cas_protocol.h` 의 `enum t_cas_protocol`
- 도입 시점: 각 버전이 들어온 커밋을 `git log -S` 로 찾고 그 커밋을 포함하는 최초 릴리스 태그를 조회
- 릴리스가 실제로 답하는 번호: 각 릴리스 계열 최신 태그에서 `git show <tag>:src/broker/cas_protocol.h` 로 `CURRENT_PROTOCOL` 을 직접 읽음 (백포트 때문에 커밋 조상 판정으로는 안 된다. 아래 "검증 방법 주의" 참고)
- 교차 확인: 10.2 / 11.0 / 11.4 서버를 실제로 띄워 드라이버가 받은 값과 대조

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

여기서 "최초 릴리스"는 그 번호가 **처음 등장한** 릴리스이지, 그 릴리스가 답하는 번호가 아니다. 뒤에 나오듯 V9 와 V10 은 10.2 와 11.0/11.1 유지보수 라인으로 백포트되었다.

### 릴리스가 실제로 답하는 번호

각 릴리스 계열 최신 태그에서 `cas_protocol.h` 를 직접 읽었다.

| 릴리스 계열 최신 태그 | `CURRENT_PROTOCOL` |
|---|---|
| v10.2.18 | `PROTOCOL_V10` |
| v11.0.16 | `PROTOCOL_V10` |
| v11.1.0.0441 | `PROTOCOL_V10` |
| v11.2.9.0868 | `PROTOCOL_V11` |
| v11.3.5.1280 | `PROTOCOL_V12` |
| v11.4.6.1963 | `PROTOCOL_V12` |

실제 서버에 접속해서도 같은 값이 나왔다. 10.2.18.9024 → 10, 11.0.16.0419 → 10, 11.4.6.1963 → 12. 드라이버는 브로커가 보낸 바이트를 그대로 읽을 뿐 협상하지 않는다(`UClientSideConnection` 의 `protocolVersion = version & CAS_PROTO_VER_MASK`).

즉 **V8 은 10.2 를 뜻하지 않는다.** V9(헬스 체크)와 V10(SSL)이 10.2 와 11.0/11.1 유지보수 라인에 들어갔기 때문에 이 세 계열은 전부 V10 을 답한다. 프로토콜 번호로 릴리스를 역산할 수 없다.

반면 **V11 경계는 정확하다.** 10.2 / 11.0 / 11.1 계열 최신 태그가 모두 V10 에서 멈추고 11.2 부터 V11 이다. 따라서 `protocolVersion >= PROTOCOL_V11` 은 "서버가 11.2 이상"과 같은 뜻이다.

### 검증 방법 주의

처음에는 도입 커밋의 SHA 가 각 태그의 조상인지(`git merge-base --is-ancestor`) 로 확인했는데, 이 방법은 **백포트를 놓친다.** 유지보수 라인의 백포트는 체리픽이라 SHA 가 달라 조상 판정이 거짓이 된다. 실제로 이 방법으로는 10.2 계열에 V10 이 없다고 나오지만, `git show v10.2.18:src/broker/cas_protocol.h` 는 `CURRENT_PROTOCOL = PROTOCOL_V10` 을 보여준다. 기능 포함 여부는 해당 태그의 **파일을 직접 읽어서** 판단해야 한다.

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

다만 이것은 V11 에 한한 이야기다. 프로토콜 번호 일반으로는 릴리스를 역산할 수 없다(V9, V10 은 백포트됨). 게이트를 새로 만들 때마다 그 번호의 경계를 위 방법으로 직접 확인해야 한다.

또한 모든 기능이 프로토콜 경계와 맞는 것은 아니다. 프로토콜을 올리지 않고 들어간 기능은 이 표로 판별할 수 없고, 그때는 버전 문자열을 봐야 한다. 기능마다 어느 쪽인지 먼저 확인하고 수단을 고르는 것이 맞다.

## 다음 단계

- `supportsSchemasIn*` 계열을 `PROTOCOL_V11` 게이트로 바꾸는 작업을 이슈로 등록한다
- 같은 스키마 계열인 `getSchemas()`, `getSchemaTerm()` 도 함께 본다. 값 하나씩 고치면 조합이 여전히 앞뒤가 안 맞는다
- 프로토콜을 올리지 않은 기능이 있는지, 즉 이 표로 판별 못 하는 경우가 무엇인지 별도로 정리한다 (정리함: [CAS 프로토콜 버전별 기능](2026-09-28-cas-protocol-version-features.md))

## 참고

- 엔진 정의: `src/broker/cas_protocol.h` 의 `enum t_cas_protocol`
- 드라이버 상수: `src/jdbc/cubrid/jdbc/jci/UConnection.java` 의 `PROTOCOL_V*`, `CAS_PROTOCOL_VERSION`
- 관련 노트: [CAS 프로토콜 버전별 기능과, 번호 없이 늘어난 것들](2026-09-28-cas-protocol-version-features.md)

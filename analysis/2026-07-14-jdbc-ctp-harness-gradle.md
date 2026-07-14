# JDBC 테스트 하네스 모던화 — CTP 위 Gradle 공존 하네스 (제안)

- 분류: analysis
- 날짜: 2026-07-14
- 관련: `2026-07-14-cas-mock-fake-broker.md`(isValid mock 검토), CTP `JdbcLocalTest` 실행 모델

## 요약
CTP JDBC 테스트 하네스는 단일 JVM·per-test 타임아웃 없음·손실 리포팅이라, 다른 드라이버(pgjdbc·MySQL·MariaDB)처럼 공유 부작용을 막아주는 안전장치가 없다. 조사 결과 **JDBC 전용으로, 기존 CTP 경로와 공존하는 Gradle 하네스를 추가**하는 것은 실현 가능하고 위험도 낮다. 근거는 두 가지 — (1) `test_jdbc` 테스트 소스가 CTP에 전혀 의존하지 않아 러너/스크립트 계층만 바꾸면 되고, (2) 사내에 이미 Gradle+JUnit5 선례(`pl_engine`)가 있다. Maven이 아니라 **Gradle**이 맞다(저장소에 Maven 부재). 단계적(공존→등가검증→선택적 전환)으로 진행하면 일일 CI를 흔들지 않고 도입할 수 있으며, 첫 수혜는 isValid 죽은-커넥션 감지 같은 **장애 주입 TC가 per-test 타임아웃·fork 격리를 얻는 것**이다. 단, CTP/`test_jdbc`·일일 파이프라인은 QA팀 소유이므로 합의가 전제다.

## 목적
CTP JDBC 테스트 하네스의 구조적 약점을 다른 JDBC 드라이버 수준으로 보강할 수 있는지, 어떤 방식(Gradle/Maven)으로, 어느 정도 위험으로, JDBC에만 스코프해서 가능한지를 판단한다.

## 배경
- CTP의 JDBC 실행 러너 `JdbcLocalTest`는 **JUnit4 전용·단일 JVM·per-test 타임아웃 없음**이고, 실패는 첫 메시지만 기록하며, 컴파일된 `.class`를 스캔해 메서드 단위로 실행한다.
- 실 DBMS를 스위트 전체가 공유(TC별 환경 격리 아님)하는 것은 pgjdbc·MySQL·MariaDB도 동일한 **업계 공통 관행**이다. 문제는 격리 자체가 아니라, 공유의 부작용(클라이언트 JVM 상태 누수, 한 TC의 hang이 전체 정지)을 막아주는 하네스 장치(**JVM 포크·per-test 타임아웃·풀 리포팅**)가 CTP에는 없다는 점이다.
- 이 약점은 앞선 `cas-mock` 검토의 결론과 직결된다 — isValid MockBroker/프록시 같은 장애 주입 TC는 정확히 이 부재(타임아웃·격리) 때문에 CTP에서 위험하다.

## 범위 / 방법
- CTP 카테고리 구조와 **blast radius**: JDBC 경로가 공유 인프라인지 전용인지.
- `test_jdbc`의 **CTP 결합도**와 표준 빌드로의 **이관 표면**.
- 사내 **모던 하네스 선례**(`pl_engine`의 Gradle+JUnit5) 재사용성.
- (모두 실제 소스 대조로 확인; 근거는 참고 절 file:line.)

## 발견 / 관찰
1. **CTP는 카테고리별로 잘 분리돼 있다.** ~20개 테스트 카테고리(SQL/SHELL/ISOLATION/JDBC…)가 각자 전용 러너·conf·CI 래퍼로 1:1 구성된다. JDBC 전용 요소는 `jdbc/bin/run.sh`, `JdbcLocalTest`(호출처는 `run.sh` 단 한 곳), `conf/jdbc.conf`, CI 래퍼 `common/ext/run_jdbc.sh`뿐이다. → **공유 코어(`ctp.sh`·`CTP.java`·`common/`)나 다른 카테고리를 건드리지 않고** JDBC 실행 경로만 교체할 수 있다. `-c`로 커스텀 conf 주입, `jdbc_template.conf` 오버라이드 슬롯도 이미 지원된다.
2. **`test_jdbc` 테스트 소스는 CTP에 비의존.** `src` 전역에서 CTP(`com.navercorp.cubridqa`) 참조 0건. 결합은 전적으로 러너/스크립트 계층에만 있다. 게다가 `build.xml`에 이미 표준 `<junit>` 타깃이 있어(현재 CTP 실행 경로는 이걸 쓰지 않지만) 표준 빌드로의 이행 기반이 존재한다.
3. **이관 표면 ~376개 파일. 반드시 손볼 결합점 4개:**
   - ① `jdbc.properties` 로딩이 실행 디렉터리(CWD) 의존 + CTP가 `sed`로 접속정보 주입 → 시스템 프로퍼티/클래스패스 리소스 방식으로 전환.
   - ② `cubrid_jdbc.jar`를 CTP가 런타임에 설치본으로 교체 → Gradle 파일 의존성/프로퍼티 주입(`pl_engine`의 `-PcubridJdbcPath` 방식).
   - ③ **split-package**: `package cubrid.jdbc.*`로 선언된 테스트 98개(그중 `cubrid.jdbc.jci` 30개)가 드라이버 package-private 심볼에 접근 → 동일 패키지 유지 + `--add-opens`.
   - ④ **PowerMock 8·EasyMock 8·Mockito 2**(구버전) → JUnit vintage 엔진으로 무수정 실행(전면 재작성 회피). JUnit3(`extends TestCase`) 41개·JUnit4 152개 혼재도 vintage로 함께 수용.
4. **사내 선례 `pl_engine`** = Gradle 8.1.1(wrapper 커밋) + JUnit Jupiter 5.9.1 + 우수한 로깅/요약 리스너 + CMake↔Gradle 연동 + 로컬 JDBC jar 주입. 뼈대로 즉시 재사용 가능. **단 fork/병렬·타임아웃·접속 파라미터 주입은 없어**, JDBC 통합테스트용으로 이 3가지를 신규 추가해야 한다.

## 결론
**JDBC 전용 Gradle "공존형" 하네스**를 권장한다.
- **표준**: Gradle(사내 일관성; Maven은 저장소에 전무).
- **핵심 설계**: `useJUnitPlatform()` + **vintage 엔진**으로 기존 JUnit3/4·PowerMock을 무수정 실행하고, CTP에 없던 **① per-test 타임아웃 ② fork 격리(`forkEvery` 등) ③ 접속 파라미터 주입**을 새로 넣는다. 신규 테스트만 Jupiter로 작성한다(전면 JUnit5 재작성은 비권장).
- **단계적 도입**:
  - Phase 0 — 공존(additive): `test_jdbc`에 Gradle 빌드 추가. 기존 CTP 경로는 그대로 병행. (위험 낮음)
  - Phase 1 — 등가 검증: 같은 실 브로커로 돌려 CTP 기준선(현행 통과/실패 집합)과 일치 확인. 집계 단위가 메서드→클래스로 바뀌므로 리포트 재기준선화 필요.
  - Phase 2 — (선택) 전환: `run_jdbc.sh`/`run.sh`가 `./gradlew test`를 부르게 소폭 변경(롤백 용이).
- **위험 판단**: 스코프를 JDBC로 한정하고 additive로 하면 낮다(blast radius = JDBC 전용 요소 + 선택적으로 `CTP.java`의 executeJdbc 한 메서드). big-bang 교체나 JUnit5 전면 재작성은 위험·비권장.
- **직접 이득**: isValid MockBroker/프록시 등 장애 주입 TC가 per-test 타임아웃·fork 격리를 즉시 확보 — CTP에서 위험했던 바로 그 부분이 해소된다.
- **전제/비용**: CTP·`test_jdbc`·일일 파이프라인은 QA팀 소유 → JDBC 스코프라도 **합의 필요**. 구버전 목킹 스택(PowerMock 1.4.12 등)은 기술부채로 남는다. 376파일 규모의 실작업.

## 다음 단계
- 이 제안서를 근거로 **QA팀 논의/합의**.
- 합의 시 Phase 0 구현 플랜 수립 + 결합점 4개(properties 로딩·jar 주입·split-package·vintage) **PoC** 선행.
- 첫 적용 케이스로 isValid 죽은-커넥션 감지 TC(프록시 방식)를 새 하네스 위에서 작성 — 하네스와 TC를 함께 검증.
- 이슈화: "JDBC 모던 하네스 도입"과 "isValid TC 추가"는 성격이 달라 별도 태스크로 분리.

## 참고
- CTP 구조(blast radius): `cubrid-testtools/CTP/common/src/com/navercorp/cubridqa/ctp/CTP.java`(`executeJdbc`), `jdbc/bin/run.sh`(DB준비→javac→러너), `shell/src/com/navercorp/cubridqa/shell/main/JdbcLocalTest.java`(JUnit4·단일JVM·타임아웃 없음), `conf/jdbc.conf`, `common/ext/run_jdbc.sh`(CI 래퍼)
- test_jdbc 결합/이관: `interface/JDBC/test_jdbc/build.xml`(`<junit>` 타깃), `.../PropertiesUtil.java`(CWD 의존 로딩), `.../ConnectionProvider.java`, `jdbc.properties`(CTP sed 주입 대상)
- 사내 선례: `cubrid/pl_engine/pl_server/build.gradle.kts`(Jupiter 5.9.1·`useJUnitPlatform()`·리포팅 리스너), `cubrid/pl_engine/gradle/wrapper/`(Gradle 8.1.1, 커밋됨), `cubrid/pl_engine/CMakeLists.txt`(`pl_unittest` 연동), `-PcubridJdbcPath` 로컬 jar 주입
- 타 드라이버 가짜/프록시 선례 및 배경: `2026-07-14-cas-mock-fake-broker.md`

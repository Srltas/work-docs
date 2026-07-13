# CMT MySQL·MariaDB ENUM/SET/JSON → CUBRID 이관 이슈

- 분류: cmt_bug
- 날짜: 2026-07-13
- 관련: CMT e2e MySQL·MariaDB 소스 추가 작업 중 발견 (tests/e2e)

## 요약
ENUM/SET/JSON 컬럼이 있는 테이블을 CUBRID로 이관할 때, MySQL은 CMT가 데이터 추출 SQL을 잘못 생성해 테이블 전체가 실패하고, MariaDB는 스키마는 이관되나 데이터 행 import에서 실패한다. 두 엔진 모두 마이그레이션이 FAILED로 끝나 e2e 시드에서 제외했다.

## 목적
MySQL·MariaDB 소스의 ENUM/SET/JSON 타입 이관 동작 차이와 실패 지점을 기록한다.

## 배경
CMT e2e에 MySQL·MariaDB 소스의 타입 배터리를 구성하는 과정에서, MySQL 계열 고유 타입(ENUM/SET/JSON)을 담은 확장 테이블이 이관에 실패했다. MariaDB는 CMT 플러그인이 MySQL과 별개 구현이라 동작을 별도로 확인했다.

## 범위 / 방법
- 환경: CMT console 12.0.0, 소스 MySQL 8.0 / MariaDB 11, 타깃 CUBRID 11.4 online
- ENUM/SET/JSON 3개 컬럼 + 데이터 3행을 담은 프로브 테이블로 각 엔진에서 online 이관 실행
- 마이그레이션 리포트(table/record 카운트)와 타깃 카탈로그 컬럼 매핑 확인

## 발견 / 관찰
- 엔진별 동작 차이:

| 엔진 | 스키마(테이블/컬럼) | 데이터 | 결과 |
|---|---|---|---|
| MySQL | 실패 (레코드 추출 SELECT가 `near FROM` 구문 오류) | 미이관 | FAILED |
| MariaDB | 이관됨 (enum → CUBRID `ENUM`, set → `SET`, json → `STRING`) | 행 import 실패 (record Exported[39] / Imported[36], 3행 손실) | FAILED |

- MySQL: `JDBCExporter`가 ENUM/SET/JSON 컬럼이 포함된 테이블의 레코드 추출 SELECT를 잘못 생성해 `SQLSyntaxErrorException (near 'FROM ...')`로 추출 자체가 실패한다. 스키마도 데이터도 넘어가지 못한다.
- MariaDB: CUBRID에 ENUM/SET 타입이 있고 JSON은 STRING으로 매핑되어 스키마 생성까지는 성공한다. 그러나 해당 테이블 데이터 행이 타깃에 import되지 않아(record 손실) 전체가 FAILED. 뷰 이관 실패와 마찬가지로 CMT가 상세 에러를 표출하지 않아 리포트 카운트로만 확인된다.

## 결론
ENUM/SET/JSON은 두 엔진 모두 현재 CMT로 CUBRID online 이관이 완결되지 않는다. MySQL은 스키마 추출 단계에서, MariaDB는 데이터 import 단계에서 실패한다. MariaDB가 스키마까지는 처리한다는 점에서 더 진전되어 있으나, 데이터 손실로 마이그레이션이 실패하는 것은 동일하다. 따라서 e2e 시드에서는 두 엔진 모두 이 타입들을 제외했다.

## 다음 단계
- CMT JIRA 등록 후보 2건: (1) MySQL ENUM/SET/JSON 레코드 추출 SQL 생성 오류, (2) MariaDB ENUM/SET/JSON 데이터 import 실패
- MariaDB 데이터 import 실패의 구체 원인(값 변환/제약 위반 등)은 추가 조사 필요
- e2e는 현행처럼 ENUM/SET/JSON 제외 유지

## 참고
- CMT e2e: tests/e2e (`MySqlToCubridTest`, `MariaDbToCubridTest`)
- online 뷰 이관 실패는 별도 노트 참조 (2026-07-12-mysql-online-view-migration)

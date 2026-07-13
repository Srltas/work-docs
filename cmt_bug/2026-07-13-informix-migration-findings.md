# Informix → CUBRID 이관 발견 사항 (인덱스 방향, FK 지원 인덱스, 타입 매핑)

- 분류: cmt_bug
- 날짜: 2026-07-13
- 관련: CMT e2e 테스트에 Informix 소스 추가 작업 중 발견

## 요약
Informix 소스를 CUBRID로 이관할 때 CMT가 (1) DESC 인덱스의 내림차순 방향을 잃어버리고(ASC로 변환), (2) FK 지원 인덱스를 CUBRID FK와 별개의 일반 인덱스로 중복 이관하며 그 과정에서 Informix 시스템 자동 인덱스 명이 그대로 노출된다.

## 목적
CMT e2e 테스트에 Informix 소스를 추가하면서 생성된 스냅샷(online 카탈로그, unload 덤프)을 검토해, CMT의 Informix 플러그인이 다른 소스(Oracle, MySQL) 대비 다르게 동작하는 지점을 식별한다. 마이그레이션 자체는 성공(exit 0)하므로 실패가 아닌 결과물 품질 관점의 발견이다.

## 배경
CMT e2e 스위트는 소스 DB를 CUBRID online 및 unload(LoadDB) 덤프로 이관한 뒤, 결과 카탈로그를 골든 스냅샷과 비교한다. Oracle/MySQL/MariaDB에 이어 Informix 소스를 추가했고, 모든 테스트는 green이지만 스냅샷 내용에서 아래 동작이 관찰되었다.

## 범위 / 방법
- 소스: Informix 14.10 개발자 에디션 이미지, 단일 데이터베이스(로그드), 일반 JDBC로 스키마/데이터 시딩.
- 타깃: CUBRID 11.4 (online), 그리고 CMT unload(LoadDB) 덤프.
- 테스트 빌드: CMT Console 12.0.0.0318.
- 비교 기준: 동일 구조의 Oracle/MySQL 소스 스냅샷(같은 비즈니스 그래프, 같은 이름의 DESC 인덱스 `idxd_...`).
- 인덱스 방향은 스냅샷만으로 단정하지 않고, Informix 시스템 카탈로그를 직접 확인해 소스가 실제로 DESC로 저장했는지 검증했다.

## 발견 / 관찰

### 1. DESC 인덱스 방향 소실 (버그)
소스에서 내림차순으로 만든 단일 컬럼 인덱스가 CUBRID로 이관되면 오름차순(ASC)이 된다.

Informix는 내림차순 방향을 실제로 저장한다(정규화하지 않음). 별도 컨테이너에서 확인:

```sql
-- 소스 Informix 카탈로그 확인
create index ixd on t (b desc) using btree;   -- dbschema 출력: "(b desc)"
-- sysindexes.part1 = -2  (컬럼 번호가 음수이면 내림차순)
-- 비교용 ASC 인덱스 ixa: part1 = 1
```

그러나 CMT 이관 결과는 online, unload 양쪽 모두 ASC이다:

```
-- online 타깃 (db_index/db_index_key)
idxd_e2e_order_ordered_at | ordered_at | key_order 0 | ASC

-- unload 덤프 (indexes DDL)
CREATE INDEX [idxd_e2e_order_ordered_at] ON [INFORMIX].[e2e_order]([ordered_at]);
```

같은 이름의 DESC 인덱스를 다른 소스에서는 방향까지 보존한다:

| 소스 | 동일 DESC 인덱스 이관 결과 |
|------|----------------------------|
| Oracle | DESC 보존 (추가로 함수 기반 인덱스도 보존) |
| MySQL | DESC 보존 |
| Informix | ASC로 변환 (방향 소실) |

CUBRID 및 CMT 코어는 DESC 인덱스를 표현할 수 있으므로(Oracle, MySQL 경로에서 보존됨), 이는 Informix 플러그인이 소스의 내림차순 속성을 읽지 못하거나 버리는 문제로 판단된다.

### 2. FK 지원 인덱스의 중복 이관 및 시스템 명 노출
Informix는 FK 제약을 만들 때 이를 지원하는 인덱스를 시스템 자동 명(`<tabid>_<n>` 형태)으로 생성한다. CMT는 이 지원 인덱스를 CUBRID FK와 별개의 일반 인덱스로도 이관한다. CUBRID FK 제약은 자체 인덱스를 생성하므로, 같은 컬럼에 인덱스가 2개 생기고 그중 하나는 의미 없는 Informix 시스템 명을 그대로 갖는다.

| 컬럼 | FK 인덱스 (is_foreign_key=YES) | 일반 인덱스 (is_foreign_key=NO) |
|------|-------------------------------|--------------------------------|
| e2e_order.customer_id | fk_e2e_order_customer | 101_8 |
| e2e_order_line.order_id | fk_e2e_order_line_order | 102_13 |
| e2e_employee.manager_id | fk_e2e_employee_manager | 103_18 |

일반 인덱스(`101_8` 등)의 대상 컬럼은 각 FK 컬럼과 동일하며, 이 컬럼에는 소스에서 그 외의 인덱스를 만든 적이 없다. 즉 위 일반 인덱스는 Informix가 FK를 위해 자동 생성한 지원 인덱스가 그대로 이관된 것이다.

참고로 같은 컬럼에 인덱스 2개가 생기는 현상 자체는 다른 소스에서도 나타날 수 있으나(FK 인덱스 + 사용자가 명시 생성한 인덱스), MySQL의 경우 일반 인덱스는 사용자가 이름을 붙여 의도적으로 만든 것이라 이름이 유의미하다. Informix에서 드러나는 문제는, 사용자가 만들지 않은 FK 지원 인덱스가 중복 이관되고 그 이름이 CUBRID로 새어 나온다는 점이다.

### 3. BOOLEAN → INTEGER 매핑 (관찰, 버그 아님)
Informix `BOOLEAN`은 CUBRID `INTEGER`로 매핑된다. CUBRID 11.4에는 네이티브 BOOLEAN 타입이 없으므로 정수 매핑은 타당하다. 다만 값 범위상 `SHORT`(SMALLINT)가 더 조밀하다는 여지는 있다.

### 4. 범위 밖 참고
- SPL 루틴, grant 미이관: Informix 스냅샷에서 함수/프로시저/grant는 비어 있으나 MySQL 소스도 동일하므로 Informix 고유 문제가 아니다.
- `DATETIME YEAR TO FRACTION(5)`는 CUBRID `DATETIME`(밀리초, scale 3)으로 축소된다. CUBRID DATETIME의 정밀도 상한상 예상된 결과다.

## 결론
- 발견 1(DESC 방향 소실)은 다른 소스와 비교했을 때 명확한 Informix 플러그인 결함으로 보이며, 이슈화 가치가 있다.
- 발견 2(FK 지원 인덱스 중복 및 시스템 명 노출)는 결과물 품질(불필요한 중복 인덱스, 비의미 인덱스 명) 개선 관점의 발견이다.
- 발견 3은 정보성 관찰로 조치 불필요.
- e2e 스냅샷은 위 현재 동작을 그대로 골든값으로 고정하여, 향후 CMT가 개선되면 스냅샷 diff로 감지되도록 해 두었다.

## 다음 단계
- 발견 1은 JIRA 버그로 등록 검토 (제목 예: Informix 소스의 DESC 인덱스가 CUBRID로 ASC로 이관됨).
- 발견 2는 개선 과제로 등록 검토 (FK 지원 인덱스 중복 이관 및 시스템 인덱스 명 노출).
- 등록 시 e2e 스냅샷 경로와 재현용 시드를 근거로 첨부.

## 참고
- CUBRID 11.4 데이터 타입(네이티브 BOOLEAN 부재 확인): https://www.cubrid.org/manual/en/11.4/sql/datatype.html
- 소스 방향 검증: Informix `dbschema` 출력 및 `sysindexes.part1`(음수 = 내림차순)

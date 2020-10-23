# 15. 사이트 내의 사용자 행동 파악하기
- 웹 사이트의 특징적인 지표, 리포트를 작성하는 SQL
  - 방문자 수, 방문 횟수, 직귀율, 이탈률
- **접근 분석 도구**를 사용하는 서비스라면, 위 지표를 이미 파악했을 것
- 하지만 **빅데이터 기반**으로 데이터 추출 시,
  - **사용자** 또는 **상품 데이터** 등의 각종 **업무 데이터**와 조합하여 상세히 분석 가능
- **직귀율**과 **이탈률**은 비슷해 보이나, 실제 지표의 정의가 다름
- 샘플 데이터 : 구인/구직 데이터
  - 사이트 맵이 정해져 있음
  - `검색 결과 목록` 페이지에서 `지역`과 `직종`을 조건으로 지정할 수 있음
  - `지역`은 여러 depth 가 있음
  - `직종`은 `인사`, `영업`, `기술` 지정 가능
  - `검색 결과 목록 페이지`에서 검색을 하면 로그가 저장됨

## 1. 입구 페이지와 출구 페이지 파악하기
- 개념
  - SQL : `FIRST_VALUE`, `LAST_VALUE`
  - 분석 : 입구페이지, 출구페이지
- 입구 페이지 : 사이트 방문시, **처음 접근한 페이지**
  - **랜딩 페이지**라고도 부름
- 출구 페이지 : 마지막 접근 페이지
  - **이탈 페이지**라고도 부름

### 입구 페이지와 출구 페이지 집계하기
- 접근 로그의 각 **세션**에서, 입구페이지와 출구페이지의 URL을 집계하려면
  - `FIRST_VALUE`, `LAST_VALUE`의 **윈도 함수**를 사용해야 함
- `OVER` 구문 지정에는 `ORDER BY stamp ASC`로, 타임스탬프를 오름 차순으로 지정
  - `ORDER BY`를 지정한 윈도 함수의 **파티션**은
    - 기본적으로 `첫 행 ~ 현재 행`
    - `ROWS BETWEEN ~`을 사용해서 각 세션 내부의 **모든 행**을 대상으로 지정

#### CODE.15.1. 각 세션의 입구 페이지와 출구 페이지 경로를 추출하는 쿼리
```sql
WITH
activity_log_with_landing_exit AS (
  SELECT
    session
    , path
    , stamp
    , FIRST_VALUE(path)
      OVER(
        PARTITION BY session
        ORDER BY stamp ASC
          ROWS BETWEEN UNBOUNDED PRECEDING
                    AND UNBOUNDED FOLLOWING
      ) AS landing
    , LAST_VALUE(path)
      OVER(
        PARTITION BY session
        ORDER BY stamp ASC
          ROWS BETWEEN UNBOUNDED PRECEDING
                    AND UNBOUNDED FOLLOWING
      ) AS exit
  FROM activity_log
)
SELECT *
FROM
  activity_log_with_landing_exit
;
```
- 각 세션의 **입구 페이지**와 **출구페이지**의 URL이 추가된 것을 확인할 수 있음
- 다음 코드처럼, **입구 페이지**와 **출구 페이지**의 URL을 기반으로
  - 유니크한 **세션**의 수를 집계
  - **방문 횟수**를 구할 수 있음

#### CODE.15.2. 각 세션의 입구 페이지와 출구 페이지를 기반으로 방문 횟수를 추출하는 쿼리
- `PostgreSQL`, `Hive`, `Redshit`, `BigQuery`, `SparkSQL`
```sql
WITH
activity_log_with_landing_exit AS (
  ...
)
, landing_count AS (
  -- 입구 페이지의 방문 횟수 집계
  SELECT
    landing AS path
    , COUNT(DISTINCT session) AS count
  FROM
    activity_log_with_landing_exit
  GROUP BY landing
)
, exit_count AS (
  -- 출구 페이지의 방문 횟수 집계
  SELECT
    exit AS path
    , COUNT(DISTINCT session) AS count
  FROM
    activity_log_with_landing_exit
  GROUP BY exit
)
-- 입구 페이지와 출구 페이지 방문 횟수 결과 한번에 출력
SELECT 'landing' AS type, * FROM landing_count
UNION ALL
SELECT 'exit' AS type, * FROM exit_count
;
```

### 어디에서 조회를 시작해서, 어디에서 이탈하는지 집계하기
- 위 코드에서 입구 페이지와 출구 페이지의 **방문 횟수**를 파악할 수 있으나,
  - 어떤 페이지에서 조회하기 시작하여,
  - 어떤 페이지에서 이탈하는지 확인 불가
- 조회 **시작 페이지**와 **이탈 페이지**의 조합을 집계하는 쿼리

#### CODE 15.3. 세션별 입구 페이지와 출구 페이지의 조합을 집계하는 쿼리
- `PostgreSQL`, `Hive`, `Redshit`, `BigQuery`, `SparkSQL`
```sql
WITH
activity_log_with_landing_exit AS (
  ...
)
SELECT
  landing
  , exit
  , COUNT(DISTINCT session) AS count
FROM
  activity_log_with_landing_exit
GROUP BY
  landing, exit
;
```

### 원포인트
- 웹 사이트의 담당자는 최상위 페이지부터 사이트 설계 진행
- 하지만 최상위 페이지부터 **조회**를 시작하는 사용자는 **거의 없음**
- **상세 페이지**부터 조회를 시작하는 사용자도 많음
- 사용자가 어디로 **유입**되는지 **입구 페이지**를 파악하면
  - 사이트를 더 매력적으로 설계 가능

## 2. 이탈률과 직귀율 계산하기
- 개념
  - SQL : `ROW_NUMBER`, `COUNT 윈도 함수`, `AVG(CASE ~ END)`
  - 분석 : 이탈률, 직귀율
- **출구 페이지**를 사용하여 **이탈률**을 계산하고,
  - 문제가 되는 페이지를 찾아내는 방법
- **직귀율**
  - 특정 페이지만 조회하고, 곧바로 이탈한 비율을 나타내는 지표
  - 직귀율이 높은 페이지
    - 성과로 바로 **이어지지 않을 가능성**이 높음
    - 확인하고 대책 필요
  - 광고로 수익을 얻는 사이트의 경우
    - 여러 페이지를 조회해야 수익이 많지만,
    - **메인 페이지**에서 **직귀율**이 높다면,
      - 사용자가 어떤 정보를 찾을 동기가 약하기 때문
    - 사이트 배치등을 검토할 수 있음

### 이탈률 집계하기
- 이탈률 공식
  - `이탈률 = 출구 수 / 페이지 뷰`
- 단순히 **이탈률**이 높다고 해서, 나쁜 페이지는 아님
  - 사용자가 만족해서 이탈하는 페이지(`구매 완료`, `기사 상세 화면`)는 당연히 이탈률이 높아야 함
    - 문제가 되지 않음
- 다만, 사용자가 만족하지 못해, 중간 과정에 이탈한다면
  - 개선 검토 필요
- 이탈률 리포트(경로, 출구 수, 페이지 뷰, 이탈률)
  >**경로**|**출구 수 **|**페이지 수**|**이탈률**
  >-----|-----|-----|-----
  >/detail| 3,667 | 9,365 |39.10%
  >/search\_list| 2,574 | 8,516 |30.20%

#### CODE.15.4. 경로별 이탈률을 집계하는 쿼리
- `PostgreSQL`, `Hive`, `RedShift`, `BigQuery`, `SparkSQL`
```sql
WITH
activity_log_with_exit_flag AS (
  SELECT
    *
    -- 출구 페이지 판정
    , CASE
        WHEN ROW_NUMBER() OVER(PARTITION BY session ORDER BY stamp DESC) = 1 THEN 1
        ELSE 0
      END AS is_exit
    FROM
      activity_log
)
SELECT
  path
  , SUM(is_exit) AS exit_count
  , COUNT(1) AS page_view
  , AVG(100.0 * is_exit) AS exit_ratio
FROM
  activity_log_with_exit_flag
GROUP BY path
;
```

### 직귀율 집계하기
- 직귀 수(**한 페이지만을 조회한 방문 횟수**)를 구한 뒤, 다음과 같은 식을 사용해 계산
  - `직귀율 = 직귀 수 / 입구 수`
  - `직귀율 = 직귀 수 / 방문 횟수`
- 순수하게 **랜딩 페이지**에서 다른 페이지로 이동하는지 평가할 때는,
  - 전자의 지표가 더 바람직한 계산식
- 특정 페이지를 조회하고, 곧바로 이탈하는 원인은
  - 연관 기사 또는 상품으로 사용자를 **이동**시키는 모듈이 **기능하지 않는 점**
  - 페이지 자체의 **콘텐츠**에 사용자가 만족하지 못하는 경우
  - 이동이 **복잡**해서 다음 단계로 이동하지 못하는 경우
- 직귀율 리포트(경로, 직귀 수, 입구 수, 직귀율)
  >**경로**|**직귀 수**|**입구 수**|**직귀율**
  >-----|-----|-----|-----
  >/detail| 1,652 | 3,539 |46.60%
  >/search\_list| 585 | 2,330 |25.10%

#### CODE.15.5. 경로들의 직귀율을 집계하는 쿼리
- `PostgreSQL`, `Hive`, `RedShift`, `BigQuery`, `SparkSQL`
```sql
WITH
activity_log_with_landing_bounce_flag AS (
  SELECT
    *
    -- 입구 페이지 판정
    , CASE
        WHEN ROW_NUMBER() OVER(PARTITION BY session ORDER BY stamp ASC) = 1 THEN 1
        ELSE 0
      END AS is_landing
    -- 직귀 판정
    , CASE
        WHEN COUNT(1) OVER(PARTITION BY session) = 1 THEN 1
        ELSE 0
      END AS is_bounce
  FROM
    activity_log
)
SELECT
  path
  , SUM(is_bounce) AS bounce_count
  , SUM(is_landing) AS landing_count
  , AVG(100.0 * CASE WHEN is_landing = 1 THEN is_bounce END) AS bounce_ratio
FROM
  actibity_log_with_landing_bounce_flag
GROUP BY path
;
```

### 원포인트
- **컴퓨터 전용 사이트**와 **스마트폰 전용 사이트**가 개별적으로 존재하는 경우
  - 사용자가 원하는 **콘텐츠**나 **콘텐츠 배치**에 차이가 있을 수 있음
- 이런 경우, 컴퓨터 전용 사이트와 스마트폰 전용 사이트를
  - **따로따로 집계**할 것
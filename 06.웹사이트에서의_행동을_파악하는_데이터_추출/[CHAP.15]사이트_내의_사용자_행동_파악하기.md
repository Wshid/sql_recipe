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
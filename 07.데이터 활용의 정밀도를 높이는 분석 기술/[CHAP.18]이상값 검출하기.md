# 18. 이상값 검출하기
- 데이터 분석을 진행할 때는, **데이터 정합성 보장**을 전제로 함
- 하지만 실제 데이터에는 **누락 부분**이 있거나, **노이즈**가 섞인 경우가 많음
  - 웹사이트 접근 로그, 센서 데이터 등에는
  - 분석 담당자가 의도하지 않은 데이터가 많이 섞여 있음
- 샘플 데이터
  - 액션 로그의 샘플 데이터
  - stamp, session, action, products, url, ip, user_agent
  
## 1. 데이터 분산 계산하기
- 개념
  - SQL : `PERCENT_RANK` 함수
  - 분석 : 상위 n%, 하위 n%
- 로그 데이터에서 **이상값**을 검출하는 가장 기본적인 방법은
  - 데이터의 **분산**을 계싼하고, 그 분산에서 많이 벗어난 값을 찾는 것
- 예시
  - 웹 사이트 접근 로그에서
  - 어떤 세션의 **페이지 조회수**가 극단적으로 많다면,
    - 다른 업체 또는 크롤러일 가능성이 존재
  - 극단적으로 접근이 적다면, **존재하지 않는 URL**에서 잘못 접근했을 가능성이 있음

### 세션별로 페이지 열람 수 랭킹 비율 구하기
- 어떤 세션의 조회수가 **극단적으로 많은지** 확인하려면
  - **세션별로 조회 수**를 계산한 뒤, **조회수**가 많은 상위 `n%`의 데이터 확인
- 세션별로 페이지 조회수를 집계하고
  - `PERCENT_RANK` 함수를 사용해 **페이지 조회 수 랭킹**을 **비율**로 구하는 쿼리
  - `PERCENT_RANK`의 값은 `(rank - 1) / (전체 페이지 수 - 1)`로 계산된 비율

#### CODE.18.1. 세션별로 페이지 열람 수 랭킹 비율을 구하는 쿼리
- `PostgreSQL`, `Hive`, `REdhisft`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  session_count AS (
    SELECT
      session
      , COUNT(1) AS count
    FROM
      action_log_with_noise
    GROUP BY
      session
  )
  SELECT
    session
    , count
    , RANK() OVER(ORDER BY count DESC) AS rank
    , PERCENT_RANK() OVER(ORDER BY count DESC) AS percent_rank
  FROM
    session_count
  ;
  ```
- `percent_rank`가 `0.05` 이하인 것을 필터링 하면,
  - 세션별로 페이지 열람 수가 많은 **상위 5%**의 세션 필터링이 가능

### URL 접근 수 worst 랭킹 비율 구하기
- 접근 수가 적은 URL을 상위로 출력
- 윈도 함수의 `ORDER BY` 구문을 `ASC`로 지정

#### CODE.18.2. URL 접근 수 워스트 랭킹 비율을 출력하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  url_count AS (
    SELECT
      url
      , COUNT(*) AS count
    FROM
      action_log_with_noise
    GROUP BY
      url
  )
  SELECT
    url
    , count
    , RANK() OVER(ORDER BY count ASC) AS rank
    , PERCENT_RANK() OVER(ORDER BY count ASC)
  FROM
    url_count
  ;
  ```
- 위 결과를 사용하면
  - 접근 수가 **일정 기준 이하**거나
  - `percent_rank`가 지정값보다 큰 것들을 필터링해서 **접근 수가 적은 레코드**를 제외할 수 있음
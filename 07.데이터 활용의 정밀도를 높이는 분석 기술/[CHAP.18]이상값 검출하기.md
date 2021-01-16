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

## 2. 크롤러 제외하기
- 개념
  - SQL : `LIKE` 연산자
  - 분석 : 크롤러 제외하기
- 접근 로그에는 사용자 행동 이외에도 `검색 엔진`이나 `도구(크롤러)`가 접근한 로그가 섞임
- 웹 사이트에 따라 섞이는 **비율**은 다르나
  - 사용자의 접근 수만큼, 또는 그 이상 `크롤러로부터 접근`이 발생하는 경우도 존재
- 분석할 때 크롤러의 로그는 **노이즈**가 되므로, 제거한 뒤 분석해야 함
- 크롤러인지, 사용자인지 판별하려면 `user-agent`를 보고 구분
- 크롤러 접근 로그를 **처음부터 출력하지 않게** 설정할 수는 있으나,
  - 새로운 크롤러가 **계속 개발**되므로, 크롤러의 접근 로그는 결국 섞임
  - **로그**로 저장된 이후에 **제외**하는 것이 좋음
- **크롤러 접근**도 **크롤러 유도**를 분석할 때 사용할 수 있는 정보
  - 일단 저장해두면 언제인가 활용 가능함

### 크롤러 접근을 제외하는 방법
- 두가지 방법
  - **규칙**을 기반으로 제외하기
  - **마스터 데이터**를 기반으로 제외하기

#### 규칙을 기반으로 제외하기
- 크롤러의 `user-agent`에 있는 특징들을 사용하면 쉽게 **제외** 가능
- 일반적으로 크롤러에는 다음과 같은 **문자열**이 포함되어 있거나, 이름이 붙는다
- 크롤러 규칙
  - 특정 문자열 포함 : `bot`, `crawler`, `spider`, ...
  - 이름 포함 : `Googlebot`, `Baiduspider`, `Yeti`, `Yahoo`, `Timblr`, ...

##### CODE.18.3. 규칙을 기반으로 크롤러를 제외하는 쿼리
- 새로운 크롤러가 발생했을 때에도 **규칙**만 조금 수정하면, 쉽게 제외가 가능
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    *
  FROM
    action_log_with_noise
  WHERE
    NOT
    -- 크롤러 판정 조건
    ( user_agent LIKE '%bot%'
    OR user_agent LIKE '%crawler%'
    OR user_agent LIKE '%spider%'
    OR user_agent LIKE '%archiver%'
    -- 생략
    )
  ;
  ```

#### 마스터 데이터를 사용해 제외하기
- **규칙**을 사용할 경우,
  - `SQL`에 직접 규칙을 작성하므로, 규칙이 많아질 수록 `SQL`이 길어지게 됨
  - 여러 곳에서 이런 규칙을 사용할 경우 `SQL`을 관리하기 번거로울 수 있음
- 별도의 **크롤러 마스터 데이터**를 만들경우, 이러한 번거로움을 줄일 수 있음

##### CODE.18.4. 마스터 데이터를 사용해 제외하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  mst_bot_user_agent AS (
    SELECT '%bot%' AS rule
    UNION ALL SELECT '%crawler%' AS rule
    UNION ALL SELECT '%spider%' AS rule
    UNION ALL SELECT '%archiver%' AS rule
  )
  , filtered_action_log AS (
    SELECT
      l.stamp, l.session, l.action, l.products, l.url, l.ip, l.user_agent
      -- UserAgent의 규칙에 해당하지 않는 로그만 남기기
      -- PostgreSQL, Redshift, BigQuery의 경우 WHERE 구문에 상관 서브쿼리 사용 가능
    FROM
      action_log_with_noise AS l
    WHERE
      NOT EXISTS (
        SELECT 1
        FROM mst_bot_user_agent AS m
        WHERE
          l.user_agent LIKE m.rule
      )
    -- 상관 서브 쿼리를 사용할 수 없는 경우
    -- CROSS JOIN으로 마스터 테이블을 결합하고
    -- HAVING 구문으로 일치하는 규칙이 0(없는) 레코드만 남기기
    -- PostgreSQL, Hive, Redshift, BigQuery, SparkSQL의 경우
    FROM
      action_log_with_noise AS l
      CROSS JOIN
        mst_bot_user_agent AS m
    GROUP BY
      l.stamp, l.session, l.action, l.products, l.url, l.ip, l.user_agent
    HAVING SUM(CASE WHEN l.user_agent LIKE m.rule THEN 1 ELSE 0 END) = 0
  )
  SELECT
    *
  FROM
    filtered_action_log
  ;
  ```

### 크롤러 감시하기
- 크롤러가 **새로 발생하는지 감지**하려면, 아래 쿼리를 주기적으로 실행
- **마스터 데이터**를 사용하여 제외한 로그에서
  - 접근이 많은 `user-agent`를 순위대로 추출한 뒤
  - 크롤러의 마스터 데이터에서 누락된 `user-agent`가 없는지 확인

#### CODE.18.5. 접근이 많은 사용자 에이전트를 확인하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  mst_bot_user_agent AS (
    -- CODE.18.4.
  )
  , filtered_action_log AS (
    -- CODE.18.4.
  )
  SELECT
    user_agent
    , COUNT(1) AS count
    , 100.0
      * SUM(COUNT(1)) OVER(ORDER BY COUNT(1) DESC
          ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
        / SUM(COUNT(1)) OVER() AS cumulative_ratio
  FROM
    filtered_action_log
  GROUP BY
    user_agent
  ORDER BY
    count DESC
  ;
  ```
- `cumulative_ratio`는 **구성비누계**를 의미
  - **10.2** 참고

### 정리
- 접근이 없었던 **크롤러**가 어느 날을 기점으로 많아지거나,
  - 새로운 크롤러가 추가되는 등의, 상황은 계속 변함
- 매주 또는 매달 **크롤러 접근 로그**를 확인할 것
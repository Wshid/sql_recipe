## 12. 시계열에 따른 사용자 전체의 상태 변화 찾기
- 사용자는 서비스 사용 시작일부터
  - 시간이 지나면, 상태 변화가 일어남
    - 충성도 높은 사용자로 성장
    - 사용 중지
    - 가입은 되어 있으나, 사용하지 않는 상태(휴면)
- **사용자가 계속해서 사용(repeat)**
- **사용자가 사용을 중단(탈퇴/휴면)**
- 서비스 운영자 입장에서는 사용자가 지속 서비스를 사용하기 원함
  - 그러려면 어느사용자가 어느정도 계속해서 사용하고 있는지 파악
  - 목표와의 괴리를 어떻게 해결할지 검토
  - 추가로, 잠시 서비스를 사용하지 않게 되어버린 **휴면**사용자를
    - 어떻게 하면 다시 사용하게 만들지도 생각해야 함
- 완전 탈퇴의 경우 어떠한 대책도 어려울 수 있으나,
  - 휴면 사용자의 경우, 메일/매거진, CM/광고 등을 활용해 다시 사용하게 할 수 있음
- 이번 절에서는
  - 사용자의 서비스 사용을 **시계열**로 수치화 하고
  - **변화를 시각화**하는 방법 소개
- 이를 활용하면
  - 현재 상태 파악이 가능하며
  - 대책의 효과를 파악하거나, 이후의 계획을 세울 때 도움이 될 것
- 샘플 데이터
  - 두개의 테이블 사용
    - 사용자 마스터
      ```sql
      CREATE TABLE mst_users(
        user_id STRING,
        sex CHAR(1),
        birth_date DATE,
        register_Date DATE,
        register_device STRING,
        withdraw_date DATE
      )
      ```
    - 액션 로그
      ```sql
      CREATE TABLE action_log(
        session STRING,
        user_id STRING,
        action  STRING,
        stamp DATETIME
      )
      ```

### 등록 수의 추이와 경향 보기
- 중요한 지표 : 등록 수
  - 사용자 등록이 필요한 서비스에서 활용
- **등록 수**가 감소 경향일 경우  
  - 서비스를 활성화 하기 어려워진다는 의미
- 날짜/등록수를 계산
- 날짜별 등록 수의 추이를 집계하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    register_Date
    , COUNT(DISTINCT user_id) AS register_count
  FROM
    mst_users
  GROUP BY
    register_date
  ORDER BY
    register_date
  ;
  ```

#### 월별 등록 수 추이
- 월별 등록수 / 전월비 를 계산
- `year_month` 컬럼을 사용하여, 월별 집계 진행
- `전월 등록 수`와 비율을 집계하기 위하여
  - `LAG 윈도 함수` 사용
- 매달 등록 수와 전월비를 계산하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  mst_users_with_year_month AS (
    SELECT
      *
      -- PostgreSQL, Hive, Redshift, SparkSQL, substring으로 연~월 부분 추출
      , substring(register_Date, 1, 7) AS year_month
      -- PostgreSQL, Hive, BigQuery, SparkSQL, substr 사용
      , substr(register_date, 1, ,7) AS year_month
    FROM
      mst_users
  )
  SELECT
    year_month
    , COUNT(DISTINCT user_id) AS register_count

    , LAG(COUNT(DISTINCT user_id)) OVER(ORDER BY year_month)
    -- SparkSQL의 경우 다음과 같이 사용
    , LAG(COUNT(DISTINCT user_id))
      OVER(ORDER BY year_month ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING)
    
    AS last_month_count
    
    , 1.0
      * COUNT(DISTINCT user_id)
      / LAG(COUNT(DISTINCT user_id)) OVER(ORDER BY year_month)
    -- sparkSQL의 경우 다음과 같이 사용
      / LAG(COUNT(DISTINCT user_id))
        OVER(ORDER BY year_month ROWS BETWEEN PRECEDING AND 1 PRECEDING)
    AS month_over_month_ratio
  FROM
    mst_users_with_year_month
  GROUP BY
    year_month
  ;
  ```

#### 등록 디바이스별 추이
- 월별 등록자 집계 이후
  - 레코드 내에 저장된 정보를 사용하여, 여러가지 내역을 집계
- 등록한 디바이스가 어떤 것인지를 나타내는
  - `mst_users`의 `device` 컬럼을 이용하여 집계
- 디바이스 외에도, pc 등록, sp 등록과 같은 다양한 컬럼으로 대체하여 집계 가능
- 디바이스들의 등록 수를 집계하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  mst_users_with_year_month AS (
    ...
  )
  SELECT
    year_month
    , COUNT(DISTINCT user_id) AS register_count
    , COUNT(DISTINCT CASE WHEN register_device='pc' THEN user_id END) AS register_pc
    , COUNT(DISTINCT CASE WHEN register_device='sp' THEN user_id END) AS regsiter_sp
    , COUNT(DISTINCT CASE WHEN register_device='app' THEN user_id END) AS register_app
  FROM
    mst_users_with_year_month
  GROUP BY
    year_month
  ;
  ```

#### 원포인트
- 등록한 디바이스에 따라 사용자 행동이 다를 수 있음
- 추가로 여러 장치를 사용하는 사용자가 있을 수 있다.
  - 이런 사용자의 존재를 파악해두면 도움이 됨

### 지속률과 정착률 산출하기
- 서비스를 지속하게 사용하는 사용자가 중요
- 등록 시점을 기준으로 
  - 일정 기간동안 사용자가 **지속해서 사용**하고 있는지를 조사할 때
  - **지속률**과 **정착률**을 사용하면, 경향 파악이 쉬움

#### 지속률과 정착률의 정의
- 지속률
  - 등록일 기준으로 이후 **지정일 동안** 사용자가 서비스를 **얼마나 이용**했는지를 나타내는 지표
  - 매일 사용하지 않아도, 판정 날짜에 사용 했다면 지속자로 취급
- 정착률
  - 등록일 기준으로 지정한 **7일 동안** 사용자가 서비스를 사용했는지 나타내는 지표
  - 7일 중, 한 번이라도 서비스를 사용했다면 **정착자**로 다룸
  - 사용만 했으면 되며, 그 기간동안 1일, 3일 등은 의미가 없음
  - 산출방법 `사용자수 / 등록 수`

#### 지속률과 정착률 사용 구분하기
- 사용자에게 기대하는 사용 사이클은 **서비스별로 다름**
- 지속률
  - 사용자가 **매일 사용**했으면 하는 서비스
  - 뉴스 사이트, 소셜 게임, SNS 등 ...
- 정착률
  - 사용자에게 **어떤 목적**이 생겼을 때 사용했으면 하는 서비스
  - EC 사이트, 리뷰 사이트, Q&A 사이트, 사진 투고 사이트 등

#### 지속률과 관계있는 리포트 - 날짜별 n일 지속률 추이
- 지속률을 날짜에 따라 집계한 리포트를 생성
  - 서비스를 활발하게 하려면, 당장 다음 날의 지속률을 높이는 것이 중요
- **다음날(1일) 지속률**
  - 지정한 날짜에 등록한 사용자 중에서
  - **다음날**에도 서비스를 사용한 사람의 비율
- 산출 방법
  - 지정한 날짜 다음에 사용한 사용자에 `1`,
    - 사용하지 않은 사용자에 `0` 플래그 사용
  - 이 값에 AVG를 사용하여 평균을 구함
- 주의할 점
  - 다음날 지속률을 집계하려면
  - 다음날의 로그 데이터가 **모두 쌓여 있어야 함**
  - 로그 집계 기간 중에
    - 가장 최신 날짜를 추출하고
    - 이러한 최신 일자를 넘는 기간의 지속률은 `NULL`로 출력하게 만들어서 해결
- 로그 최근 일자와 사용자별 등록일의 다음날을 계산하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  action_log_with_mst_users As (
    SELECT
      u.user_id
      , u.register_date
      -- 액션 날짜와 로그 전체의 최신 날짜를 날짜 자료형으로 변환
      , CAST(a.stamp AS date) AS action_date
      , MAX(CAST(a.stamp AS date)) OVER() AS latest_date
      -- BigQuery, 한번 타임 스탬프 자료형으로 변환 후 날짜 자료형으로 변환
      , date(timestamp(a.stamp)) AS action_date
      , MAX(date(timestamp(a.stamp))) OVER() AS latest_date

      -- 등록일의 다음 날짜 계산
      -- PostgreSQL
      , CAST(u.register_date::date + '1 day'::interval AS date)
      -- Redishift
      , dateadd(day, 1, u.register_date::date)
      -- BigQuery
      , date_add(CAST(u.register_date AS date), interval 1 day)
      -- Hive, SparkSQL
      , date_add(CAST(u.register_date AS date), 1)
      AS next_day_1
    FROM
      mst_users AS u
      LEFT OUTER JOIN action_log AS a ON u.user_id = a.user_id
  )
  SELECT *
  FROM
    action_log_with_mst_users
  ORDER BY
    register_date
  ```
- 지정한 날의 다음날에 액션을 했는지 `0`과 `1`의 플래그로 표현
- 사용자의 액션 플래그를 계산하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  action_log_with_mst_users AS (
    ...
  )
  , user_action_flag AS (
    SELECT
      user_id
      , register_date
      -- 등록일 다음날에 액션 여부를 플래그로 표현
      , SIGN(
        -- 사용자별 등록일 다음날에 한 액션의 합계 구하기
        SUM(
          -- 등록일 다음날이 로그 최신 날짜 이전인지 확인
          CASE WHEN next_day_1 <= latest_date THEN
            -- 등록일 다음날의 날짜에 액션을 했다면 1, 아니면 0 지정
            CASE WHEN next_day_1 = action_date THEN 1 ELSE 0 END
          END
        )
      ) AS next_1_day_action
    FROM
      action_log_with_mst_users
    GROUP BY
      user_id, register_date
  )
  SELECT *
  FROM
    user_action_flag
  ORDER BY
    register_date, user_id;
  ```
- 위의 쿼리에서, **지정한 날의 다음날 > 로그의 가장 최근 날짜**일 경우
  - 플래그가 `NULL`로 표현됨
- 다음날 지속률을 계산하는 쿼리
  ```sql
  WITH
  action_log_with_mst_users AS (
    ...
  )
  , user_action_flag As (
    ...
  )
  SELECT
    register_Date
    , AVG(100.0 * next_1_day_action) AS repeat_rate_1_day
  FROM
    user_action_flag
  GROUP BY
    register_date
  ORDER BY
    register_date
  ;
  ```
- 2일째 이후의 지속률을 계산할 경우
  - `action_log_with_mst_users` 테이블을 기반으로, 등록일 기준 `n`번째후의 날짜를 구하면 됨
- 각 지속률 값을 테이블의 컬럼으로 구성할 시, 복잡도 증가
  - 지표를 관리하는 **일시 테이블**을 사용하여, 지표를 세로 기반으로 표현
- `index_name`, `interval_date`를 가진 정보로 변환
  - `index_name` : 지표의 이름
  - `interval_date` : 등록 후 며칠째의 지표
- 지속률 지표를 관리하는 마스터 테이블을 작성하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  repeat_interval(index_name, interval_date) AS (
    -- PostgreSQL의 경우 VALUES 구문으로 일시 테이블 생성 가능
    -- Hive, Redshift, BigQuery, SparkSQL의 경우 UNION ALL로 대체 가능
    -- 8.5 참조
    VALUES
      ('01 day repeat', 1)
      , ('02 day repeat', 2)
      , ('03 day repeat', 3)
      , ('04 day repeat', 4)
      , ('05 day repeat', 5)
      , ('06 day repeat', 6)
      , ('07 day repeat', 7)
  )
  SELECT *
  FROM repeat_interval
  ORDER BY index_name
  ;
  ```
- 지표 마스터를 사용하여, 이전과 같은 과정으로 지속률 계싼
- 지속률을 세로 기반으로 집계하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  repeat_interval AS (
    ...
  )
  , action_log_with_index_date AS (
    SELECT
      u.user_id
      , u.register_date
      , CAST(a.stamp AS date) AS action_date
      , MAX(CAST(a.stamp AS date)) OVER() AS latest_date
      -- BigQuery
      , date(timestamp(a.stamp)) AS action_date
      , MAX(date(timestamp(a.stamp))) OVER() latest_date

      -- 등록일로부터 n일 후의 날짜
      , r.index_name

      -- PostgreSQL
      . CAST(CAST(u.register_date As date) + interval '1 day' * r.interval_date AS date)
      -- Redshift
      , dateadd(day, r.interval_date, u.register_date::date)
      -- BigQuery
      , date_add(CAST(u.register_date As date), interval r.interval_date day)
      -- Hive, SparkSQL
      , date_add(CAST(u.register_Date AS date), r.interval_date)
      AS index_date
    FROM
      mst_users AS u
      LEFT OUTER JOIN action_log AS a ON u.user_id = a.user_id
      CROSS JOIN
        repeat_interval AS r
  )
  , user_action_flag AS (
    SELECT
      user_id
      , register_date
      , index_name
      -- 등록일로부터 n일후 액션 여부 플래그 표현
      , SIGN(
        SUM(
          CASE WHEN index_date <= latest_date THEN
            CASE WHEN index_date = action_date THEN 1 ELSE 0 END
          END
        )
      ) AS index_date_action
    FROM
      action_log_with_index_date
    GROUP BY
      user_id, register_date, index_name, index_date
  )
  SELECT
    register_date
    , index_name
    , AVG(100.0 * index_date_action) AS repeat_rate
  FROM
    user_action_flag
  GROUP BY
    register_date, index_name
  ORDER BY
    register_Date, index_name
  ```
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

### 1. 등록 수의 추이와 경향 보기
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

### 2. 지속률과 정착률 산출하기
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
    register_date, index_name
  ```
- n일 지속률를 계산하기 위해 필요한 **판정 기간의 로그**가 존재하지 않는 경우
  - 지표는 `NULL`로 출력 됨

#### 정착률 관련 리포트
- 정착률에 대한 리포트, 리포트 작성하기 위한 SQL

#### 매일의 n일 정착률 추이
- 대책이 의도한 대로의 효과가 있는지 확인하려면
  - **정착률**을 매일 집계한 리포트가 필요
- **7일 정착률**이 극단적으로 낮은 경우에는
  - 정착률이 아니라,
  - **다음날 지속률 ~ 7일 지속률**을 확인해서 문제를 검토하는 것이 일반적
- **정착률**을 산출할 경우,
  - 대상이 되는 기간이 **여러 일자**에 걸쳐 있으므로,
  - `repeat_interval` 테이블의 `interval_date`를
    - `interval_begin_date`
    - `interval_end_date`로 확장해야 함
- 정착률 지표를 관리하는 마스터 테이블을 작성하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  repeat_interval(index_name, interval_begin_date, interval_end_date) AS (
    -- PostgreSQL, VALUES 구문으로 일시 테이블 생성
    -- Hive,Redshift, BigQuery, SparkSQL, UNION ALL 등으로 대체 가능
    -- 8-5
    VALUES
      ('07 day retention', 1, 7)
      , ('14 day retention', 8, 14)
      , ('21 day retention', 15, 21)
      , ('28 day retention', 22, 28)
  )
  SELECT *
  FROM repeat_interval
  ORDER BY index_name
  ;
  ```
- 정착률을 계산하는 쿼리
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
      -- 액션의 날짜와 로그 전체의 최신 날짜를 날짜 자료형으로 변환
      , CAST(a.stamp AS date) AS action_date
      , MAX(CAST(a.stamp AS date)) OVER() AS latest_date
      -- BigQuery, 한 번 타임 스탬프 자료형으로 변환하고, 날짜 자료형으로 변환
      , date(timestamp(a.stamp)) AS action_date
      , MAX(date(timestamp(a.stamp))) OVER() AS latest_date
      , r.index_name

      -- 지표의 대상 기간 시작일 / 종료일 계산
      -- PostgreSQL
      , CAST(u.register_date::date + '1 day'::interval * r.interval_begin_date AS date) AS index_begin_date
      , CAST(u.register_date::date + '1 day'::interval * r.interval_end_date AS date) AS index_end_date
      -- Redshift
      , dateadd(day, r.interval_begin_date, u.register_date::date) AS index_begin_date
      , dateadd(day, r.interval_end_date, u.register_date::date) AS index_end_date
      -- BigQuery
      , date_add(CAST(u.register_date AS date), interval r.interval_begin_date day) AS index_begin_date
      , date_add(CAST(u.register_date AS date, interval r.index_end_date day)) AS index_end_date
      -- Hive, SparkSQL
      , date_add(CAST(u.register_date AS date), r.interval_begin_date) AS index_begin_date
      , date_add(CAST(u.register_date AS date), r.interval_end_date) AS index_end_date)
    FROM
      mst_users AS u
      LEFT OUTER JOIN action_log AS a 
      ON u.user_id = a.user_id
      CROSS JOIN
        repeat_interval AS r 
  )
  , user_action_flag As (
    SELECT
      user_id
      , register_date
      , index_name
      -- 지표의 대상 기간에 액션을 했는지 플래그 판단
      , SIGN(
        -- 사용자별 대상 기간에 한 액션의 합 구하기
          SUM(
            -- 대상 기간의 종료일이, 로그 최신날짜 이전인지 판단
            CASE WHEN index_end_Date <= latest_date THEN
              -- 지표의 대상 기간안에 액션을 했다면 1 아니면 0
              CASE WHEN action_date BETWEEN index_begin_date AND index_end_date
                THEN 1 ELSE 0
              END
            END
          )
      ) AS index_date_action
    FROM
      action_log_with_index_date
    GROUP BY
      user_id, register_date, index_name, index_begin_date, index_end_date
  )
  SELECT
    register_date
    , index_name
    , AVG(100.0 * index_date_action) AS index_rate
  FROM
    user_action_flag
  GROUP BY
    register_date, index_name
  ORDER BY
    register_date, index_name
  ```

### n일 지속률과 n일 정착률의 추이
- **n일 지속률**과 **n일 정착률**을 따로 집계하면
  - 등록 후 며칠간, 사용자가 안정적으로 서비스를 사용하는지,
  - 며칠 후에 서비스를 그만두는 사용자가 많아지는지
    - 를 파악 가능함
- **지속률**과 **정착률**이 극단적으로 떨어지는 시점이 있다면
  - 해당 시점을 기준으로 공지사항 등을 전달하거나
  - `n일 이상` 사용한 사용자에게 보너스를 주는 등의 대첵을 이행하여
    - 지속률과 정착률이 다시 안정적으로 돌아오는 날까지, 사용자를 잡을 수 있음
- **정착률**을 계산하기 위해 만들었던,
  - `repeat_interval` 테이블의 형식을 수정하면
  - **지속률**도 계산 가능
- 지속률 지표를 관리하는 마스터 테이블을 정착률 형식으로 수정한 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  repeat_interval(index_name, interval_begin_date, interval_end_date) AS (
    -- PostgreSQL, VALUES로 일시 테이블 생성 가능
    -- Hive, Redshift, BigQuery, SparkSQL의 경우 UNION ALL 대체 가능
    VALUES
      ('01 day repeat'      , 1, 1)
      , ('02 day repeat'    , 2, 2)
      , ('03 day repeat'    , 3, 3)
      , ('04 day repeat'    , 4, 4)
      , ('05 day repeat'    , 5, 5)
      , ('06 day repeat'    , 6, 6)
      , ('07 day repeat'    , 7, 7)
      , ('07 day retention' , 1, 7)
      , ('14 day retention' , 8, 14)
      , ('21 day retention' , 15, 21)
      , ('28 day retention' , 22, 28)
  )
  SELECT *
  FROM repeat_interval
  ORDER BY index_name
  ;
  ```
- n일 지속률들을 집계하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  repeat_interval AS (
    ...
  )
  , action_log_with_index_date AS (
    ...
  )
  , user_action_flag AS (
    ...
  )
  SELECT
    index_name
    , AVG(100.0 * index_date_action) AS repeat_rate
  FROM
    user_action_flag
  GROUP BY
    index_name
  ORDER BY
    index_name
  ```

#### 원포인트
- **지속률**과 **정착률**은 모두
  - **등록일**기준으로 `n`일 후의 행동을 집계
- 따라서 **등록일**로부터 `n`일 경과하지 않는 상태라면, 집계꺄ㅏ 불가능
- 그러므로 `30일 지속률`과 `60일 지속률`처럼
  - 값을 구하기 위해 시간이 오래 걸리는 지표보다는
  - `1일 지속률`, `7일 지속률`, `7일 정착률`처럼
    - **단기간**에 결과를 보고 대책을 세울 수 있는 지표를 활용하는 것이 좋음
- **정착률**은 `7일`동안의 기간을 집계하므로
  - 실제로 며칠 사용했는지는 알 수 없음
- 예시로
  - `14일`의 정착률 기간 동안 70% 사용자는 1~2일 정도에 그친다 등을 알고 싶다면
  - `5-4`에서 확인했던 SQL을 다시 볼 것

### 3. 지속과 정착에 영향을 주는 액션 집계하기
- 지속률과 정착률의 추이를 계산하여 사용자의 상황을 이해하는 것도 중요하지만
  - **무엇 때문에** 그러한 추이가 발생하는지 모르면 대책 세우기가 어려움
- 지표와 리포트를 만들때 **OO율을 올리자** 처럼 새로운 목표와 과제가 있어야 하며
  - 그 목표를 위해 무엇을 해야할지 구체적인 **대책** 제시가 필요
- **OO하는 사용자는 무엇에 영향을 주는지**에 대해 구하는 방법

#### 1일 지속률을 개선하기
- **1일 지속률**을 개선하려면
  - 등록한 당일 사용자들이 무엇을 했는지 확인하면 됨
- **14일 정착률**을 개선하려면
  - 7일 정착률의 판정 기간 동안, 사용자가 어떤 행동을 했는지를 조사
- 1일 지속률을 개선하기 위해
  - **등록일의 액션 사용 여부**와 **1일 지속률**에 어떤 차이가 있는지를 집계
  - `profile_regist`, `tutorial-step1`, ...
  - 이 액션 마다의
    - 사용자 수, 사용률, 사용자 1일 지속률, 비사용자 1일 지속률
  - 1일 지속률에 더 영향을 주는 주요 포인트
    - **사용자**의 1일 지속률이 높고
    - **비사용자**의 1일 지속률이 낮을 경우
  - 사용자 영향을 많이 주는 **액션의 사용률이 낮다면**
    - 사용자들이 해당 액션을 할 수 있게 **설명**을 추가하거나
    - 이벤트를 통해 **액션 사용**을 촉진하고,
      - 사이트의 디자인/동선도 같이 검토해야 함
- 각 액션에 대한 **사용자**와 **비사용자**의 다음날 지속률을 한번에 계싼하려면
  - 모든 사용자의 **모든 액션 조합**을 생성한 후
    - 사용자의 액션 실행 여부를 `0/1`로 나타내는 테이블을 만들어 집계
- 모든 사용자와 액션의 조합 만들기
  - **사용자 마스터 테이블**과 **액션 마스터 테이블**을 `CROSS JOIN`하기
- 모든 사용자와 액션의 조합을 도출하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  repeat_interval(index_name, interval_begin_Date, interval_end_date) AS (
    -- PostgreSQL의 경우 VALUES로 일시 테이블 생성 가능
    -- Hive, Redshift, Bigquery, SparkSQL의 경우 SELECT로 대체 가능
    -- 4.5 참고
    VALUES ('01 day repeat', 1, 1)
  )
  , action_log_with_index_Date AS (
    ...
  )
  , user_action_flag AS (
    ...
  )
  , mst_actions AS (
    SELECT 'view' AS action
    UNION ALL SELECT 'comment' AS action
    UNION ALL SELECT 'follow' AS action
  )
  , mst_user_actions AS (
    SELECT
      u.user_id
      , u.register_date
      , a.action
    FROM
      mst_users AS u
      CROSS JOIN
        mst_actions AS a
  )
  SELECT *
  FROM mst_user_actions
  ORDER BY user_id, action
  ;
  ```
- 사용자의 액션 로그를 0/1 플래그로 표현하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  repeat_interval AS (
    ...
  )
  , action_log_with_index_date AS (
    ...
  )
  , user_action_flag AS (
    ...
  )
  , mst_actions AS (
    ...
  )
  , mst_user_actions AS (
    ...
  )
  , register_action_flag AS (
    SELECT DISTINCT
      m.user_id
      , m.register_date
      , m.action
      , CASE
          WHEN a.action IS NOT NULL THEN 1
          ELSE 0
        END AS do_action
      , index_name
      , index_date_action
    FROM
      mst_user_actions AS m
      LEFT JOIN
        action_log AS a
        ON m.user_id = a.user_id
        AND CAST(m.register_date AS date) = CAST(a.stamp AS date)
        -- BigQuery, Timestamp 자료형으로 변환한 뒤, 날짜 자료형으로 변환
        AND CAST(m.regsiter_date AS date) = date(timestamp(a.stamp))
        AND m.action = a.action
      LEFT JOIN
        user_action_flag AS f
        ON m.user_id = f.user_id
    WHERE
      f.index_date_action IS NOT NULL
  )
  SELECT *
  FROM register_action_flag
  ORDER BY user_id, index_name, action
  ;
  ```
- 실행 결과 형태
  >**user\_id**|**register\_date**|**action**|**do\_action**|**index\_name**|**index\_date\_action**
  >:-----:|:-----:|:-----:|:-----:|:-----:|:-----:
  >U001|2016.10.1|comment|0|01 day repeat|1
  >U001|2016.10.1|follow|1|01 day repeat|1
  >U002|2016.10.1|comment|0|01 day repeat|1
- `U001`의 경우, 2016-10-01에 `follow` 액션을 수행,
  - 다음날 지속률 판정 기간(1일) 동안, `comment`, `follow`를 수행했다는 의미
- 사용자의 액션로그가 `0/1`로 표현되었다면
  - 전제 조건에 따라 비율을 `AVG` 함수로 계산
- 등록일에 해당 액션을 사용한 사용자는 `do_action = 1`이며,
  - 등록일에 해당 액션을 사용하지 않은 사용자는 `do_action = 0`으로 구분
- 이를 활용하여 **사용자**와 **비사용자**기반으로
  - `index_date_action`의 평균을 계산하면, **1일 지속률**을 구할 수 있음
- 액션에 따른 지속률과 정착률을 집계하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  repeat_interval AS (
    ...
  )
  , action_log_with_index_date AS (
    ...
  )
  , user_action_flag AS (
    ...
  )
  , mst_actions(action) AS (
    ...
  )
  , mst_user_actions AS (
    ...
  )
  , register_action_flag AS (
    ...
  )
  SELECT
    action
    , COUNT(1) users
    , AVG(100.0 * do_action) AS usage_rate
    , index_name
    , AVG(CASE do_action WHEN 1 THEN 100.0 * index_date_action END) AS idx_rate
    , AVG(CASE do_action WHEN 0 THEN 100.0 * index_date_action END) AS no_action_idx_rate
  FROM
    register_action_flag
  GROUP BY
    index_name, action
  ORDER BY
    index_name, action
  ``` 
- 실행 결과
  >**action**|**users**|**usage\_rate**|**index\_name**|**idx\_rate**|**no\_action\_idx\_rate**
  >:-----:|:-----:|:-----:|:-----:|:-----:|:-----:
  >comment|28|7.14|01 day repeat|0.00 |15.38
  >follow|28|7.14|01 day repeat|50.00 |11.53
  >view|28|100|01 day repeat|14.28 | 

#### 원포인트
- 특정 액션의 실행이 지속률과 정착률 상승으로 이어질 것으로 보여도
  - 해당 액션을 실행하는 **진입 장벽**이 높다면,
  - 지속률과 정착률에 조금 영향을 주더라도, 액션을 실행하는 진입 장벽이 **낮은** 쪽으로 대책을 세우는 것이 좋음
    - ex) 동영상 업로드 보다는 이미지 업로드 촉진 등
- 액션 여부뿐 아니라, **액션 수**에 따라서도 차이가 있을 수 있음

### 4. 액션 수에 따른 정착률 집계하기
- `CROSS JOIN`, `CASE`, `AVG`
- SNS 사례
  - 페이스북과 트위터 등 대표적인 SNS 사례 중
  - `등록 후 1주일 이내에 10명을 팔로우하면, 해당 사용자는 서비스를 계속해서 사용한다`
    - 알 수도 있는 사람
    - 00님을 함께 알고 있습니다 등의 마케팅을 하는 이유
  - 인기 사용자를 팔로우하는 튜토리얼을 만들어, **서비스 활성화**를 유도
- 등록일과, 이후 7일 동안(**7일 정착률 기간**)에
  - 실행한 액션 수에 따라 14일 정착률이 어떻게 변화하는지 확인
- 특정 기간동안의 **액션수 집계** 이후
  - 달성한 사람의 14일 정착률을 집계한다.
- 최종적으로, 각 액션별로 도수 분포표를 만들고,
  - 달성자의 14일 정착률을 집계
- **액션 단계 마스터**를 일시 테이블로 정의
  - `CROSS JOIN`을 사용하여, **사용자와 액션 조합** 구성
- 액션의 계급 마스터와 사용자 액션 플래그의 조합을 산출하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  repeat_interval(index_name, interval_begin_date, interval_end_date) AS (
    -- PostgreSQL의 경우 VALUES로 일시 테이블 생성 가능
    -- Hive, Redshift, BigQuery, SparkSQLL의 경우 SELECT로 ㄷ ㅐ체 가능
    -- 8강 5절
    VALUES ('14 day retention', 8, 14)
  )
  , action_log_with_index_date AS (
    -- 12-10 코드
  )
  , user_action_flag AS (
    -- 12-10 코드
  )
  , mst_action_bucket(action, min_count, max_count) AS (
    -- 액션 단계 마스터
    VALUES
      ('comment', 0, 0)
      , ('comment', 1, 5)
      , ('comment', 6, 10)
      , ('comment', 11, 9999) -- 최댓값으로 간단하게 9999 입력
      , ('follow', 0, 0)
      , ('follow', 1, 5)
      , ('follow', 6, 10)
      , ('follow', 11, 9999) --최댓값으로 간단하게 9999 입력
  )
  , mst_user_action_bucket AS (
    -- 사용자 마스터와 액션 단계 마스터 조합
    SELECT
      u.user_id
      , u.register_date
      , a.action
      , a.min_count
      , a.max_count
    FROM
      mst_users AS u
      CROSS JOIN
        mst_action_bucket AS a
  )
  SELECT *
  FROM
    mst_user_action_bucket
  ORDER BY
    user_id, action, min_count
  ```
- 위 코드 결과에 7일 동안의 로그를 `LEFT JOIN`하고,
  - 등록 후 7일 동안의 액션수를 집계하는 쿼리 작성
  - 7일 동안의 로그를 결합할 수 있게 `JOIN`구문 내부에 `BETWEEN`을 사용했지만
    - 이는 `Hive`에서 동작하지 않음
    - `Hive`에서는 별도의 `WHERE` 구문을 추가로 사용해야한다.
- 등록 후 7일 동안의 액션 수를 집계하는 쿼리
  - `PostgreSQL`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  repeat_interval AS (
    ...
  )
  , action_log_with_index_date AS (
    ...
  )
  , user_action_flag AS (
    ...
  )
  , mst_action_bucket AS (
    ...
  )
  , mst_user_action_bucket AS (
    ...
  )
  , register_action_flag As (
    -- 등록일에서 7일 후까지 액션수를 세고,
    -- 액션 단계와 14일 정착 달성 플래그 계산
    SELECT
      m.user_id
      , m.action
      , m.min_count
      , m.max_count
      , COUNT(a.action) AS action_count
      , CASE
          WHEN COUNT(a.action) BETWEEN m.min_count AND m.max_count THEN 1
          ELSE 0
        END As achieve
      , index_name
      , index_date_action
    FROM
      mst_user_action_bucket AS m
      LEFT JOIN
        action_log AS a
        ON m.user_id = a.user_id
        -- 등록일 당일부터 7일 후까지의 액션 로그 결합
        -- PostgreSQL, Redshift의 경우
        AND CAST(a.stamp AS date)
          BETWEEN CAST(m.register_date AS date)
          AND CAST(m.register_date AS date) + interval '7 days'
        -- BigQuery의 경우
        AND date(timestamp(a.stamp))
          BETWEEN CAST(m.register_date AS date)
          AND date_add(CAST(m.register_date AS date), interval 7day)
        -- SparkSQL
        AND CAST(a.stamp AS date)
          BETWEEN CAST(m.register_date AS date)
          AND date_add(CAST(m.register_date AS date), 7)
        -- Hive의 경우 JOIN 구문에 BETWEEN을 사용할 수 없으므로, WHERE을 사용하여야 함
        AND m.action = a.action
      LEFT JOIN
        user_action_flag AS f
        ON m.user_id = f.user_id
      WHERE
        f.index_date_action IS NOT NULL
      GROUP BY
        m.user_id
        , m.action
        , m.min_count
        , m.max_count
        , f.index_name
        , f.index_date_action
  )
  SELECT *
  FROM
    register_action_flag
  ORDER BY
    user_id, action, min_count
  ;
  ```
- 앞 코드에서 구한 액션 횟수를 기반으로 **14일 정착률 집계**
- 등록 후 7일동안의 액션 횟수별로 14일 정착률을 집계하는 쿼리
  - `PostgreSQL`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  repeat_interval AS (
    ...
  )
  , action_log_with_index_date AS (
    ...
  )
  , user_action_flag AS (
    ...
  )
  , mst_action_bucket AS (
    ...
  )
  , mst_user_action_bucket AS (
    ...
  )
  , register_action_flag AS (
    ...
  )
  SELECT
    action
    -- PostgreSQL, Redshift, 문자열 연결
    , min_count || '~' || max_count AS count_range
    -- BigQuery
    , CONCAT(CAST(min_count AS string), '~', CAST(max_count AS string)) AS count_range
    , SUM(CASE achieve WHEN 1 THEN 1 ELSE 0 END) AS archieve
    , index_name
    , AVG(CACSE achieve WHEN 1 THEN 100*0 * index_date_action END) AS achieve_index_date
  FROM
    register_action_flag
  GROUP BY
    index_name, action, min_count, max_count
  GROUP BY
    index_name, action, min_count;
  ```
- 사용자의 **액션**을 기반으로 사용자를 집계하는 방법
- 액션별로 사용자를 집계하면
  - **사용자가 어떤 기능을 더 많이 사용하도록 유도해야 하는지 파악 가능**

### 5. 사용 일수에 따른 정착률 집계하기
- `COUNT`, `SUM` 윈도 함수, `AVG`함수
- 정착률
- **7일 정착 기간**동안
  - 사용자가 며칠 사용했는지가
  - 이후 정착률에 어떠한 영향을 주는지 확인하는 방법
- 특정 날짜에 등록한 사용자가
  - `등록 다음날부터 7일 중 며칠을 사용했는지`를 집계하고
  - `28일 정착률`(등록일 `21`일부터 `28`일 후까지도 사용한 사용자 비율)을 집계
    - **등록일 다음날부터 7일 이내의 사용일에 따른 28일 정착률**
- 매일 사용하는 사용자가 당연히 오래 사용하겠으나,
  - 리포트로 만들면, 이를 수치화해서 근거를 명확하게 제시 가능
  - 이러한 리포트 활용시, 등록후 7일 동안 사용자가 계속해서 사용하게 만드는 대책을 세울 수 있음

#### 리포트를 만드는 방법
- 등록일 다음날부터 **7일 동안의 사용 일수** 집계
- 사용 일수별로 집계한 **사용자 수**와 **구성비**, **구성비누계**를 계산
- **사용 일수별로 집계한 사용자 수**를 분모로 두고,
  - **28일 정착률**을 집계한 뒤, 그 비율을 계산
- 정착률 지표 마스터를 만들고, 내부에 `user_action_flag`를 집계
  - 이어서 **사용자 마스터**에
    - **등록일 다음날**부터 **7일 이내의 액션 로그를 결합**
    - 해당 기간의 사용일 수 계산
- 등록일 다음날부터 7일 동안의 사용일수와 28일 정착 플래그를 생성하는 쿼리
  - `PostgreSQL`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  repeat_interval(index_name, interval_begin_date, interval_end_date) AS (
    -- PostgreSQL은 VALUES로 일시 테이블 생성 가능
    -- Hive, Redshift, BigQuery, SparkSQL의 경우 SELECT로 대체 가능
    VALUES ('28 day retention', 22, 28)
  )
  , action_log_with_index_date AS (
    ...
  )
  , user_action_flag AS (
    ...
  )
  , register_action_flag AS (
    SELECT
      m.user_id
      , COUNT(DISTINCT CAST(a.stamp AS date)) AS dt_count
      -- BigQuery의 경우 다음과 같이 사용
      , COUNT(DISTINCT date(timestamp(a.stamp))) AS dt_count
      , index_name
      , index_date_action
    FROM
      mst_users AS m
      LEFT JOIN
        action_log AS a
        ON m.user_id = a.user_id

        -- 등록 다음날부터 7일 이내의 액션 로그 결합하기
        -- PostgreSQL, Redshift의 경우 다음과 같이 사용
        AND CAST(a.stamp AS date)
          BETWEEN CAST(m.register_date AS date) + interval '1 day'
            AND CAST(m.register_date AS date) + interval '8 days'
        
        -- BigQuery
        AND date(timestamp(a.stamp))
          BETWEEN date_add(CAST(m.register_date AS date), interval 1 day)
            AND date_add(CAST(m.register_date AS date), interval 8 day)

        -- SparkSQL
        AND CAST(a.stamp AS date)
          BETWEEN date_add(CAST(m.register_Date AS date), 1)
            AND date_add(CAST(m.register_Date AS date), 8)
        
        -- Hive, JOIN에 BETWEEN을 사용할 수 없으므로, WHERE 사용
      LEFT JOIN
        user_action_flag AS f
        ON m.user_id = f.user_id
    WHERE
      f.index_date_action IS NOT NULL
    GROUP BY
      m.user_id
      , f.index_name
      , f.index_date_action
  )
  SELECT *
  FROM
    register_action_flag;
  ```
  - 등록일 다음날부터 7일 동안의 사용 일수와, 해당 사용자의 28일 정착 플래그 생성
- 사용 일수에 따른 정착율을 집계하는 쿼리
  - `PostgreSQL`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  repeat_interval AS (
    ...
  )
  , action_log_with_index_date AS (
    ...
  )
  , user_action_flag AS (
    ...
  )
  , register_action_flag AS (
    ...
  )
  SELECT
    dt_count AS dates
    , COUNT(user_id) AS users
    , 100.0 * COUNT(user_id) / SUM(COUNT(user_id)) OVER() AS user_ratio
    , 100.0 *
      SUM(COUNT(user_id))
        OVER(ORDER BY index_name, dt_count)
          ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
      / SUM(COUNT(user_id)) OVER() AS cum_ratio
    , SUM(index_date_action) AS achieve_users
    , AVG(100.0 * index_date_action) AS achieve_ratio
  FROM
    register_action_flag
  GROUP BY
    index_name, dt_count
  ORDER BY
    index_name, dt_count;
  ```

#### 원포인트
- 등록일 다음날부터 7일 동안의 사용일수를 사용해 집계했지만,
  - 사용 일수대신 
    - SNS 서비스에 올린 **글의 개수** 또는 **소셜 게임의 레벨**등으로 대상을 변경하여 사용할 수 있음

### 6. 사용자의 잔존율 집계하기
- `CROSS JOIN`, `SUM(CASE ~)`, `AVG(CASE ~)`
- 잔존율
- 서비스 등록 수개월 후에, 어느 정도 비율의 사용자가
  - 서비스를 지속해서 사용하는지, 대략적으로 파악해두면
  - 서비스에 어떤 문제가 있는지 찾거나,
    - 과거와 현재를 비교하고, 미래에 대한 목표 전망 검토가 가능
- 가로축 - 등록일, 세로 축 - 해당 월의 서비스 사용자수 집계

 |**2016년 1월**|**2016년 2월**|**2016년 3월**|**2016년 4월**|**2016년 5월**|**2016년 6월**
:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:
2016년 1월|100(100%)| | | | | 
2016년 2월|90(90%)|150(100%)| | | | 
2016년 3월|80(80%)|120(80%)|120(100%)| | | 
2016년 4월|70(70%)|90(60%)|90(75%)|200(100%)| | 
2016년 5월|60(60%)|60(40%)|90(75%)|150(75%)|250(100%)| 
2016년 6월|50(50%)|30(20%)|30(25%)|100(50%)|125(50%)|220(100%)
- 위 표의 의미
  - 이전과 비교해 `n`개월 후의 잔존율이 내려감
    - 신규 등록자가 서비스를 사용하기 위한 **진입 장벽**이 높아지진 않았는지 확인
  - n개월 이후에 **잔존율이 갑자기 낮아짐**
    - 서비스 사용 목적을 달성하는 기간이, 예상보다 너무 짧지는 않은지
  - 오래 사용하던 사용자인데도, **특정 월**을 기준으로 사용하지 않음
    - 사용자가 서비스 내부의 경쟁으로 빨리 지친것은 아닌지
- 사용자의 잔존율을 월단위로 집계하려면,
  - 사용자의 서비스 등록일로부터 **12개월 후**까지 월을 도출하기 위한 **추가 테이블 필요**
- 12개월 후까지의 월을 도출하기 위한 보조 테이블 생성 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  mst_intervals(interval_month) AS (
    -- 12개월 동안의 순번 만들기(generate_series 등으로 대체 가능)
    -- PostgreSQL의 경우 VALUES 구문으로 일시 테이블 생성
    -- Hive, Redshift, BigQuery, SparkSQL의 경우 SELECT 구문과 UNION ALL로 대체 가능
    VALUES (1), (2), (3), (4), (5), (6), (7), (8), (9), (10), (11), (12)
  )
  SELECT *
  FROM mst_intervals
  ;
  ```
- 위 보조 테이블을 사용하여
  - 사용자의 등록일부터 **12개월 후**까지의 월을 **사용자 마스터**에 추가
- 이후 **액션 로그**의 **로그 날짜**를 월 단위 표현으로 변경,
  - 등록 월부터 12개월 후까지의 월을 추가한 **사용자 마스터**와 결합하여 **월 단위 잔존율 집계**
- 등록 월에서 12개월 후까지의 잔존율을 집계하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  mst_intervals AS (
    ...
  )
  , mst_users_with_index_month AS (
    -- 사용자 마스터에 등록 월부터 12개월 후까지의 월 추가
    SELECT
      u.user_id
      , u.register_date
      -- n개월 후의 날짜, 등록일, 등록 월 n개월 후의 월 계산
      -- PostgreSQL의 경우 다음과 같이 사용
      , CAST(u.register_date::date + i.interval_month * '1 month'::interval AS date) AS index_date
      , substring(u.register_date, 1, 7) AS register_month
      , substring(CAST(
        u.register_date::date + i.interval_month * '1 month'::interval AS text), 1, 7) AS index_month
      
      -- Redshift
      , dateadd(month, i.interval_month, u.register_date::date) AS index_date
      , substring(u.register_date, 1, 7) AS register_month
      , substring(
        CAST(dateadd(month, i.interval_month, u.regsiter_date::date) AS text), 1, 7) AS index_month
      
      -- BigQuery
      , date_add(date(timestamp(u.register_date)), interval i.interval_month month) AS index_date
      , substr(u.register_date, 1, 7) AS register_month
      , substr(CAST(
        date_add(date(timestamp(u.register_date)), interval i.interval_month month) AS string), 1,7) AS index_month
      
      -- Hive, SparkSQL
      , add_month(u.register_date, i.interval_month) AS index_date
      , substring(u.register_date, 1, 7) AS register_month
      , substring(
        CAST(add_months(u.register_date, i.interval_month) AS string), 1, 7) AS index_month
    FROM
      mst_users AS u
      CROSS JOIN
        mst_intervals AS i
  )
  , action_log_in_month AS (
    -- 액션 로그의 날짜에서 월 부분만 추출
    SELECT DISTINCT
      user_id
      , substring(stamp, 1, 7) AS action_month
      -- BigQuery의 경우 substr 사용
      , substr(stamp, 1, 7) AS action_month
    FROM
      action_log
  )
  SELECT
    -- 사용자 마스터와 액션 로그를 결합 후, 월별로 잔존율 집계
    u.register_month
    , u.index_month
    -- action_month가 NULL이 아니라면(액션을 했다면) 사용자 수 집계
    , SUM(CASE WHEN a.action_month IS NOT NULL THEN 1 ELSE 0 END) AS users
    , AVG(CASE WHEN a.action_month IS NOT NULL THEN 100.0 ELSE 0.0 END) AS retention_rate
  FROM
    mst_users_with_index_month AS u
    LEFT JOIN
      action_log_in_month AS a
      ON u.user_id = a.user_id
      AND u.index_month = a.action_month
  GROUP BY
    u.register_month, u.index_month
  ORDER BY
    u.register_month, u.index_month
  ;
  ```

#### 원포인트
- 매일 확인해야할 리포트는 아니지만,
  - **장기적인 관점**에서 **사용자 등록**과 **지속 사용**을 파악할 때는 유용하게 활용 가능
- 이런 리포트를 작성할 때는
  - 해당 **월**에 실시한 대책 또는 캠페인 등의 **이벤트를 함께 기록**하면,
  - 수치 변화의 원인 등도 쉽게 파악이 가능

### 7. 방문 빈도를 기반으로 사용자 속성을 정의하고 집계하기
- `CASE`식, `NULLIF`, `LAG`
- `MAU`, 리피트, 컴백
- 서비스 사용자의 방문 빈도를 **월 단위**로 파악하고,
  - 방문 빈도에 따라 **사용자 분류** 후 내역을 집계하는 방법

#### MAU
- 특정 월에 서비스를 사용한 사용자 수를 **MAU**(Monthly Active User)라고 함
- 단, `이번 달의 MAU는 30만이다`라고 하더라도,
  - 30만명중에 몇명이 **신규 사용자**인지는 확인할 수 없음
- 이런 내역에 따라 **서비스의 가치**가 달라짐

#### MAU를 3개로 나누어 분석하기
- MAU만으로는, 어떤 사용자가 서비스를 사용하는지 제대로 파악 불가능
- 서비스 사용자의 구성을 더 자세하게 파악하려면
  - `MAU`를 **3개의 속성**으로 나누어 분석
- 3개의 속성
  - **신규 사용자** : 이번 달에 등록한 신규 사용자
  - **리피트 사용자** : 이전 달에도 사용한 사용자
  - **컴백 사용자** : 이번달의 신규 사용자가 아니며, 리피트 사용자도 아닌, 한동안 사용하지 않았다 돌아온 사용자
- 10만의 MAU가 있다는 정보를 기반으로 대책을 세운다면,
  - 대책이 애매할 수 밖에 없음
- 신규 사용자 2만, 리피트 사용자 7만, 컴백 1만명으로 분류 후,
  - 신규 사용자를 대상으로 어떠한 대책을 시행할지 등을 검토하는 것이 효율적
- 신규 사용자 수, 리피트 사용자 수, 컴백 사용자 수를 집계하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  monthly_user_action AS (
    -- 월별 사용자 액션 집약하기
    SELECT DISTINCT
      u.user_id
      -- PostgreSQL
      , substring(u.register_date, 1, 7) AS register_month
      , substring(l.stamp, 1, 7) AS action_month
      , substring(CAST(
        l.stamp::date - interval '1 month' AS text
      ), 1, 7) AS action_month_priv

      -- Redshift
      , substring(u.regsiter_date, 1, 7) AS register_month
      , substring(l.stamp, 1, 7) As action_month
      , substring(
        CAST(dateadd(month, -1, l.stamp::date) AS text), 1, 7
      ) AS action_month_priv

      -- BigQuery
      , substr(u.register_date, 1, 7) AS register_month
      , substr(l.stamp, 1, 7) AS action_month
      , substr(CAST(
        date_sub(date(timestamp(l.stamp)), interval 1 month) AS string
      ), 1, 7) AS action_month_priv

      -- Hive, SparkSQL
      , substring(u.register_date, 1, 7) AS register_month
      , substring(l.stamp, 1, 7) AS action_month
      , substring(
        CAST(add_month(to_date(l.stamp), -1) AS string), 1, 7
      ) AS action_month_priv
    
    FROM
      mst_users AS u
      JOIN
        action_log AS l
        ON u.user_id = l.user_id
  )
  , monthly_user_with_type AS (
    -- 월별 사용자 분류 테이블
    SELECT
      action_month
      , user_id
      , CASE
          -- 등록 월과 액션월이 일치하면 신규 사용자
          WHEN register_month = action_month THEN 'new_user'
          -- 이전 월에 액션이 있다면, 리피트 사용자
          WHEN action_month_priv = 
            LAG(action_month)

            OVER(PARTITION BY user_id ORDER BY action_month)
            -- SparkSQL의 경우 다음과 같이 사용
            OVER(PARTITION BY user_id ORDER BY action_month ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING)
            THEN 'repeat_user'
          -- 이외의 경우에는 컴백 사용자
          ELSE 'come_back_user'
        END AS c
      , action_month_priv
    FROM
      monthly_user_action
  )
  SELECT
    action_month

    -- 특정 달의 MAU
    , COUNT(user_id) AS mau
    -- new_users / repeat_users / com_back_users
    , COUNT(CASE WHEN c = 'new_user' THEN 1 END) AS new_users
    , COUNT(CASE WHEN c = 'repeat_user' THEN 1 END) AS repeat_users
    , COUNT(CASE WHEN c = 'come_back_user' THEN 1 END) AS come_back_users
  FROM
    monthly_user_with_type
  GROUP BY
    action_month
  ORDER BY
    action_month
  ;
  ```

#### 리피트 사용자를 3가지로 분류하기
- 리피트 사용자
  - 이전 달에서도 사용한 사용자
  - 3가지 분류
    - 신규 리피트 사용자 : 이전 달에는 **신규 사용자**, 이번달에도 사용
    - 기존 리피트 사용자 : 이전 달도 **리피트 사용자**, 이번달에도 사용
    - 컴백 리피트 사용자 : 이전 달에 **컴백 사용자**, 이번달에도 사용
- 효과
  - 같은 리피트 사용자라도,
    - 등록한지 얼마 안 되어 서비스 이용 경험이 적은 사용자인지
    - 지속적으로 사용하는 사용자인지
      - 등으로 상세히 파악 가능
- 리피트 사용자를 세분화해서 집계하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  monthly_user_action AS (
    ...
  )
  , monthly_user_with_type AS (
    ...
  )
  , monthly_users AS (
    SELECT
      m1.action_month
      , COUNT(m1.user_id) AS mau
      , COUNT(CASE WHEN m1.c = 'new_user'   THEN 1 END) AS new_users
      , COUNT(CASE WHEN m1.c = 'repeat_user'  THEN 1 END) AS repeat_users
      , COUNT(CASE WHEN m1.c = 'come_back_user' THEN 1 END) AS come_back_users
      
      , COUNT(
        CASE WHEN m1.c = 'repeat_user' AND m0.c = 'new_user' THEN 1 END
      ) AS new_repeat_users
      , COUNT(
        CASE WHEN m1.c = 'repeat_user' AND m0.c = 'repeat_user' THEN 1 END
      ) AS continuous_repeat_users
      , COUNT(
        CASE WHEN m1.c = 'repeat_user' AND m0.c = 'come_back_user' THEN 1 END
      ) AS come_back_repeat_uesrs
    FROM
      -- m1 : 해당 월의 사용자 분류 테이블
      monthly_user_with_type AS m1
      LEFT OUTER JOIN
      -- m0 : 이전 달의 사용자 분류 테이블
      ON m1.user_id = m0.user_id
      AND m1.action_month_priv = m0.action_month
    GROUP BY
      m1.action_month
  ) 
  SELECT
    *
  FROM
    monthly_users
  ORDER BY
    action_month;
  ```

#### MAU 속성별 반복률 계산하기
- MAU를 몇 가지 속성으로 분류하고
  - 사용자 수를 계산해서, 원형 그래프로 그리는 방법
- **구성비**만으로는
  - `이전 달의 신규 등록 사용자 중에 어느 정도가 리피트 사용자로 전환되었는가`
  - `이전 달에 시행한 컴백 사용자를 늘리기 위한 캠페인의 효과가 얼마나 되는가`
    - 등을 파악하기 어려움
- 각 출력 결과를 바탕으로 다음과 같은 표 생성
  - 이 표를 통해, 위 질문에 대한 답이 가능
  - 반복률 계산시, 사용한 값(분자/분모 값)은 따로 출력하지 않음
- 표

 |**2016년 3월**|**2016년 4월**|**2016년 5월**|**2016년 6월**|**2016년 7월**|**2016년 8월**
-----|-----|-----|-----|-----|-----|-----
MAU| 21,931 | 35,555 | 117,819 | 320,684 | 429,247 | 293,566 
신규 MAU| 21,931 | 27,755 | 106,249 | 277,121 | 314,249 | 138,463 
신규 반복 MAU| | 7,800 | 7,267 | 34,593 | 87,932 | 78,403 
신규 반복률| |35.60%|26.20%|32.60%|31.70%|24.90%
반복 MAU| | 7,800 | 10,634 | 40,586 | 109,213 | 127,257 
기존 반복 MAU| | | 3,367 | 5,590 | 20,176 | 49,385 
기존 반복률| | |43.20%|52.60%|49.70%|43.40%
컴백 MAU| | | 936 | 2,977 | 5,785 | 27,846 
컴백 반복 MAU| | | | 403 | 1,105 | 1,469 
컴백 반복률| | | |43.10%|37.10%|25.40%
- MAU 내역과 MAU 속성들의 반복률을 계산하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  monthly_user_action AS (
    ...
  )
  , monthly_user_with_type AS (
    ...
  )
  , monthly_users AS (
    ...
  )
  SELECT
    action_month
    , mau
    , new_users
    , repeat_users
    , come_back_users
    , new_repeat_users
    , continuous_repeat_users
    , come_back_repeat_users

    -- PostgreSQL, Redshift, BigQuery
    -- Hive의 경우 NULLIF를 CASE로 변경
    -- SparkSQL, LAG 함수의 프레임에 ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING 지정
    -- 이번 달에 신규 사용자이면서, 해당 월에 신규 리피트 사용자인 사용자의 비율
    , 100.0 * new_repeat_users
      / NULLIF(LAG(new_users) OVER(ORDER BY action_month), 0)
      AS priv_new_repeat_ratio
    
    -- 이전 달에 리피트 사용자이면서, 해당 월에 기존 리피트 사용자인 사용자의 비율
    , 100.0 * continuous_repeat_users
      / NULLIF(LAG(repeat_users) OVER(ORDER BY action_month), 0)
      AS priv_continuous_repeat_ratio
    
    -- 이전 달에 컴백 사용자이면서, 해당 월에 컴백 리피트 사용자인 사용자의 비율
    , 100.0 * come_back_repeat_users
      / NULLIF(LAG(come_back_users) OVER(ORDER BY action_month), 0)
      AS priv_come_back_repeat_ratio
    
  FROM
    monthly_users
  ORDER BY
    action_month
  ;
  ```

#### 원 포인트
- 이번 절의 리포트는 **월 단위**로 집계하므로,
  - 1일에 등록한 사용자가, 30일 이상 사용하지 않으면 **리피트 사용자가 되지 않음**
  - 월의 마지막 날에 등록한 사용자는 2일만 사용해도 **리피트 사용자**로 구분됨
- 판정 기간에 약간 문제가 있긴 하지만
  - 간단하게 추이를 확인하거나, 서비스끼리 비교할 때는 굉장히 유용한 리포트
- 판정 기간의 차이가 신경쓰인다면,
  - 독자적인 정의를 추가로 만들고 집계해야 함

### 8. 방문 종류를 기반으로 성장지수 집계하기
- Growth Hacking Team
  - 서비스를 운영할 때 **사용자 등록**을 포함해
  - 사용자의 **지속 사용**, **리텐션** 등을 높일 수 있는 대책과 서비스 성장을 가속하기 위한 팀
- 성장 지수
  - 서비스의 성장을 **지표화**하거나,
  - 그로스 해킹 팀의 성과를 지표화하는 한 방법

#### 성장지수
- 사용자의 서비스 사용과 관련한 **상태 변화**를 수치화해서
  - 서비스가 성장하는지 알려주는 지표
- 성장 지수가 `1`이상이라면, 서비스가 성장한다는 의미
  - `0`보다 낮다면, 서비스가 퇴보 중
- 서비스 사용과 관련한 상태 변화 패턴
  - `Signup` : 신규 등록하고 사용을 시작
  - `Deactivation` : 액티브 유저 -> 비액티브 유저
  - `Reactivation` : 비액티브 유저 -> 액티브 유저
  - `Exit` : 서비스를 탈퇴하거나 사용 중지
- 사용자의 서비스 등록부터 탈퇴까지
  - **사용 상태**를 기반으로 성장지수를 산출하려면,
  - 상태 변화 판정이 필요함
- 성장 지수 산출
  - `Signup + Reactivation - Deactivation - Exit`
- `서비스를 사용하게 된 사용자`와 `떠난 사용자`를 집계하고
  - 어떤 사용자가 많은지 수치화 해서 나타냄
- 서비스를 `계속 사용하는 사용자`와 `계속 사용하지 않는 사용자`는
  - 성장지수에 영향을 주지 않음
- 성장 지수를 개선하는 두가지 방법
  - `Signup`과 `Reactivation`을 높이는 방법
  - `Deactivation`을 낮추는 방법

#### 성장지수 집계하기
- 성장 지수 집계를 위해서, **사용자의 상태**를 확인해야 함
- 아래의 상태로 **날짜별 판정**
  - 신규 등록(`is_new`)
  - 탈퇴(`is_exit`)
  - 특정 날짜에 서비스 접근(`is_access`)
  - 전라 서비스 접근(`was_access`)
- 성장지수 산출을 위해 사용자 상태를 집계하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  unique_action_log AS (
    -- 같은 날짜 로그를 중복하여 세지 않도록 중복 배제
    SELECT DISTINCT
      user_id
      , substring(stamp, 1, 10) AS action_date
      -- BigQuery
      , substr(stamp, 1, 10) AS action_date
    FROM
      action_log
  )
  , mst_calendar AS (
    -- 집계하고 싶은 기간을 캘린더 테이블로 생성
    -- generate_series 등으로 동적 생성도 가능
              SELECT '2016-10-01' AS dt
    UNION ALL SELECT '2016-10-02' AS dt
    UNION ALL SELECT '2016-10-03' AS dt
    -- 생략
    UNION ALL SELECT '2016-11-04' AS dt
  )
  , target_date_with_user AS (
    -- 사용자 마스터에 캘린더 테이블의 날짜를 target_date로 추가
    SELECT
      c.dt AS target_date
      , u.user_id
      , u.register_date
      , u.withdraw_date
    FROM
      mst_users AS u
      CROSS JOIN
        mst_calendar AS c
  )
  , user_status_log AS (
    SELECT
      u.target_date
      , u.user_id
      , u.regsiter_date
      , u.withdraw_date
      , a.action_date
      , CASE WHEN u.register_date = a.action_date THEN 1 ELSE 0 END AS is_new
      , CASE WHEN u.withdraw_date = a.action_date THEN 1 ELSE 0 END AS is_exit
      , CASE WHEN u.target_date = a.action_date THEN 1 ELSE 0 END AS is_access
      , LAG(CASE WHEN u.target_date = a.action_date TEHN 1 ELSE 0 END)
        OVER(
          PARTITION BY u.user_id
          ORDER BY u.target_date
          -- SparkSQL, 다음과 같이 프레임 지정
          ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING
        ) AS was_access
    FROM
      target_date_With_user AS u
      LEFT JOIN
        unique_action_log AS a
        ON u.user_id = a.user_id
        AND u.target_date = a.action_date
    WHERE
      -- 집계 기간을 등록일 이후로만 필터링
      u.register_date <= u.target_date
      -- 탈퇴 날짜가 포함되어 있으면, 집계 기간을 탈퇴 날짜 이전만으로 필ㅇ터링
      AND (
        u.withdraw_date IS NULL
        OR u.target_date <= u.withdraw_date
      )
  )
  SELECT
    target_date
    , user_id
    , is_new
    , is_exit
    , is_access
    , was_access
  FROM
    user_status_log
  ;
  ```
- 날짜별 사용자 상태를 판정했다면,
  - **성장 지수를 계산하는 패턴**을 계산하고, 최종적으로 성장 지수 계산
- `signup`, `reactivation`, `deactivation`, `exit`, `growth_index` 날짜별 계산
- 매일의 성장지수를 계산하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  unique_action_log AS (
    ...
  )
  , mst_calendar AS (
    ...
  )
  , target_date_with_user AS (
    ...
  )
  , user_status_log AS (
    ...
  )
  , user_growth_index AS (
    SELECT
      *
      , CASE
        -- 어떤 날짜에 신규 등록 또는 탈퇴한 경우 signup | exit으로 판정
        WHEN is_new + is_exit = 1 THEN
          CASE
            WHEN is_new = 1 THEN 'signup'
            WHEN is_exit = 1 THEN 'exit'
          END
        -- 신규 등록과 탈퇴가 아닌 경우 reactivation 또는 deactivation으로 판정
        -- 이때 reactivation, deactivation의 정의에 맞지 않는 경우는 NULL
        WHEN is_new + is_exit = 0 THEN
          CASE
            WHEN was_access = 0 AND is_access = 1 THEN 'reactivation'
            WHEN was_access = 1 AND is_access = 0 THEN 'deactivation'
          END
        --어떤 날짜에 신규 등록과 탈퇴를 함께 했다면(is_new + is_exit = 2) NULL로 지정
        END AS growth_index
    FROM
      user_status_log
  )
  SELECT
    target_date
    , SUM(CASE growth_index WHEN 'signup'         THEN 1 ELSE 0 END) AS signup
    , SUM(CASE growth_index WHEN 'reactivation'   THEN 1 ELSE 0 END) AS reactivation
    , SUM(CASE growth_index WHEN 'deactivation'   THEN -1 ELSE 0 END) AS deactivation
    , SUM(CASE growth_index WHEN 'exit'           THEN -1 ELSE 0 END) AS exit
    -- 성장 지수 정의에 맞게 계산
    , SUM(
        CASE growth_index
          WHEN 'signup'       THEN 1
          WHEN 'reactivation' THEN 1
          WHEN 'deactivation' THEN -1
          WHEN 'exit'         THEN -1
          ELSE 0
        END
      ) AS growth_index
  FROM
    user_growth_index
  GROUP BY
    target_date
  ORDER BY
    target_date
  ;
  ```

#### 서비스 런칭 때부터의 성장지수 추이
- 서비스의 성장지수가 어떤식으로 변하는지,
- 성장 지수를 높이려면 어떻게 해야 하는지
- 성장지수 - 서비스 런칭 직후
  - 사전에 계획한 광고 등으로 인해 **급격하게 증가**
  - 하지만 곧바로, **80% 정도가 비액티브 유저**로 변화
- 런칭 직후의 비약적인 성장이 일단락 되는 시점부터는
  - 어떠한 **사용자 층**이 남는지, 어떻게 하면 **다른 사용자를 계속 유지**할 수 있는지 검토 필요
  - 런칭 시점에 맞추어 광고할 경우, 쉽게 사용자 획득은 가능하나
    - 일시적인 영향에 그칠 뿐, 성장을 지속시키지 못함
- 미디어를 통한 사용자 획득 보다는
  - 서비스(제품)로 사용자를 획득하고, 지속해서 이용하게 만들어야 안정적인 성장 가능
  
#### 원포인트
- 일상적으로 사용하는 서비스(뉴스 사이트 또는 소셜 게임)와,
  - 특정 목적이 발생할 때 사용하는 서비스(음식점 검색 서비스, EC 사이트)는
  - 사용 빈도가 서로 다름
- 성장지수 계산을 위해 사용하는 지표를 집계할 때는
  - **서비스의 특성**에 맞게
    - 날짜, 주차, 월차 등의 적절한 집계 기간 선택 필요

### 9. 지표 개선 방법 익히기
- 지금까지 사용자를 파악하기 위한 리포트와 SQL을 소개
- 사용자 파악은 목표가 아니라
  - **지표를 개선**하려는 과정에 지나지 않음
- 서비스 제공자라면, 당연히 **매출** 또는 **사용자 수**를 늘리는 것이 목표
- 목표를 달성하려면
  - 사용자를 파악하고, 더 좋은 대책을 검토해야 함

#### 지표를 향상 시키려면
- 다음과 같은 방법대로 분석 진행
  - 1) 달성하고 싶은(높이고 싶은) 지표를 결정
  - 2) 사용자 행동 중에서 **지표에 영향을 많이 줄 것**으로 보이는 행동 결정
  - 3) 2)에서 결정한 행동 여부와 횟수 집계하고,
    - 1)에서 **결정한 지표를 만족하는 사용자의 비율** 비교
- 달성하고 싶은 행동이 있는 경우
  - **어떤 액션이 영향을 주었는지 확인**하는 것 이 포인트

### 예시
- 일반적인 데모 그래픽 정보를 기반으로
  - **남성**보다 **여성**의 다음날 지속률이 높다고 판명되었다면,
    - 이를 기반으로 광고 배너, 사이트 디자인, 사이트 컨셉 등을 재검토 할 수 있음
- 하지만, 대규모 개선이 필요하거나,
  - 회원 구성비가 약간 바뀌는 것만으로는 **서비스가 개선**되었다고 말할 수 없음
  - 당연히 **컨셉**을 확인하거나, 대상 고객층을 포함하는지 확인할 때는 중요하겠지만,
    - 개선의 관점에서는 **어떤 액션이 결과에 영향을 주었는지 확인하는 것**이 중요

### 시점 집계 예시
- 글의 업로드와 댓글 수를 늘리고 싶은 경우
  - 팔로우된 사람 수에 따라 차이가 있는가?
  - 팔로우하는 사람 수에 따라 차이가 있는가?
  - 프로필 사진을 등록한 사람에 따라 차이가 있는가?
- 신규 사용자의 **리피트율**을 개선하고 싶은 경우
  - 등록한 달에 올린 글의 수에 따라 차이가 있는가
  - 등록한 달에 팔로우한 사람 수에 따라 차이가 있는가
  - 등록 다음날부터 7일 이내의 사용 일수에 따라 차이가 있는가
- CVR을 개선하고 싶은 경우
  - 구매 전에 상세 페이지를 본 횟수에 따라 차이가 있는가
  - 구매 전에 관심 상품 기능 사용 여부에 따라 차이가 있는가

### 정리
- 개선하고 싶은 지표에 **다양한 가정**을 세워 검증해보면
  - 주력해야 하는 부분과, 대책 찾기가 가능
  - 직접 담당하는 서비스에서 다양한 관점으로 생각 필요
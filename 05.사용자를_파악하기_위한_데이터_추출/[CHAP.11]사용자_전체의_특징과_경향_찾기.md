## 11. 사용자 전체의 특징과 경향 찾기
### 서비스 제공에 대한 가치
- 서비스를 제공하는 측에서 사용자와 관련된 정보로 알고 싶은 것(2)
    - 사용자의 속성(나이, 성별, 주소지)
    - 사용자의 행동(구매한 상품, 사용한 기능, 사용자의 빈도
- `어떤 속성의 사용자가 사용중인가?` & `어떻게 사용하는가?`
    - 위 두개를 파악하고 **서비스 개선**을 검토해야 함
- 이번 장의 목표
    - **사용자의 속성** 또는 **행동에 관련된 정보**를 집계하여
        - **사용자 행동**에 대한 조사
    - 서비스 개선시의 실마리가 될 수 있는 리포트를 만드는 SQL
    
### 샘플 데이터
- 두가지 테이블
    - 사용자 마스터 테이블
        - 가입시 사용자가 성별과 날짜등을 입력하고 저장
        - 가입 시점의 날짜와 가입에 사용한 장치를 추가 기록
        - 스키마
          ```sql
          user_id STRING,
          sex STRING,
          birth_date DATE,
          register_date DATE,
          register_device STRING,
          withdraw_date STRING
          ```
    - 액션 로그 테이블
        - 페이지 열람, 관심 상품 등록, 카트 추가, 구매, 상품 리뷰등의 각 액션에 이름 명시
        - 로그인한 사용자의 경우 `user_id` 추가 기록
        - 스키마
          ```sql
          session STRING,
          user_id STRING,
          action STRING,
          category STRING,
          products STRING,
          amount INT,
          stamp DATETIME
          ```
- 액션 로그 테이블에 대한 팁
  - 일반적으로 서비스 관련된 **업무 데이터**가 저장된 db에는
    - 관심 상품 등록, 카트 추가, 구매, 댓글 등의 각각의 테이블 존재
  - 분리하지 않고, 위와같이 하나로 구성할 경우
    - 별도의 `JOIN`이나 `UNION` 없이  데이터를 다룰 수 있음
    
### 사용자의 액션 수 집계하기
- `COUNT`, `COUNT(DISTINCT ~)`, `ROLLUP`
- `UU(Unique Users)`

#### 액션과 관련된 지표 집계
- 사용자들이 **특정 기간**동안 기능을 얼마나 사용하는지 집계
- `특정 액션 UU/ 전체 액션 UU = 사용률(usage_rate)`
- 1명당 액션수(`count_per_user`)
- 액션 수의 비율을 계산하는 쿼리
  - `PostgreSQL`, `Hive`, `RedShift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  stats AS (
  -- 로그 전체의 유니크 사용자 수 구하기
  SELECT COUNT(DISTINCT session) AS total_uu
  FROM action_log
  )
  SELECT
    l.action
    -- action uu
    , COUNT(DISTINCT l.session) AS action_uu
    -- action count
    , COUNT(1) AS action_count
    -- total uu
    , s.total_uu
    -- usage_rate
    , 100.0 * COUNT(DISTINCT l.session) / s.total_uu AS usage_rate
    -- count_per_user
    , 1.0 * COUNT(1) / COUNT(DISTINCT l.session) AS count_per_user
  FROM
    action_log AS l
    -- 로그 전체의 유니크 사용자 수를 모든 레코드에 결합
    CROSS JOIN 
          stats AS s
    GROUP BY
      l.action, s.total_uu
    ;
  ```
  - `CROSS JOIN` : catesian product
  - 전체 `UU`를 구하고 `CROSS JOIN`해서 액션 로그로 결합

#### 로그인 사용자와 비로그인 사용자를 구분하여 집계
- 로그인과 비로그인 구별  
- 서비스에 대한 **충성도**가 높은 사용자와, 낮은 사용자의 경향을 파악 할 수 있음
- 로그인, 비로그인, 회원, 비회원 판별 시
  - 로그 데이터에 `session` 정보가 있어야 함
- 로그인 상태를 판별하는 쿼리
  - `PostgreSQL`, `Hive`, `RedShift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  action_log_with_status AS (
    SELECT
      session
      , user_id
      , action
      -- user_id = NULL OR user_id != '', login
      , CASE WHEN COALESCE(user_id, '') <> '' THEN 'login' ELSE 'guest' END
        AS login_status
    FROM action_log
  )
  SELECT *
  FROM
    action_log_with_status
  ;
  ```
- `login_status`가 `guest`와 `login`인 경우
  - 모두 로그인/비로그인을 따지지 않고, `all`에 함께 집계
  - `ROLLUP`구문을 사용
- `ROLLUP`을 지원하지 않는 `RedShift`와 `BigQuery` 에서는
  - `UNITON ALL`을 사용
- 로그인 상태에 따라 액션 수 등을 따로 집계하는 쿼리
  - `PostgreSQL`, `Hive`, `SparkSQL`
  ```sql
  WITH
  action_log_with_status AS (
    ...
  )
  SELECT
    COALESCE(action, 'all') AS action
    , COALESCE(login_status, 'all') AS login_status
    , COUNT(DISTINCT session) AS action_uu
    , COUNT(1) AS action_count
  FROM
    action_log_with_status
  GROUP BY
    -- PostgreSQL, SparkSQL
    ROLLUP(action, login_status)
    -- Hive
    action, login_status WITH ROLLUP
  ;
  ```
  - 로그 정보에 `user_id` 정보를 기반으로 집계한 데이터
    - 비로그인 사용자가 로그인하게 되면  
      - 각각의 액션에 1씩 추가 됨
  - 반대로 `all`은 `session`을 기반으로 집계 
    - `guest + login != all`임에 유의할 것

#### 회원과 비회원을 구분하여 집계
- 로그인하지 않은 상태에서도, 
  - 이전에 한 번이라도 로그인 했다면, 회원으로 계산하기
- `session_log_with_status`의 정의를 다음과 같이 변경하여  
  - 회원 상태를 추가함
- 회원 상태를 판별하는 쿼리
  - `PostgreSQL`, `Hive`, `RedShift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  action_log_with_status AS (
    SELECT
      session
      , user_id
      , action
      -- log를 timestamp 순으로 나열, 한번이라도 로그인했을 경우,
      -- 이후의 모든 로그 상태를 member로 설정
      , CASE
        WHEN
            COALESCE(MAX(user_id)
              OVER(PARTITION BY session ORDER BY stamp
                ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
              , '') <> ''
            THEN 'member'
          ELSE 'none'
        END AS member_status
      , stamp
    FROM
      action_log
    )
    SELECT *
    FROM
      action_log_with_status
    ;
  ```
- 윈도 함수에서 `session` 별로 파티션 정의 
- 해당 `session`에서 한 번이라도 로그인했다면
  - `MAX(user_id)`로 사용자 id를 추출할 수 있음
- 로그인 하기 이전의 상태를 **비회원**으로 다루고 싶다면  
- 윈도 함수 내부에 `ORDER BY timestamp`를 지정하면 됨

#### 정리
- 로그인하지 않은 상태일 경우
  - `user_id`의 컬럼 값이 빈 문자열 또는 `NULL`일 수 있다 판단하여
  - `COALESCE` 함수를 사용해 `EMPTY_STRING`로 변환
- 로그인 하지 않은 때의 사용자 id를 `EMPTY_STRING`로 저장시,
  - `COUNT(DISTINCT user_id)` 결과에 1이 추가 됨
- 따라서 `COUNT(DISTINCT user_id)`로 사용자 수를 정확히 추출하려면
  - `user_id is NULL`로 지정하는 것이 좋음
- 비어있는 값을 `NULL`로 할지 `EMPTY_STRING`과 같은 특수값으로 처리할지에 따라  
  - 쿼리의 최적화 방법이 달라질 수 있음
- 따라서 `COALESCE` 함수 또는 `NULLIF` 함수를 사용하여, 변환하는 방법을 기억할 것

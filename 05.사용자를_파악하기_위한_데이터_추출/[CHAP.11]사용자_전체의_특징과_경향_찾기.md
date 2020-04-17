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

### 연령별 구분 집계하기
- 회원 정보를 저장하는 서비스는
  - 해당 서비스의 사용자를 파악하고
  - 의도한대로 서비스가 사용되는지 파악해야 함
- 또한 광고 디자인과, 다양한 캐치 프레이즈를 검토하려면
  - **사용자 속성 집계**
- 사용자의 속성을 정의하고 집계하면, 다양한 리포트 제작이 가능
- 예시 : 연령별 구분 목록
  - C : Child, 4~12y, w/m
  - T : Teenager, 13~19y, w/m
  - M1 : Male, 20~34y, m
  - M2 : Male, 35~49, m
  - F3 : Female, 50~, f
- 연령별 구분하여 집계
  - 처음 가입시의 나이를 기반으로 db에 저장할 경우, 변동 우려
  - 일반적으로 **나이**를 저장하지 않고, 생일을 기반으로 리포트 만드는 시점 집계
- 사용자의 생일을 계산하는 쿼리
  - `PostgreSQL`, `Hive`, `RedShift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  -- 생일과 특정 날짜를 정수로 표현한 결과
  mst_users_with_int_birth_date AS (
    SELECT
      * 
      -- 특정 날짜의 정수 표현(2017.01.01)
      , 20170101 AS int_specific_date
      
      -- 문자열로 구성된 생년월일 -> 정수 표현 변환
      -- PostgreSQL, Redshift
      , CAST(replace(substring(birth_date, 1, 10), '-', '') AS integer) AS int_birth_date
      -- BigQuery
      , CAST(replace(substr(birth_date, 1, 10), '-', '') AS int64) AS int_birth_date
      -- Hive, SparkSQL
      , CAST(regexp_replace(substring(birth_date, 1, 10), '-', '') AS int) AS int_birth_date
    FROM
      mnt_users
  )
  -- 생일 정보를 부여한 사용자 마스터
  , mst_users_with_age AS (
    SELECT
      *
      -- 2017.01.01의 나이
      , floor((int_specific_date - int_birth_date) / 10000) AS age
    FROM mst_users_with_int_birth_date
  )
  SELECT
    user_id, sex, birth_date, age
  FROM
    mst_users_with_age
  ;
  ```
- 성별과 연령으로 연령별 구분을 계산하는 쿼리
  ```sql
  WITH
  mst_users_with_int_birth_date AS (
    ...
  )
  , mst_users_with_age AS (
    ...
  )
  , mst_users_with_category AS (
    SELECT
      user_id
      , sex
      , age
      , CONCAT (
        CASE
          WHEN 20 <= age THEN sex
          ELSE ''
        END
      , CASE
          WHEN age BETWEEN 4 AND 12 THEN 'C'
          WHEN age BETWEEN 13 AND 19 THEN 'T'
          WHEN age BETWEEN 20 AND 34 THEN '1'
          WHEN age BETWEEN 35 AND 49 THEN '2'
          WHEN age >= 50 THEN '3'
        END
      ) AS category
    FROM
      mst_users_with_age
  )
  SELECT *
  FROM
    mst_users_with_category
  ;
  ```
- `category` 컬럼에서, 연령별 구분 계산
- 연령이 20살 이상이라면, `M`과 `F`를 출력
- 이어 나이구분에 따라 `C`, `T`, `1`, `2`, `3` 출력
- 두 값을 `CONCAT`으로 결합
- 연령이 3살 이하일 경우
  - 연령 구분 코드가 `NULL`이 되어, `CONCAT`의 결과도 `NULL`
- 연령별 구분의 사람 수를 계산하는 쿼리
  - `PostgreSQL`, `Hive`, `RedShift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  mst_users_with_age AS (
    ...
  )
  , mst_users_with_category AS (
    ...
  )
  SELECT
    category
    , COUNT(1) AS user_count
  FROM
    mst_users_with_category
  GROUP BY
    category
  ;
  ```

#### 정리
- 연령을 단순하게 계산하면, 특징 파악이 어렵기 때문에
  - `Demographic` 등에 활용하기 힘듦
- 그에 따라 서비스별 `M1`, `F1`처럼
  - 사용자 속성을 정의하는 것이 적절하지 않을 수 있음
- 이럴 때는 독자적으로 새로운 기준을 정의하기
  - `CASE` 식 부분 변경 등

### 연령별 구분의 특징 추출
- 서비스의 사용 형태가
  - 사용자 속성에 따라 다르다는 것을 확인하면
  - 상품 또는 기사를 **사용자 속성**에 맞게 추천 가능
- 이전 절의 내용인 **연령별 구분**을 사용하여
  - 각 구매한 상품의 카테고리를 집계하기
- 연령별 구분과 카테고리를 집계하는 쿼리
  - `PostgreSQL`, `Hive`, `RedShift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  mst_users_with_int_birth_date AS (
    ...
  )
  , mst_users_with_age AS (
    ...
  )
  , mst_users_with_category AS (
    ...
  )
  SELECT
    p.category AS product_category
    , u.category AS user_category
    , COUNT(*) AS purchase count
  FROM
    action_log AS p
  JOIN
    mst_users_with_category AS u ON p.user_id = u.user_id
  WHERE
    -- 구매 로그만 선택
    action = 'purchase'
  GROUP BY
    p.category, u.category
  ORDER BY
    p.category, u.category
  ;
  ```
- 연령별 * 카테고리별을 활용하여 다양한 분석이 가능
  - M1계층에서 드라마가 인기
  - 드라마는 모든 계층에서 인기
  - 액션은 M1 계층에서 인기
- 연령별 * 카테고리별 vs 카테고리별 * 연령별로 나눠서 분석이 가능함

#### 정리
- **ABC 분석** 및 **구성비누계**를 리포트에 포함하면
  - 리포트의 내용 전달성 향상이 가능

### 사용자의 방문 빈도 집계
- SUM의 윈도 함수 사용
- 사용자가 일주일 또는 한 달 동안 서비스를 **얼마나 사용하는지**알면 업무 분석에 도움
- 예시
  - 뉴스 사이트에서의 방문에 따른 차이
    - 일주일에 한번만 방문하는 사용자
    - 매일 방문하는 사용자
- 서비스를 한 주동안 **며칠 사용**하는 사용자가 몇 명인지 집계
- 일주일 동안의 사용자 사용 일수와 구성비
- `action_log` 테이블
  - 사용자 ID, 액션, 날짜
  - 사용자 ID별로 `DISTINCT`를 적용하여 사용 일 수 집계
- 한 주에 며칠 사용되었지는지를 집계하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  action_log_with_dt AS (
    SELECT *
    -- 타임 스탬프에서 날짜 추출하기
    -- PostgreSQL, Hive, Redshift, SparkSQL의 경우 substring으로 날짜 추출
    , substring(stamp, 1, 10) AS dt
    -- PostgerSQL, Hive, BigQuery, SparkSQL의 경우 substr 사용
    , substring(stamp, 1, 10) AS dt
    FROM action_log
  )
  , action_day_count_per_user AS (
    SELECT
      user_id
      , COUNT(DISTINCT dt) AS action_day_count
    FROM
      action_log_with_dt
    WHERE
      -- 2016년 11월 1일부터 11월 7일까지의 한 주 동안을 대상으로 지정
      dt BETWEEN '2016-11-01' AND '2016-11-07'
    GROUP BY
      user_id
  )
  SELECT
    action_day_count
    , COUNT(DISTINCT user_id) AS user_count
  FROM
    action_day_count_per_user
  GROUP BY
    action_day_count
  ORDER BY
    action_day_count
  ;
  ```
- 구성비와 구성비누계를 계산하는 쿼리
  ```sql
  WITH
  action_day_count_per_user AS (
    ...
  )
  SELECT
    action_day_count
    , COUNT(DISTINCT user_id) AS user_count

    -- 구성비
    , 100.0
      * COUNT(DISTINCT user_id)
      / SUM(COUNT(DISTINCT user_id)) OVER() AS composition_ratio
    
    -- 구성비누계
    , 100.0
      * SUM(COUNT(DISTINCT user_id))
        OVER(ORDER BY action_day_count
          ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
      / SUM(COUNT(DISTINCT user_id)) OVER() AS cumulative_ratio
  FROM
    action_day_count_per_user
  GROUP BY
    action_day_count
  ORDER BY
    action_day_count
  ;
  ```

### 벤 다이어그램으로 사용자 액션 집계하기
- `SIGN` 함수, `SUM` 함수, `CASE` 식, `CUBE` 구문
- 여러 기능의 사용 현황을 조사한 뒤
  - 제공하는 기능을 사용자가 받아들이는지,
  - 예상대로 사용하는지 확인
- 벤 다이어그램으로 표현하기
  - `purchase`, `review`, `favorite`이라는 3개의 액션을 행한 로그가 존재하는지를
    - `0`과 `1` 플래그로 표기
- 사용자의 액션 플래그를 집계하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  user_action_flag AS (
    -- 사용자가 액션을 했으면 1, 하지 않았을 경우 0 플래그
    SELECT
      user_id
      , SIGN(SUM(CASE WHEN action = 'purchase' THEN 1 ELSE 0 END)) AS has_purchase
      , SIGN(SUM(CASE WHEN action = 'review' THEN 1 ELSE 0 END)) AS has_review
      , SIGN(SUM(CASE WHEN action = 'favorite' THEN 1 ELSE 0 END)) AS has_favorite
    FROM
      action_log
    GROUP BY
      user_id
  )
  SELECT *
  FROM user_action_flag;
  ```
- 대부분의 리포트 도구에는
  - 위 쿼리의 결과 데이터를 파일로 입력하면, 벤 다이어그램으로 만들어주는 기능이 있음
- 벤 다이어 그램에 필요한 데이터 집계하기
- 구매 액션만 한 사용자 수 또는 구매와 리뷰 액션을 한 사용자 수처럼
  - **하나 액션** 또는 **두개의 액션**을 한 사용자가 몇명인지 계산 필요
- 표준 SQL에 정의되어 있는 `CUBE` 구문 사용하면 쉽게 계산 가능
- `CUBE`구문을 사용하여 모든 액션 조합에 대한 사용자 수를 세는 쿼리
  - 단, `PostgreSQL`만 CUBE 구문이 구현 됨
- 모든 액션 조합에 대한 사용자 수 계산
  - `PostgreSQL`
  ```sql
  WITH
  user_action_flag AS (
    ...
  )
  , action_venn_diagram AS (
    -- cube를 사용하여 모든 액션 조합 구하기
    SELECT
      has_purchase
      , has_review
      , has_favorite
      , COUNT(1) AS users
    FROM
      user_action_flag
    GROUP BY
      CUBE(has_purchase, has_revicew, has_favorite)
  )
  SELECT *
  FROM action_venn_diagram
  ORDER BY
    has_purchase, has_review, has_favorite
  ;
  ```
  
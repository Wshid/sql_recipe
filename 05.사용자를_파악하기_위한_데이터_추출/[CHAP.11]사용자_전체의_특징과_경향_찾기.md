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
- has_purchase, has_review, has_favorite 컬럼의 값이 없는 경우,
  - `NULL` 레코드가 존재함.
  - 해당 액션의 시행 여부를 모르는 경우
- `CUBE` 구문을 사용하지 않고 표준 SQL 구문만으로 작성한 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  user_adction_flag AS (
    ...
  )
  , action_venn_diagram AS (
    -- 모든 액션 조합을 개별적으로 구하고 UNION ALL로 결합

    -- 3개의 액션을 모두 한 경우 집계
    SELECT has_purhcase, has_review, has_favorite, COUNT(1) AS users
    FROM user_action_flag
    GROUP BY has_purchase, has_review, has_favorite

    -- 3개의 액션 중에서 2개의 액션을 한 경우 집계
    UNION ALL
      SELECT NULL AS has_purhcase, has_review, has_favorite, COUNT(1) AS users
      FROM user_action_flag
      GROUP BY has_review, has_favorite
    UNION ALL
      SELECT has_purchase, NULL AS has_review, has_favorite, COUNT(1) AS users
      FROM user_action_flag
      GROUP BY has_purchase, has_favorite
    UNION ALL
      SELECT has_purhcase, has_review, NULL AS has_favorite, COUNT(1) AS users
      FROM user_action_flag
      GROUP BY has_purchase, has_review
    
    -- 3개의 액션 중에서 하나의 액션을 한경우 집계
    UNION ALL
      SELECT NULL AS has_purhcase, NULL AS has_review, has_favorite, COUNT(1) AS users
      FROM user_action_flag
      GROUP BY has_favorite
    UNION ALL
      SELECT NULL AS has_purhcase, has_review, NULL AS has_favorite, COUNT(1) AS users
      FROM user_action_flag
      GROUP BY has_review
    UNION ALL
      SELECT has_purhcase, NULL AS has_review, NULL AS has_favorite, COUNT(1) AS users
      FROM user_action_flag
      GROUP BY has_purhcase

    -- 액션과 관계 없이 모든 사용자 집계
    UNION ALL
      SELECT NULL AS has_purhcase, NULL AS has_review, NULL AS has_favorite, COUNT(1) AS users
      FROM user_action_flag
  )
  
  SELECT *
  FROM action_venn_diagram
  ORDER BY
    has_purchase, has_review, has_favorite
  ;
  ```
- 모든 미들웨어에서 동작하나,
  - `UNION ALL`을 많이 사용하기 때문에, 성능이 좋지 않음
- `Hive`, `BigQuery`, `SparkSQL`은
  - **Chap.7-4**를 활용하여
  - 액션 플래그의 컬럼에 유사적으로 `NULL`을 포함한 레코드를 생성하여
  - `CUBE`를 사용할 때와 같은 결과를 얻을 수 있음
- 유사적으로 `NULL`을 포함한 레코드를 추가하여 `CUBE` 구문과 같은 결과를 얻는 쿼리
  - `Hive`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  user_action_flag AS (
    ...
  )
  , action_venn_diagram AS (
    SELECT
      mod_has_purchase AS has_purchase
      , mod_has_review AS has_review
      , mod_has_favorite AS has_favorite
      , COUNT(1) AS users
    FROM
      user_action_flag
      -- 각 컬럼에 NULL을 포함하는 레코드 유사적으로 추가
      -- BigQuery, CROSS JOIN unnest 함수 사용
      CROSS JOIN unnest(array[has_purchase, NULL]) AS mod_has_purhcase
      , CROSS JOIN unnest(array[has_review, NULL]) AS mod_has_review
      , CROSS JOIN unnest(array[has_favorite, NULL]) AS mod_has_favorite

      -- Hive, SparkSQL의 경우 LATERAL VIEW와 explode 함수 사용
      LATERAL VIEW explode(array(has_purchase, NULL)) e1 AS mod_has_purchase
      , LATERAL VIEW explode(array(has_review, NULL)) e2 AS mod_has_review
      , LATERAL VIEW explode(array(has_favorite, NULL)) e3 AS mod_has_favorite
    GROUP BY
      mod_has_purchase, mod_has_review, mod_has_favorite
  )
  SELECT *
  FROM action_venn_diagram
  ORDER BY
    has_purchase, has_review, has_favorite
  ;
  ```
- 벤 다이어그램을 만들기 위해 데이터를 가공하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  user_action_flag AS (
    ...
  )
  , action_venn_diagram AS (
    ...
  )
  SELECT
    -- 0,1 플래그를 문자열로 가공
    CASE has_purchase
      WHEN 1 THEN 'purchase' WHEN 0 THEN 'not purhcase' ELSE 'any'
    END AS has_purchase
    , CASE has_review
        WHEN 1 THEN 'review' WHEN 0 THEN 'not review' ELSE 'any'
      END AS has_review
    , CASE has_favorite
        WHEN 1 THEN 'favorite' WHEN 0 THEN 'not favorite' ELSE 'any'
      END AS has_favorite
    , users
      -- 전체 사용자 수를 기반으로 비율 구하기
    , 100.0 * users
        / NULLIF(
          -- 모든 액션이 NULL인 사용자 수 = 전체 사용자 수,
          -- 해당 레코드의 사용자 수를 Windows 함수로 구하기
          SUM(CASE WHEN has_purhcase IS NULL
                    AND has_review   IS NULL
                    AND has_favorite IS NULL
              THEN users ELSE 0 END) OVER()
          , 0)
        AS ratio
    FROM
      action_venn_diagram
    ORDER BY
      has_purchase, has_review, has_favorite
  ;
  ```
  - 가독성이 높은 형식이 되도록 결과 가공
  - 각 액션 구조에서의 **사용자 수**와 **구성 비율** 포함
  - 각 레코드 별의 의미
    - `has_purhcase / has_review / has_favorite` 형태
    - `purchase / any / any` : 구매 액션을 한 사용자
    - `purchase / not_review / any` : 구매 액션을 했으나, 리뷰를 작성하지 않은 사용자

#### 정리
- 예시로 들었던 `EC`외에, `SNS`등의 사이트에서 활용 가능
  - 글을 작성하지 않고, 다른 사람의 글만 확인하는 사용자
  - 글을 많이 작성하는 사용자
  - 글을 거의 작성하지 않지만, 댓글은 많이 작성하는 사용자
  - 글과 댓글 모두 적극적으로 작성하는 사용자
- 어떠한 대첵을 수행했을 때, 해당 대책으로 효과가 발생한 사용자가 얼마나 되는지,
  - 벤 다이어그램으로 확인하면
  - 대책을 제대로 세웠는지(대상을 제대로 가정했는지) 확인 가능

### Decile 분석을 사용해, 사용자를 10단계 그룹으로 나누기
- `NTILE` 함수
- 데이터를 10단계로 분할하여 중요도를 파악하는 방법
  - `Decile`은 1/10을 의미
- Decile 분석 과정
  - 사용자를 구매 금액이 많은 순으로 정렬
  - 정렬된 사용자 상위부터 10%씩 `Decile 1 ~ Decile 10`으로 그룹 할당
  - 각 그룹의 구매 금액 합계 집계
  - 전체 구매 금액에 대해 각 `Decile`의 구매 금액 비율(**구성비**)를 계산
  - 상위에서 누적으로 어느 정도의 비율을 차지하는지 **구성비누계** 집계
- 일단, 사용자를 구매 금액이 많은 순서로 정렬하고,
  - 정렬된 사용자의 상위 10%씩 `Decile 1 ~ Decile 10`까지의 그룹 할당
- 같은 수로 데이터 그룹을 만들 때는, `NTILE` 윈도 함수 사용
- 구매 금액이 많은 수너로 사용자 그룹을 10등분 하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  user_purchase_amount AS (
    SELECT
      user_id
      , SUM(amount) AS purchase_amount
    FROM
      action_log
    WHERE
      action = 'purchase'
    GROUP BY
      user_id
  )
  , users_with_decile AS (
    SELECT
      user_id
      , purchase_amount
      , ntile(10) OVER (ORDER BY purchase_amount DESC) AS decile
    FROM
      user_purchase_amount
  )
  SELECT *
  FROM users_with_decile
  ```
- `OVER` 뒤의 구문 분석
  - `ORDER BY`를 사용할 경우, 누계 또는 rank와 같은 내용
  - `PARTITION BY`를 사용할 경우, 해당 내용별 group by가 되는듯 하다.
  - `PARTITION BY ... ORDER BY ...`를 사용할 경우 그룹별 분할 한 후, order by 조건이 먹는듯함
  - `ORDER BY ... PARTITION BY ...` 구문은 존재하지 않음
- 이어서 각 그룹의 합계, 평균 구매 금액, 누계 구매 금액, 전체 구매 금액 등의 집약 계산
  - `GROUP BY`로 `Decile`들을 집약하고,
  - 집약 함수와 윈도 함수를 조합하여 한번에 이러한 값들을 계산
- 10분할 한 Decile들을 집계하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  user_purchase_amount AS (
    ...
  )
  , users_with_decile AS (
    ...
  )
  , decile_with_purhcase_amount AS (
    SELECT
      decile
      , SUM(purchase_amount) AS amount
      , AVG(purchase_amount) AS avg_amount
      , SUM(SUM(purchase_amount)) OVER (ORDER BY decile) AS cumulative_amount
      , SUM(SUM(purchase_amount)) OVER () AS total_amount
    FROM
      users_with_decile
    GROUP BY
      decile
  )
  SELECT *
  FROM
    decile_with_purchase_amount
  ;
  ```
- 마지막으로 구성비와 구성 비누계 계산
- 구매액이 많은 `Decile` 순서로 구성비와 구성비누계를 계산하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH user_purchase_amount AS (
    ...
  )
  , users_with_decile AS (
    ...
  )
  , decile_with_purchase_amount AS (
    ...
  )
  SELECT
    decile
    , amount
    , avg_amount
    , 100.0 * amount / total_amount AS total_ratio
    , 100.0 * cumulative_amount / total_amount AS cumulative_ratio
  FROM
    decile_with_purchase_amount;
  ```

#### 정리
- `Decile` 분석을 시행하고,
  - `Decile`의 특징을 다른 분석 방법으로 세분화하여 조사하면
  - 사용자의 속성 자세하게 파악이 가능함
- `Decile 7 ~ 10`은 정착되지 않은 고객을 나타냄
  - 메일 매거진 등으로 **retention**(고객 유지 비율)을 높이는 등의 대책 마련 강구
  - 만약 메일 매거진을  이미 보내고 있다면,
    - 메일 매거진을 보낼 때 추가적인 데이터를 수집하여,
    - 해당 그룹에 해당하는 사람들의 속성 관련 데이터를 더 수집하고 활용 가능

### RFM 분석으로 사용자를 3가지 관점의 그룹으로 나누기
- `CASE`식, `generate_series` 함수
- `Decile 분석`의 문제
  - 검색 기간이 너무 길면,
    - 과거에는 우수고객이었어도,
    - 현재에는 다른 서비스를 사용하는 휴면 고객이 포함될 수 있음
  - 반대로, 검색기간이 단기간이라면
    - 정기적으로 구매하는 안정 고객이 포함되지 않고,
    - 해당 기간 동안에만 일시적으로 많이 구매한 사용자가, 우수 고객 취급
- `RFM 분석`은 `Decile` 분석보다도, 자세하게사용자를 그룹으로 나눌 수 있는 방법

#### RFM 분석의 3가지 지표 집계
- **Recency** : 최근 구매일
  - 최근 무언가를 **구매한 사용자**를 우량 고객으로 취급
- **Frequency** : 구매 횟수
  - 사용자가 구매한 횟수를 세고, 많을수록 우량 고객으로 취급
-  **Monetary** : 구매 금액 합계
   - 사용자의 구매 금액 합계를 집계하고, 금액이 높을수록 우량 고객

#### RFM 집계 쿼리
- 사용자별로 FRM을 집계하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  purchase_log AS (
    SELECT
      user_id
      , amount
      -- timestamp를 기반으로 날짜 추출
      -- PostgreSQL, Hive, Redshift, SparkSQL, substring 날짜 추출
      , substring(stamp, 1, 10) AS dt
      -- PostgreSQL, Hive, BigQuery, SparkSQL, substr 사용
      , substr(stamp, 1, 10) AS dt
    FROM
      action_log
    WHERE
      action = 'purchase'
  )
  , user_rfm AS (
    SELECT
      user_id
      , MAX(dt) AS recent_date
      -- PostgreSQL, Redshift, 날짜 형식간 빼기 연산 가능
      , CURRENT_DATE - MAX(dt::date) AS recency
      -- BigQuery, date_diff 함수
      , date_diff(CURRENT_DATE, date(timestamp(MAX(dt))), day) AS recency
      -- Hive, SparkSQL, datediff 함수
      , datediff(CURRENT_DATE(), to_date(MAX(dt))) AS recency
      , COUNT(dt) AS frequency
      , SUM(amount) AS mometary
    FROM
      purchase_log
    GROUP BY
      user_id
  )
  SELECT *
  FROM
    user_rfm
  ```

#### RFM 랭크 정의하기
- **RFM 분석**에서는 **3개의 지표**를 각각 **5개의 그룹**으로 나누는게 일반 적
- 총 125개의 그룹으로 사용자를 나눠 파악
- 단계 정의
  >**랭크**|**R: 최근 구매일**|**F: 누적 구매 횟수**|**M: 누계 구매 금액**
  >-----|-----|-----|-----
  >5|14일 이내|20회 이상|300만원 이상
  >4|28일 이내|10회 이상|100만원 이상
  >3|60일 이내|5회 이상|30만원 이상
  >2|90일 이내|2회 이상|5만원 이상
  >1|91일 이상|1회|5만원 미만
- 사용자들의 RFM 랭크를 계산하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  user_rfm AS (
    ...
  )
  , user_rfm_rank AS (
    SELECT
      user_id
      , recent_date
      , recency
      , frequency
      , monetary
      , CASE
          WHEN recency < 14 THEN 5
          WHEN recency < 28 THEN 4
          WHEN recency < 60 THEN 3
          WHEN recency < 90 THEN 2
        ELSE 1
        END AS r
      , CASE
          WHEN 20 <= frequency THEN 5
          WHEN 10 <= frequency THEN 4
          WHEN 5 <= frequency THEN 3
          WHEN 2 <= frequency THEN 2
          WHEN 1 <= frequency THEN 1
        END AS f
      , CASE
          WHEN 300000 <= monetary THEN 5
          WHEN 100000 <= monetary THEN 4
          WHEN 30000 <= monetary THEN 3
          WHEN 5000 <= monetary THEN 2
        ELSE 1
        END AS m
  )
  SELECT *
  FROM
    user_rfm_rank
  ;
  ```
- 각 그룹의 사람 수를 확인하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  user_rfm AS (
    ...
  )
  , user_rfm_rank AS (
    ...
  )
  , mst_rfm_index AS (
    -- 1부터 5까지의 숫자를 가지는 테이블
    -- PostgreSQL, generate_series 함수 등으로 대체 가능
    SELECT 1 AS rfm_index
    UNION ALL SELECT 2 AS rfm_index
    UNION ALL SELECT 3 AS rfm_index
    UNION ALL SELECT 4 AS rfm_index
    UNION ALL SELECT 5 AS rfm_index
  )
  , rfm_flag AS (
    SELECT
      m.rfm_index
      , CASE WHEN m.rfm_index = r.r THEN 1 ELSE 0 END AS r_flag
      , CASE WHEN m.rfm_index = r.f THEN 1 ELSE 0 END AS f_flag
      , CASE WHEN m.rfm_index = r.m THEN 1 ELSE 0 END AS m_flag
    FROM
      mst_rfm_index AS m
    CROSS JOIN
      user_rfm_rank AS r
  )
  SELECT
    rfm_index
    , SUM(r_flag) AS r
    , SUM(f_flag) AS f
    , SUM(m_flag) AS m
  FROM
    rfm_flag
  GROUP BY
    rfm_index
  ORDER BY
    rfm_index DESC
  ```
- 극단적으로 적은 사용자 수의 그룹이 발생한다면, **RFM 랭크 정의**를 수정해야 함
- 총 125개의 그룹을 3차원으로 보았을 때
  - 우량 고객 : RFM
  - 신규 우량 고객 : RfM
  - 신규 고객 : Rfm
  - 안정 고객 : RFMrfm
  - 이탈 고객 : rfFM
  - 비우량 고객 : rfm
    - 각 수치가 클 경우 **대문자**, 적을 경우 **소문자**
      - **대/소문자**가 모두 존재할경우 중간을 의미

#### 사용자를 1차원으로 구분하기
- **RFM 분석**을 3차원으로 표현하면, 125개의 그룹이 발생하므로
  - 관리하기 매우 어려움
- RFM의 각 랭크 합계를 기반으로 **13개의 그룹**으로 나누어 관리하기
- `R+F+M` 값을 **통합 랭크**로 계산하는 쿼리
  >**통합 랭크**|**R**|**F**|**M**|**사용자 수**
  >:-----:|:-----:|:-----:|:-----:|:-----:
  >15|5|5|5|9
  >14|5|5|4|13
  > |5|4|5|15
  > |4|5|5|15
  >13|5|5|3|16
  >…|…|…|…|…
  >3|1|1|1|842
- 통합 랭크를 계산하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  user_rfm AS (
    ...
  )
  , user_rfm_rank AS (
    ...
  )
  SELECT
    r + f + m AS total_rank
    , r, f, m
    , COUNT(user_id)
  FROM
    user_rfm_rank
  GROUP BY
    r, f, m
  ORDER BY
    total_rank DESC, r DESC, F DESC, m DESC;
  ```
- 종합 랭크별로 사용자 수를 집계하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  user_rfm AS (
    ...
  )
  , user_rfm_rank AS (
    ...
  )
  SELECT
    r + f + m AS total_rank
    , COUNT(user_id)
  FROM
    user_rfm_rank
  GROUP BY
    -- PostgreSQL, Redshift, BigQuery
    -- SELECT 구문에서 정의한 별칭을 GROUP BY 구문에 지정 가능
    total_rank
    -- PostgreSQL, Hive, Redshift, SparkSQL의 경우
    -- SELECT 구문에서 별칭을 지정하기 전의 식을 GROUP BY 구문에 지정할 수 있음
    -- r + f + m
  ORDER BY
    total_rank DESC;
  ```

#### 2차원으로 사용자 인식하기
- **RFM 지표** 2개를 사용하여 사용자 층을 정의하는 방법
- 각 사용자 층에 대해 어떤 **마케팅 대책**을 실시할지,
  - 각 사용자 층을 **상위 사용자 층**으로 어떻게 옮길지를 생각할 수 있음
- **R**, **F**를 사용해 집계하는 방법
  >R/F|**20회 이상**|**10회 이상**|**5회 이상**|**2회 이상**|**1회**
  >:-----:|:-----:|:-----:|:-----:|:-----:|:-----:
  >14일 이내|단골| | |신규|신규
  >28일 이내| |안정|안정| | 
  >60일 이내|단골 이탈 전조|단골 이탈 전조| |신규 이탈 전조|신규 이탈 전조
  >90일 이내| | | |신규 이탈|신규 이탈
  >91일 이상|단골/안정 이탈|단골/안정 이탈|단골/안정 이탈|신규 이탈|신규 이탈
- R과 F를 사용해 2차원 사용자 층의 사용자 수를 집계하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  user_rfm AS (
    ...
  )
  , user_rfm_rank AS (
    ...
  )
  SELECT
    CONCAT('r_', r) AS r_rank
    -- BigQuery의 경우 CONCAT 함수의 매개 변수를 string으로 변환해야 함
    , CONCAT('r_', CAST(r as string)) AS r_Rank
    , CONCAT(CASE WHEN f = 5 THEN 1 END) AS f_5
    , CONCAT(CASE WHEN f = 4 THEN 1 END) AS f_4
    , CONCAT(CASE WHEN f = 3 THEN 1 END) AS f_3
    , CONCAT(CASE WHEN f = 2 THEN 1 END) AS f_2
    , CONCAT(CASE WHEN f = 1 THEN 1 END) AS f_1
  FROM
    user_rfm_rank
  GROUP BY
    r
  ORDER BY
    r_rank DESC;
  ```
## 07. 하나의 테이블에 대한 조작
- 데이터 집약, 데이터 가공
- 하나의 테이블을 대상으로 하는
  - **데이터 집약 방법** 및 가공 방법

### 그룹의 특징 잡기
- review 테이블
  - `product_id`에 대한 `score`가 저장
  - 테이블을 사용해서 `SUM` 함수와 `AVG` 함수 등의 집약 함수

#### 테이블 전체의 특징량 계산
- 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    COUNT(*) AS total_count
    , COUNT(DISTINCT user_id) AS user_count
    , COUNT(DISTINCT product_id) AS product_count
    , ...
  FROM
    review
  ;
  ```
  - `DISTINCT` : 중복을 제거하고 계산
  - `GROUP BY` 없이 계산

#### 집약 함수를 적용한 값과, 이후 값 동시에 다루기
- `SQL:2003` 이후에 정의된 윈도 함수가 지원되는 환경일 때
  - **윈도 함수**를 사용하여 이전/이후 값 조합 가능
- `avg_score` 및 `user_avg_score`의 차이를 구한다.
- 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    user_id
    , product_id
    , score
    , AVG(score) OVER() AS avg_score -- 전체 평균 리뷰 점수
    , AVG(score) OVER(PARTITION BY user_id) AS user_avg_score -- 사용자 평균 리뷰 점수
    , score - AVG(score) OVER(PARTITION BY user_id) AS user_avg_score_diff -- 개별 리뷰 점수 - 사용자 평균 리뷰 점수
  FROM
    review
  ;
  ```
- 윈도 함수를 사용할 때
  - **집약 함수** 뒤에 `OVER` 구문을 붙이고, 윈도 함수를 지정
- `OVER` 구문에 **매개 변수**를 지정하지 않으면
  - 테이블 전체에 집약 함수를 적용한 값 리턴
- `PARTITION BY {col_name}` 사용시,
  - 해당 컬럼 값을 기반으로 그룹화 후 집약 함수 적용


### 그룹 내부의 순서
- 윈도 함수를 사용한 데이터 가공 방법

#### ORDER BY 구문으로 순서 정의
- 윈도 함수로 순서를 다루는 방법
  - 윈도 함수란
    - 테이블 내부에 `window`라고 부르는 **범위** 지정 후
    - 해당 범위 내부에 포함된 값을 **특정 레코드**에서 자유롭게 활용
- 윈도 내부에서 특정 값 참조 시, 값의 위치를 **명확**하게 지정해야 함
  - `OVER` 함수 내부에 `ORDER BY` 구문 사용
  - `ORDER BY score DESC` : 스코어 단위 내림차순
- `ROW_NUMBER` 이 순서에 유일한 순위 번호를 붙이는 함수
- `RANK` 와 `DENSE_RANK` 함수는, 같은 순위의 레코드를 동일한 순위 값으로 조정
- `RANK`의 경우,
  - 같은 순위의 레코드 뒤의 순위 번호를 건너 뜀
  - `1, 1, 3, 4, ...`
- `DENSE_RANK`
  - 순위 번호를 건너뛰지 않음
  - `1, 1, 2, 3, ...`
- `LAG`와 `LEAD` 함수는
  - 현재 행을 기준으로 **앞의 행** 또는 **뒤의 행**을 추출하는 함수
  - 두번째 매개변수에 숫자를 지정하여 `n`번째 값 추출
- 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    product_id
    , score
    -- 점수 순서로 유일한 순위
    , ROW_NUMBER()  OVER(ORDER BY score DESC) AS row
    -- 같은 순위 허용, 순위
    , RANK()        OVER(ORDER BY score DESC) AS rank
    -- 같은 순위 허용, 순위 숫자는 건너뜀
    , DENSE_RANK()  OVER(ORDER BY score DESC) AS dense_rank

    -- 현재 행보다 앞에 있는 행 추출
    , LAG(product_id)       OVER(ORDER BY score DESC) AS lag1
    , LAG(product_id, 2)    OVER(ORDER BY score DESC) AS lag2

    -- 현재 행보다 뒤에 있는 행 추출
    , LEAD(product_id)      OVER(ORDER BY score DESC) AS lead1
    , LEAD(product_id, 2)   OVER(ORDER BY score DESC) AS lead2
  FROM popular_products
  ORDER BY row
  ;
  ```

#### ORDER BY 구문과 집약 함수 조합
- `ORDER BY`와 `SUM/AVG`등의 집야 함수 조합 시,
    - 적용 범위를 유연하게 지정 가능
- `ORDER BY` 구문에 이어지는 `ROWS` 구문은 **윈도 프레임 지정 구문**
- 쿼리
  ```sql
  SELECT
    produt_id
    , score

    , ROW_NUMBER()  OVER(ORDER BY score DESC) AS row

    -- 순위 상위부터의 누계 구하기
    , SUM(score)
        OVER(ORDER BY score DESC
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
        AS cum_score
    
    -- 현재 행 기준 전/후 총 3개행의 평균
    , AVG(score)
        OVER(ORDER BY order DESC
            ROWS BETWEEN 1 PRECEDING AND 1 FOLLOWING)
        AS local_avg
    
    -- 순위가 높은 상품 ID(윈도 내부의 첫 레코드)
    , FIRST_VALUE(product_id)
        OVER(ORDER BY score DESC
            ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)
        AS first_value
    
    -- 순위가 낮은 상품 ID(윈도 내부의 마지막 레코드)
    , LAST_VALUE(product_id)
        OVER(ORDER BY score DESC
            ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)
        AS last_value

  FROM popular_products
  ORDER BY row
  ;
  ```

### 윈도우 프레임 지정
- 프레임 지정
  - 현재 레코드 위치를 기반으로 **상대적인 윈도**를 정의하는 구문
- 프레임 지정 구문
  - `ROWS BETWEEEN {start} AND {end}`
    - `start`와 `end`에는
      - `CURRENT ROW` : 현재 행
      - `n PRECEDING` : n행 앞
      - `n FOLLOWING` : n행 뒤
      - `UNBOUNDED PRECEDING` : 이전 행 전부
      - `UNBOUNDED FOLLOWING` : 이후 행 전부
- 범위 내부 상품 ID를 집약하는 쿼리
  - `PostgreSQL`, `Hive`, `SparkSQL`
  ```sql
  SELECT
    product_id
    , ROW_NUMBER()  OVER(ORDER BY score DESC) AS row

    -- 가장 앞 순위부터, 뒷 순위까지의 범위를 대상으로 상품 ID 집약
    -- PostgreSQL, array_agg
    , array_agg(product_id)
    -- Hive/SparkSQL, collect_list 사용
    , collect_list(product_id)
        OVER(ORDER BY score DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)
    AS whole_agg

    -- 가장 앞 순위부터 현재 순위까지의 범위를 대상으로 상품 ID 집약
    -- PostgreSQL, array_agg
    , array_agg(product_id)
    -- Hive/SparkSQL, collect_list 사용
    , collect_list(product_id)
        OVER(ORDER BY score DESC
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
    AS cum_agg

    -- 순위 하나 앞/뒤까지의 범위를 대상으로 상품 ID 집약
    , array_agg(product_id)
    , collect_list(product_id)
        OVER(ORDER BY score DESC
            ROWS BETWEEEN 1 PRECEDING AND 1 FOLLOWING)
    AS local_agg
  FROM popular_products
  WHERE category='action'
  ORDER BY row
  ;
  ```
- `whole_agg`, `cum_agg`, `local_agg` 모두
  - `{A001, A002, A003, A004}`와 같은 배열 형태를 가짐
- `Redshift`에는 `arragy_agg/collect_list`와 유사한 함수로
  - `listagg`함수 존재
  - 하지만, **프레임 지정**과 동시 사용 불가능
- 윈도 함수에 **프레임 지정**을 하지 않으면
  - `ORDER BY`가 없는 경우 **모든 행**
  - `ORDER BY`가 있는 경우, **첫 행 ~ 현재 행**까지가 default

### PARTITION BY와 ORDER BY 조합
- 윈도 함수를 사용해 카테고리의 순위 계산
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    category
    , product_id
    , score

    -- 카테고리별 점수 순서로 정렬, 유일 순위
    , ROW_NUMBER()
        OVER(PARTITION BY category ORDER BY score DESC)
    AS row

    -- 카테고리별 같은 순위 허가, 순차 순위
    , RANK()
        OVER(PARTITION BY category ORDER BY score DESC)
    AS rank

    -- 카테고리별 같은 순위 허가, 점프 순위
    , DENSE_RANK()
        OVER(PARTITION BY category ORDER BY score DESC)
    AS dense_rank
  FROM popular_products
  ORDER BY category, row
  ;
  ```

### 각 카테고리의 상위 n개 추출
- SQL의 사양으로, 윈도 함수를 `WHERE`구문에 작성할 수는 없음
- `SELECT` 구문에서 **윈도 함수**를 사용한 결과를 **서브쿼리**로 만들고
  - **외부**에서 WHERE 구문 적용
- 카테고리들의 순위 상위 2개까지 상품 추출 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT *
  FROM

  -- 서브 쿼리 내부에서 순위 계산
    ( SELECT
        category
        , product_id
        , score
        , ROW_NUMBER()
            OVER(PARTITION BY category ORDER BY score DESC)
        AS rank
     FROM popular_products
    ) AS popular_products_with_rank
  WHERE rank <=2
  ORDER BY category, rank
  ;
  ```
- 카테고리별 순위 순서에서, 상품 1개의 상품 ID 추출 시
  - 다음과 같이 `FIRST_VALUE` 윈도 함수를 사용하고
  - `SELECT DISTINCT` 구문으로 결과 집약
  - **서브 쿼리**를 사용하지 않아도 됨
- 단, `HIVE`에서는 `DISTINCT`부분이 동작하지 않음
- 카테고리별 순위 최상위 상품 추출 쿼리
  - `PostgreSQL`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT DISTINCT
    category
    , FIRST_VALUE(product_id)
        OVER(PARTITION BY category ORDER BY score DESC
            ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)
    AS product_id
  FROM popular_products
  ;
  ```

### 세로 기반 데이터를 가로 기반으로 변환하기
- 행 단위로 저장된 **세로 기반**을
  - 열 또는 쉼표로 구분된 문자열 등의 **가로 기반**으로 변경
- 예시 테이블
  - 날짜별 KPI데이터
  - `dt date, indecator string, val int`
- 행으로 지정된 지표 값을 열로 변환하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    dt
    , MAX(CASE WHEN indicator = 'impressions' THEN val END) AS impressions
    , MAX(CASE WHEN indicator = 'sessions' THEN val END) AS implressions
    , MAX(CASE WHEN indicator = 'users' THEN val END) AS users
  FROM daily_kpi
  GROUP BY dt
  ORDER BY dt
  ;
  ```

#### 행을 쉼표로 구분한 문자열로 집약하기
- **행**을 **열**로 변환하는 방법은
  - 미리 **열**의 종류와 수를 알아야 함
- 미리 열의 수를 정할 수 없는 경우에는,
  - 데이터를 **쉼표** 등으로 구분한 문자열로 변환
- 예시 테이블
  - `purchase_detail_log`
    - `purchase_id int, product_id string, price int`
- 코드
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    purchase_id

    -- 상품 ID 배열에 집약하고, 쉼표로 구분된 문자열로 변환
    -- PostgreSQL, BigQuery의 경우는 string_agg 사용하기
    , string_agg(product_id, ',') AS product_ids

    -- Redshift, listagg 사용
    , listagg(product_id, ',') AS product_ids

    -- Hive, SparkSQL, collect_list, concat_ws 사용
    , concat_ws(',' collect_list(product_id)) AS product_ids
    , SUM(price) AS amount
  FROM purchase_detail_log
  GROUP BY purchase_id
  ORDER BY purchase_id
  ```

### 가로 기반 데이터를 세로 기반으로 변환하기
- 세로 기반 데이터를 이미 쉼표로 구분된 열 기반 형식등으로 저장되어 있을 때
  - 분석을 위해 다시 변환하는 경우
- 샘플 데이터
  - 4분기 매출(quarterly_sales)
  - `year int, q1 int, q2 int, q3 int, q4 int`
- 컬럼으로 표현된 가로 기반 데이터의 특정 -> **데이터의 수**가 고정됨
- 코드
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    q.year

    -- q1부터 q4까지 레이블 이름 출력하기
    , CASE
      WHEN p.idx = 1 THEN 'q1'
      WHEN p.idx = 2 THEN 'q2'
      WHEN p.idx = 3 THEN 'q3'
      WHEN p.idx = 4 THEN 'q4'
    END AS quarter
    
    -- q1에서 q4까지의 매출 출력
    , CASE
      WHEN p.idx = 1 THEN q.q1
      WHEN p.idx = 2 THEN q.q2
      WHEN p.idx = 3 THEN q.q3
      WHEN p.idx = 4 THEN q.q4
    END AS sales
  FROM
    quarterly_sales AS q
  CROSS JOIN
  -- 행으로 전개하고 싶은 열의 수만큼, 순번 테이블 만들기
    (SELECT 1 AS idx
    UNION ALL SELECT AS 2 AS idx
    UNION ALL SELECT AS 3 AS idx
    UNION ALL SELECT 4 AS idx
    ) AS p
  ```
- 하나의 레코드는 `q1`부터 `q4`까지 4개의 데이터로 구성
- 행으로 전개할 데이터 수가 고정되었을 때,
  - 데이터 수와 같은 수의 **일련번호**를 가진 피벗 테이블 생성 후 `CROSS JOIN`

#### 임의 길이 가진 배열을 행으로 전개
- **고정 길이** 데이터를 **행**으로 전개하는 것은 간단하나,
  - 데이터의 길이가 **확정되지 않은 경우**는 복잡
- **테이블 함수**를 구현하고 있는 미들웨어라면
  - 배열을 쉽게 레코드로 전개 가능
- 테이블 함수를 사용해 배열을 행으로 전개하는 쿼리
  - `PostgreSQL`, `Hive`, `BigQuery`, `SparkSQL`
  ```sql
  -- PostgreSQL의 경우 unnest 함수 사용하기
  SELECT unnest(ARRAY['A001', 'A002' 'A003']) AS product_id;

  -- BigQuery의 경우도 unnest 함수를 사용
  -- 테이블 함수는 FROM에서만 사용 가능
  SELECT * FROM unnest(ARRAY['A001', 'A002', 'A003']) AS product_id;

  -- Hive, SparkSQL의 경우 explode 함수 사용
   SELECT explode(ARRAY('A001', 'A002', 'A003')) AS product_id;
  ```
  - 함수 인자로 **배열**을 받아,
    - **레코드 분할**하여 리턴
- **테이블 함수** 사용시 주의할 점
  - 일반적인 `SELECT` 구문 내부에는
    - **레코드**에 포함된 **스칼라 값**을 리턴하는 함수와 **컬럼 이름**을 지정할수 있으나
  - **테이블 함수**는 **테이블**을 리턴
- `Hive`, `BigQuery` 등에는
  - **스칼라**값과 **테이블**을 동시에 다를수 없음
- **스칼라 값**과 **테이블 함수**의 리턴값을 **동시**에 추출할 떄
  - 테이블 함수를 `FROM` 구문 내부에 작성하고
  - `JOIN`을 활용하여, 원래 테이블과, 테이블 함수의 리턴값을 결합
- 테이블 함수를 사용해, 쉼표로 구분된 문자열 데이터를 행으로 전개하는 쿼리
  - `PostgreSQL`, `Hive`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    purshase_id
    , product_id
  FROM
    purchase_log AS p
  -- string_to_array 함수, 문자열 -> 배열 변환, unnest 함수로 테이블 변환
  CROSS_JOIN unnest(string_to_array(product_ids, ',')) AS product_id

  -- BigQuery의 경우 문자열 분해에 split 함수 사용
  CROSS_JOIN unnest(split(product_ids, ',')) AS product_id

  -- Hive, SparkSQL, LATERAL VIEW explode 사용
  LATERAL VIEW explode(split(product_ids, ',')) e AS product_id
  ```
- `PostgreSQL`의 경우
  - `SELECT` 구문 내부에 **스칼라 값**과 **테이블 함수** 동시 지정 가능
  - 문자열 구분자로 분할하여 테이블화 하는 `regexp_split_to_table` 함수가 구현됨
  - 쿼리
    ```sql
    SELECT
      purcahse_id
      -- 쉼표로 구분된 문자열을 한번에 행으로 전개
      , regexp_split_to_table(product_ids, ',') AS prodcut_id
    FROM purchase_log;
    ```

### Redshift에서 문자열을 행으로 전개하기
- `Redshift`에서는 배열 자료형이 공식적으로 지원되지 않음
  - 몇가지의 **전처리가 더 필요**

#### 피벗 테이블을 사용해 문자열을 행으로 전개
- 쉼표로 구분된 문자열에 포함된
  - 데이터의 최대수 `N`
  - `1`부터 `N`까지의 정수를 하나의 행으로 가지는 **피벗 테이블**
- 쿼리
  - `PostgreSQL`, `Redshift`
  ```sql
  SELECT *
  FROM (
    SELECT 1 AS idx
    UNION ALL SELECT 2 AS idx
    UNION ALL SELECT 3 AS idx
  ) AS pivot
  ;
  ```
- `split_part` 함수 사용
  - 문자열 쉼표 등의 **seperator**로 분할해 `n`번째 요소 추출 함수
  - 쿼리
    - `PostgreSQL`, `Redshift`
    ```sql
    SELECT
      split_part('A001,A002,A003', ',', 1) AS part_1
      , split_part('A001,A002,A003', ',', 2) AS part_2 
      , split_part('A001,A002,A003', ',', 3) AS part_3
    ;
    ```
- 쉼표로 구분된 상품 ID 수 계산
  - `replace` 함수를 사용해서
    - 상품 ID 리스트 문자열에서 쉼표 제거
    - 문자 수를 제거하는 `char_length` 함수로 원래 문자열과의 차이를 계산
  - 문자 수의 차이를 사용해 상품 수를 계산하는 쿼리
    - `PostgreSQL`, `Redshift`
    ```sql
    SELECT
      purchase_id
      , product_ids

      -- 상품 ID의 문자열을 기반으로 쉼표를 제거
      -- 문자 수의 차이를 계산하여 상품수 계산
      , 1 + char_length(product_ids)
       - char_length(replace(product_ids, ',', ''))
      AS product_num
    FROM
      purchase_log
    ;
    ```
- 최종 쿼리 조합
  - 피벗 테이블을 사용해 문자열을 행으로 전개하는 쿼리
    - `PostgreSQL`, `Redshift`
    ```sql
    SELECT
      l.purchase_id
      , l.product_ids
      -- 상품 수만큼 순번 붙이기
      , p.idx
      -- 문자열 쉼표 구분하여 분할, idx 요소 추출
      , split_part(l.product_ids, ',', p.idx) AS product_id
    FROM
      purchase_log AS l
    JOIN
      ( SELECT 1 AS idx
        UNION ALL SELECT 2 AS idx
        UNION ALL SELECT 3 AS idx
      ) AS p
    -- 피벗 테이블의 id가 상품 수 이하인 경우 결합
    ON p.idx <=
      (1 + char_length(l.product_ids)
        - char_length(replace(l.product_ids, ',', '')))
    ;
    ```

### 정리
- SQL 사용시, 미리 데이터를 **레코드**단위로 분할하는 것이 기본
- 1개의 레코드에 데이터를 **집약**시키지 못한 경우도 있음
  - 그 때 **행으로**변환하여 사용
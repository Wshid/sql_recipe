## 10. 다면적인 축을 사용해 데이터 집약하기
- 매출의 시계열 뿐만 아니라
  - 상품의 카테고리, 가격등을 조합해서
  - 데이터의 특징을 추출하기
- 샘플 데이터
  - EC 사이트
  - `dt DATE, order_id INT, user_id STRING, item_id STRING, price INT, category STRING, sub_category STRING`

### 카테고리별 매출과 소계 계산하기
- `UNION ALL`, `ROLLUP`
- 매출 합계 선제시
  - 이후 **PC 사이트**와 **SP 사이트**로 구분
    - 웹, 스마트폰
  - 카테고리별 구분
  - 회원, 비회원 비율등으로 활용
- 카테고리와 소계와 총계를 한번에 출력하는 방식
  - 계층별로 집계한 결과를 같은 컬럼이 되게 변환한뒤 `UNION ALL`구문으로 하나의 테이블로 합치기
  - 대분류, 소분류, 매출이 있을때
    - `all_all_760,000`
    - `mens_all_304,700` 등
- 카테고리별 매출과 소계를 동시에 구하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  sub_category_amount AS (
    -- 소카테고리 매출 집계
    SELECT
      category AS category
      , sub_category AS sub_category
      , SUM(price) AS amount
    FROM
      purchase_detail_log
    GROUP BY
      category, sub_category
  ),
  category_amount AS (
    -- 대카테고리 매출 집계
    SELECT
      category
      , 'all' AS sub_category
      , SUM(price) AS amount
    FROM
      purchase_detail_log
    GROUP BY
      category
  )
  , total_amount AS (
    -- 전체 매출 집계
    SELECT
      'all' AS category
      , 'all' AS sub_category
      , SUM(price) AS amount
    FROM
      purchase_datail_log
  )
            SELECT category, sub_category, amount FROM sub_category_amount
  UNION ALL SELECT category, sub_category, amount FROM category_amount
  UNION ALL SELECT category, sub_category, amount FROM total_amount
  ;
  ```
- 하나의 쿼리로 **카테고리별** 소계와 총계 동시 계싼 가능
- `UNION ALL`을 사용한 방법은 
  - **테이블**을 여러번 로드하고
  - 데이터 결합 비용이 발생하므로
  - **성능이 좋지 않음**
- `SQL99`에서 도입된 `ROLLUP`을 사용한다.
  - `PostgreSQL`, `Hive`, `SparkSQL`에서 성능이 좋은 쿼리 가능
- `ROLLUP` 사용시 유의점
  - 소계 계산시 레코드 집계 키가 `NULL`이 되므로
    - `COALESCE` 함수로 `NULL`을 문자열 `all`로 변환
- `ROLLUP`을 사용해서 카테고리별 매출과 소계를 동시에 구하는 쿼리
  - `PostgreSQL`, `Hive`, `SparkSQL`
  ```sql
  SELECT
    COALESCE(category, 'all') AS category
    , COALESCE(sub_category, 'all') AS sub_category
    , SUM(price) AS amount
  FROM
    purchase_detail_log
  GROUP BY
    ROLLUP(category, sub_category)
    -- HIVE 에서는 다음과 같이 사용
    category, sub_category WITH ROLLUP
  ```

### ABC 분석으로 잘 팔리는 상품 판별하기
- `SUM(~)`, `OVER(ORDER BY)`
- ABC 분석
  - 재고 관리에서 사용하는 분석 방법
  - 매출 중요도에 따라 상품을 나누고, 그에 맞는 전략을 사용
- ABC는 등급을의미
  - 분석 목적에 따라 다르나
    - A : 상위 0 ~ 70%
    - B : 상위 70 ~ 90%
    - C : 상위 90 ~ 100%
- 데이터를 작성하는 방법
  - 매출이 높은 순서대로 데이터 정렬
  - 매출 합계 집계
  - 매출 합계를 기반으로
    - 각 데이터가 차지하는 **비율**을 계산하고 **구성비**를 구함
  - 계산한 카테고리의 **구성비**를 기반으로 **구성비누계**를 구함
- 구성비누계
  - 카테고리별 매출
  - 해당 시점까지의 누계 계산
  - 총 매출로 나눈 값
- 2015년 12월 한 달 동안의 **구매 로그**를 기반으로
  - **매출 구성비누계**와 **ABC** 등급을 계산하는 쿼리
  - 원하는 시간에 있는 **구매 로그** 압축, 상품카테고리마다 매출 계산
  - 전체 매출에 대해 항목별 **매출 구성비**와 **구성비누계**를 계산
  - 이에 따라 등급 구분
- 매출 구성비누계와 ABC등급을 계산하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  monthly_sales AS (
    SELECT
      category1
      -- 항목별 매출 계산
      , SUM(amount) AS amount
    FROM
      purchase_log
    -- 대상 1개월 동안의 로그를 조건으로 걸기
    WHERE
      dt BETWEEN '2015-12-01' AND '2015-12-31'
    GROUP BY
      category1
  )
  , sales_composition_ratio AS (
    SELECT
      category1
      , amount

      -- 구성비 : 100.0 * 항목별 매출 / 전체 매출
      , 100*0 * amount / SUM(amount) OVER() AS composition_ratio

      -- 구성비누계 : 100.0 * 항목별 구계 매출 / 전체 매출
      , 100*0 * SUM(amount) OVER(ORDER BY amount DESC)
        / sum(amount) OVER() AS cumulative_ratio
    FROM
      monthly_sales
  )
  SELECT
    *
    -- 구성비누계 범위에 따라 순위 붙이기
    , CASE
      WHEN cumulative_ratio BETWEEN 0 AND 70 THEN 'A'
      WHEN cumulative_ratio BETWEEN 70 AND 90 THEN 'B'
      WHEN cumulative_ratio BETWEEN 90 AND 100 THEN 'C'
    END AS abc_rank
  FROM
    sales_composition_ratio
  ORDER BY
    amount DESC
  ;
  ```
- 구성비, 구성비누계 등의 **비율** 계산시
  - **정수**를 **실수**로 변환하여, 비율을 구하는 방법 사용
- 분모가 **전체 매출**, **항목별 누계 매출** 처럼
  - `0`보다 큰 숫자가 되기 때문에
  - `0`으로 나누는 것을 따로 확인하지 않음
- **항목별 누계 매출**을 구할 때는
- `SUM` 윈도 함수를 사용

#### 정리
- **구성비**와 **구성비누계** 자체는 데이터 분석에서 많이 사용

### 팬차트로 상품의 매출 증가율 확인
- `FIRST_VALUE` 윈도 함수
- 팬차트
  - 어떤 기준 시점을 `100%`로 정함
  - 이후의 **숫자 변동**을 해주는 그래프
- 도입 취지
  - 상품 또는 카테고리별 매출금액의 추이 판단시
    - 매출 금액이 클 경우, 경향 판단은 쉬울 수 있으나,
    - **작은 변화**는 그래프에서 변화 확인이 어려움
- 변화가 백분율로 표시되므로
  - **작은 변화**에도 쉽게 인지 가능
- 팬 차트 작성시 필요한 데이터
  - 날짜, 카테고리, 매출, Rate
- 구하는 방법
  - **날짜 데이터** 기반으로 **연**과 **월**의 값을 추출
  - **연**과 **월**단위로 **매출**을 계산
  - 구한 매출을 **시계열 순서**로 정렬
  - 월 매출을 기준으로 **비율** 계산
  - 기준이 되는 매출이 **시계열**로 정렬했을 때
    - 가장 **첫 월**의 매출이므로
    - `FIRST_VALUE` 윈도 함수를 사용
- 팬 차트 작성 때 필요한 데이터를 구하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  daily_category_amount AS (
    SELECT
    dt
    , cateogry
    -- PostgreSQL, Hive, Redshift, SparkSQL은 다음과 같잉 구성
    -- BigQuery의 경우 substring을 sdubstr로 수정
    , substring(dt, 1, 4) AS year
    , substring(dt, 6, 2) AS month
    , substring(dt, 9, 2) AS date
    , SUM(price) AS amount
  FROM purchase_detail_log
  GROUP BY dt, category
  )
  , monthly_category_amount AS (
    SELECT
      concat(year, '-', month) AS year_month
      -- Redshift의 경우 concat 함수를 조합해서 사용하거나 || 연산자 사용
      concat(concat(year, '-'), month) AS year_month
      -- year || '-' || month AS year_month
      , category
      , SUM(amount) AS amount
    FROM
      daily_category_amount
    GROUP BY
      year, month, category
  )

  SELECT
    year_month
    , category
    , amount
    -- amount의 첫번째 값을 구한다
    -- CATEGORY로 구분하여, year_month, category로 정렬하며, 이전행 전부를 대상으로 함
    , FIRST_VALUE(amount)
      OVER(PARTITION BY category ORDER BY year_month, category ROWS UNBOUNDED PRECEDING) AS base_amount
    , 100.0 * amount / FIRST_VALUE(amount)
        OVER(PARTITION BY category ORDER BY year_month, category ROWS UNBOUNDED PRECEDING) AS rate
  FROM
    monthly_category_amount
  ORDER BY
    year_month, category
  ;
  ```
- `base_amount` 컬럼에 `FIRST_VALUE` 윈도 함수를 사용해 구한 2014년 1월 매출을 구함
  - 그러한 `base_amoount`에 대한 비율을 `rate` 컬럼에 계산

#### OVER PARTITION BY에 대한 이해
- `OVER(PARTITION BY...)`의 뜻은
  - 구문마다 `GROUP BY`를 하여 컬럼에 값을 담는다는 뜻
- 내용상 GROUP BY 구문이므로, 실제 동일한 내용으로 GROUP BY 되었을 경우,
  - 해당 내용은 같을것으로 의미
- 테이블 분할 함수에 대해
  - https://ggmouse.tistory.com/119
  - 순위 함수의 경우 `OVER(PARTITION BY {col} ORDER BY {col})` 형태
  - 집계의 경우 `OVER(PARTITION BY {col})` 형태를 취한다.

#### 정리
- **팬 차트**를 만들 때
  - **어떤 시점**에서의 매출 금액을 기준으로 할지 중요
- 이에 따라, **성장 경향**과 **쇠퇴 경향**이 나뉘기 때문
- 계절 변동이 제일 **적은 평균적인 달**을 기준으로 선택해야 함
- 근거를 가지고 **기준점**을 채택할 것

### 히스토그램으로 구매 가격대 집계
- `width_bucket` 함수 사용
- 히스토그램은
  - 가로 축에, 단계(데이터의 범위)
  - 세로 축에, 도수(데이터의 개수)
- 데이터가 어떻게 **분산**되어 있는가를 확인할 수 있음
- 데이터의 산에서 가장 **높은 부분**을 **최빈값**이라고 함
- 유의 점
  - 데이터 분포에 따라 **최빈값**은 **평균값**과 비슷하지 않을 수 있음
  - 평균값은 동일하더라도, 최빈값 자체는 당연히 다를 수 있음

#### 히스토 그램을 만드는 방법
- 도수 분포표를 만들어야 함
  - 최댓값, 최소값, 범위(min ~ max)를 구하기
  - 범위를 기반으로 **몇개의 계급**으로 나눌지 결정
    - 각 계급의 **하한**과 **상한**을 구함
  - 각 계급에 들어가는 **데이터 개수(도수)**를 구하고, 표로 정리
- 이를 그래프로 그리면, 히스토그램이 된다.

#### 임의의 계층 수로 히스토그램 만들기
- 샘플 데이터 형태
  - `가격대 하한, 가격대 상한, 도수, 매출액`
- 구매 로그 내부에서
  - 매출금액의 최댓값(`max_amount`)와 최솟값(`min_amount`), 금액 범위(`range_amount`)를 구함
- `WITH` 구문을 이용하여, 이 값을 계산한 테이블을 `stats`로 정의
- **금액 범위**는 이후에 나눗셈을 적용할 것이므로
  - `numeric` 자료형으로 정리
- 추가로 계층 수(`bucket_num`)도 테이블 내부에 직접 정의
- 최댓값, 최솟값, 범위를 구하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  stats AS (
    SELECT
      -- 금액의 최댓값
      MAX(price) AS max_price
      , MIN(price) AS min_price
      , MAX(price) - MIN(price) AS range_price
      , 10 AS bucket_num
    FROM
      purchase_detail_log
  )

  SELECT * FROM stats;
  ```
- 최소 금액에서 최대 금액의 범위를 계층으로 분할하라면
  - `매출금액 - 최소금액`
  - 이후 계층을 판별하기 위한 **정규화 금액(diff)**를 계산
- 이후 첫 계층의 범위(`bucket_range`)는
  - `range_price`를 계급 수(`bucket_num`)으로 나누어 구할 수 있음
- 정규화 금액을 **계급 범위**로 나눈 후
  - `FLOOR` 함수를 이용하여 소수자리를 버리면
  - 해당 매출 금액이 **어떤 계급**에 속하는지 확인 가능
- `SQL` 관련 시스템은 대부분
  - 히스토그램을 작성하는 함수가 표준 제공 됨
  - `PostgreSQL`의 경우 `width_bucket` 함수로 한번에 구할 수 있음
- 데이터의 계층을 구하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  stats AS (
    -- ...
  )
  , purchase_log_with_bucket AS (
    SELECT
      price
      , min_price
      -- 정규화 금액: 대상 금액 - 최소 금액
      , price - min_price AS diff
      -- 계층 범위 : 금액 범위를 계층 수로 분할
      , 1.0 * range_price / bucket_num AS bucket_range

      -- 계층 판정 : FLOOR(정규화 금액 / 계층 범위)
      , FLOOR(
          1.0 * (price - min_price) / (1.0 * range_price / bucket_num)
          -- index가 1부터 시작하므로, 1을 더함
      ) + 1 AS bucket

      -- PostgreSQL, width_bucket 함수 사용
      , width_bucket(price, min_price, max_price, bucket_num) AS bucket
    FROM
      purchase_Detail_log, stats
  )
  SELECT *
  FROM purchase_log_with_bucket
  ORDER BY amount
  ;
  ```
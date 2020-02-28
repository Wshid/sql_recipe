## 09. 시계열 기반으로 데이터 집계하기
- 시계열로 매출금액 집계시
  - **규칙성**을 찾거나
  - 이전 기간과 비교시의 **변화폭** 확인 가능
- 데이터의 집약뿐 아니라
  - 변화를 이해하기 쉽게 표현하는 리포팅 방법
- 매출을 단순 **꺾은선 그래프**로 나타냈을 때는 찾기 힘든 변화를
  - **시각적**으로 확인 가능

### 날짜별 매출 집계하기
- 쿼리
  - `PostgreSQL`, `Hive`, `RedShift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    dt
    , COUNT(*) AS purchase_count
    , SUM(purchase_amount) AS total_amount
    , AVG(purchase_amount) AS avg_amount
  FROM purchase_log
  GROUP BY dt
  ORDER BY dt
  ```

### 이동 평균을 통한 날짜별 추이 보기
- `OVER(ORDER BY ~)` 구문 사용
- 매출이 상승하는 경향 / 하락하는 경향등을 판단하기 위함
- 날짜별 **매출**과 **7일 이동 평균**을 집계하는 쿼리
- 쿼리
  - `PostgreSQL`, `Hive`, `RedShift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    dt
    , SUM(purchase_amount) AS total_amount

    -- 최근 최대 7일 동안의 평균 계산
    , AVG(SUM(purchase_amount))
      OVER(ORDER BY dt ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
    AS seven_day_avg

    -- 최근 7일 동안의 평균을 확실히 계산
    , CASE
      WHEN
        7 = COUNT(*)
        OVER(ORDER BY dt ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
      THEN
        AVG(SUM(purchase_amount))
        OVER(ORDER BY dt ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)
      END
      AS seven_day_avg_strict
    FROM purchase_log
    GROUP BY dt
    ORDER BY dt
  ```
- `seven_day_avg`는 과거 7일분의 데이터를 추출할 수 없는
  - 첫번째 6일간에 대해, 해당 6일만을 가지고 평균을 구함
- 7일의 데이터가 모두 있을 경우에만
  - `seven_day_avg_strict`를 사용할 것

#### 정리
- **이동평균**만으로 리포트를 작성하면, 날짜별 변동 파악이 어려움
- **날짜별 추이**와 **이동 평균**을 함꼐 표현하여 리포트를 만들기

### 당월 매출 누계 계산하기
- `OVER(PARTITION BY ~ ORDER BY ~)`, 누계
- 날짜별 매출뿐 아니라,
  - 해당 월에 어느정도의 매출이 누적되었는지 확인하기
- 날짜별로 매출을 집계하고
  - 해당 월의 **누계**를 구하는 리포트 만들기
- 이때 window함수를 사용한다
- **날짜별 매출**과 **당월 누계 매출**을 집계하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    dt
    -- PostgreSQL, Hive, Redshift, SparkSQL의 경우 substring로 연-월 추출
    , substring(dt, 1, 7) AS year_month

    -- PostgreSQL, Hive, Bigquery, SparkSQL의 경우 substr 함수 사용
    , substr(dt, 1, 7) AS year, month

    , SUM(purchase_amount) AS total_amount
    , SUM(SUM(purchase_amount))
      -- PostgreSQL, Hive, Redshift, SparksQL
      OVER(PARTITION BY substring(dt, 1, 7) ORDER BY dt ROWS UNBOUNDED PRECEDING)

      -- BigQuery의 경우 substring을 substr로 수정
      OVER(PARTITION BY substr(dt, 1, 7) ORDER BY dt ROWS UNBOUNDED PRECEDING)
      AS agg_amount
    FROM purchase_log
  GROUP BY dt
  ORDER BY dt
  ```
- **날짜별 매출**과 **월별 누계 매출**을 동시에 집계하기 위해
  - `substring`함수 사용 하여 `year`과 `month`를 추출
- `GROUP BY dt`로 날짜별 집계한 합계 금액 `SUM(purchase_amount)`에
  - **SUM 윈도 함수** 적용
  - `SUM(SUM(purchase_amount)) OVER (ORDER BY dt)`
    - 날짜 순서대로 **누계 매출** 계산
- 매월 누계를 구하기 위해
  - `OVER`구에 `PARTITION BY substring(dt, 1, 7)`로 월별 파티션 생성
- 가독성 개선하기
  - 반복해서 나오는 `SUM(purchase_amount)`와 `SUBSTR(dt, 1, 7)`을
    - `WITH`구문으로 계산
- 날짜별 매출을 일시 테이블로 만드는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  daily_purchase AS (
    SELECT
      dt
      -- 연, 월, 일을 각각 추출
      -- PostgreSQL, Hive, Redshift, SparkSQL은 다음과 같이 작성
      -- BigQuery의 경우 substring -> substr
      , substring(dt, 1, 4) AS year
      , substring(dt, 6, 2) AS month
      , substring(dt, 9, 2) AS date
      , SUM(purchase_amount) AS purchase_amount
    FROM purchase_log
    GROUP BY dt
  )

  SELECT *
  FROM daily_uprchase
  ORDER BY dt;
  ```
- 날짜를 연,월,일로 분할하고, 날짜별 **합계 금액**을 계산한
  - 일시 테이블을 `daily_purchase`로 지정
- `daily_purchase` 테이블을 사용해
  - 당월 매출 누계를 집계하는 쿼리
- `daily_purchase` 테이블에 대해 **당월 누계 매출**을 집계하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  daily_purchase AS ( ... )

  SELECT
    dt
    , concat(year, '-', month) AS year_month
    
    --Redshift의 경우, concat 함수를 조합해서 사용하거나 || 연산자 사용
    , concat(concat(year, '-'), month) AS year_month
    , year || '-' || month AS year_month

    , purchase_amount
    , SUM(purchase_amount)
        OVER(PARTITION BY year, month ORDER BY dt ROWS UNBOUNDED PRECEDING)
      AS agg_amount
    FROM daily_purchase
    ORDER BY dt;
  ```

#### 정리
- 타 SQL보다, 빅데이터 분석 SQL은
  - 성능이 조금 떨어지더라도, **가독성**과 **재사용성**을 중시하여 작성하는 경우가 많음
- 또한 빅데이터 분석 시,
  - `SQL`에 프로그램 개발때 사용하는
  - **전처리** 사고방식을 도입하는 경우가 많음

### 월별 매출의 작대비 구하기
- 다양한 시점의 리포트
  - 일차, 월차, 연차 매출 등
- 당월을 작년의 해당 월과 비교하기
- JOIN을 사용하지 않고 계산하는 방법
- 월별 매출과 작대비 계산 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  daily_purchase AS (
    ...
  )

  SELECT
    month
    , SUM(CASE year WHEN '2014' THEN purchase_amount END) AS amount_2014
    , SUM(CASE year WHEN '2015' THEN purchase_amount END) AS amount_2015
    , 100.0
      * SUM(CASE year WHEN '2015' THEN purchase_amount END)
      / SUM(CASE year WHEN '2014' THEN purchase_amount END)
    AS rate
  FROM
    daily_purchase
  GROUP BY month
  ORDER BY month
  ;
  ```

### Z 차트로 업적의 추이 확인
- `SUM(CASE ~ END) OVER(ORDER BY ~)`
- Z 차트
  - 3개의 지표
    - 월차매출, 매출누계, 이동년계
  - 계절 변동의 영향을 배제하고 트렌드를 분석하는 방법
- 세 지표의 의미
  - **월차매출**
    - 매출 합계를 **월별**로 집계
  - **매출누계**
    - 해당 월의 매출에 이전월까지의 매출 **누계**를 합한 값
      - 2016년 1월 : 2016.01
      - 2016년 2월 : 2016.01 + 2016.02
      - 2016년 6월 : 2016.01 ~ 2016.06
  - **이동년계**
    - 해당 월의 매출 + 11개월의 매출
      - 2016년 1월 기준일시, `2015.02 ~ 2016.01`의 매출

#### Z 차트 분석 정리
- **매출누계**
  - 월차 매출이 **일정**할 경우, **매출누계**는 직선이 됨
    - `y=x`의 그래프
  - 기울기에 따라 최근 매출의 상승/하락 확인 가능
- **이동년계**
  - 작년과 올해의 매출이 일정하다면, 이동년계가 직선이 됨
    - `y=0`의 그래프
  - 과거 1년동안의 매출이 어떤 추이를 가지는지 확인 가능

#### Z 차트 작성 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  daily_purchase AS (
    ...
  )
  , monthly_amount AS (
    -- 월별 매출 집계
    SELECT
      year
      , month
      , SUM(purchase_amount) AS amount
    FROM daily_purhase
    GROUP BY year, month
  )
  , calc_index AS (
    SELECT
      year
      , month
      , amount
      -- 2015년의 누계 매출 집계
      , SUM(CASE WHEN year='2015' THEN amount END)
        OVER(ORDER BY year, month ROWS UNBOUNDED PRECEDING)
        AS agg_amount

      -- 당월부터 11개월 이전까지의 매출 합계(이동년계) 집계
      , SUM(amount)
        OVER(ORDER BY year, month ROWS BETWEEN 11 PRECEDING AND CURRENT ROW)
        AS year_avg_amount
      FROM
        monthly_purchase
      ORDER BY
        year, month
  )

  -- 2015년의 데이터 압축
  SELECT
    concat('year', '-', month) AS year_month
    -- Redshift의 경우 concat 함수를 조합하거나, || 연산자 사용
    concat(concat(year, '-'), month) AS year_month
    year || '-' || month AS year_month

    , amount
    , agg_amount
    , year_avg_amount
  FROM cal_index
  WHERE year = '2015'
  ORDER BY year_month
  ;
  ```

#### 정리
- **계절 트렌드 영향**을 제외하고
  - 매출의 성장 또는 쇠퇴를 다양한 각도에서 살펴볼때, 차트를 활용하는 것이 좋음
- 매일 확인해야할 그래프는 아니나, 주기적으로 살펴보면 좋음

### 매출 파악시 중요 포인트
- 매출이라는 결과의
  - 원인이라 할 수 있는 **구매 횟수**, **구매 단가**등의
  - 주변 데이터를 고려해야 왜인지 확인 가능
- 매출 리포트를 만들때는, **주변 데이터**를 함께 포함해야 함
  - 판매 횟수, 평균 구매액 등
- 매출과 관련된 지표를 집계하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  daily_purchase AS (
    ...
  )
  , monthly_purchase AS (
    SELECT
      year
      , month
      , SUM(orders) AS orders
      , AVG(purchase_amount) AS avg_amount
      , SUM(purchase_amount) AS monthly
    FROM daily_purchase
    GROUP BY year, month
  )

  SELECT
    concat(year, '-', month) AS year_month
    -- Redshift의 경우 concat 함수를 조합하거나, || 연산자 사용
    concat(concat(year, '-'), month) AS year_month
    year || '-' || month AS year_month

    , orders
    , avg_amount
    , monthly
    , SUM(monthly)
      OVER(PARTITION BY year ORDER BY month ROWS UNBOUNDED PRECEDING) AS agg_amount

    -- 12개월 전의 매출 구하기
    , LAG(monthly, 12)
      OVER(ORDER BY year, month)
    -- sparkSQL의 경우 다음과 같이 사용
      OVER(ORDER BY year, month ROWS BETWEEN 12 PRECEDING AND 12 PRECEDING)
      AS last_year
    , 100.0
      * monthly
      / LAG(monthly, 12)
        OVER(ORDER BY year, month)
      -- sparkSQL의 경우 다음과 같이 사용
      OVER(ORDER BY year, month ROWS BETWEEN 12 PRECEDING AND 12 PRECEDING)
      AS rate
  FROM monthly_purchase
  ORDER BY year, month
  ;

  -- PRECEDING : 이전
  -- LAG : 현재 행 기준 n번 전
  ```
- `purchase_log` 테이블 기반으로
  - **월 단위 매출**을 정리한 `monthly_purchase` 테이블 생성
- **작년**의 동일한 **달**을 구할 때
  - `CASE` 내부에서 대상 년도를 검색했지만
  - 위에서는 `LAG`함수를 사용하여 추출
    - 이 방법이 더 일반적임
- `LAG`로 12개월전 값을 추출할 수 없는 경우
  - `year`과 `rate(작대비)`는 `NULL`
- 윈도 함수 없이 작성하려면
  - 각 지표마다 `SELECT`구문을 만들고
  - 최종적으로 결합해야 함
- `SELECT`를 여러번 사용할 경우, 성능이 좋지 않음
  - 데이터를 여러번 읽어들이기 때문
- 중간 테이블 `monthly_purchase` 테이블을
  - `WITH`구문으로 만들 필요가 없이,
  - 하나의 `SELECT`구문으로도 최종 결과 계산 가능
  - 하짐란, 과정에 해당하는 컬럼도 **임시 테이블**을 만들경우 **가독성 향상 효과**
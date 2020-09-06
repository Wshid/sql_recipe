# 13. 시계열에 따른 사용자의 개별적인 행동 분석
- 사용자가 취한 액션의 **시간**에 주목하여
  - 여러 액션이, 어느정도의 **시간차**를 두고 발생하는지
  - 얼마나 시간이 지나면 **다음 액션**을 수행하는지 집계
- 사용자의 **최종 성과**(결과)에 도달할 때까지
  - 어느정도의 **검토 기간**(과정)이 필요한지 알게 되면,
    - 캠페인 시행시기를 결정할 때 도움이 될 수 있으며,
  - 검토 기간이 예상보다 길다면, 그 기간을 줄일 대책을 세울 자료가 될 수 있음
- 사용자의 **액션**을 **시간의 경과**라는 요소와 함께 고려하여, 시각화하면
  - 사용자의 **성장 과정** 또는 **행동 패턴**을 명확하게 할 수 있음

## 1. 사용자의 액션 간격 집계하기
- SQL : 날짜 함수, LAG 함수
- 분석 : 리드 타입
- 사용자의 특정 **액션**을 기반으로, **다음 액션**까지의 **기간**을 집계하면
  - 새로운 문제를 찾고 대책을 검토할 수 있음
- 예시
  - `자료 청구 -> 방문 예약 -> 견적 -> 계약`의흐름이 있다고 할 때,
  - 실제 `계약`까지의 기간(`리드 타임`)이 예상보다 길다는 것을 발견하면
    - 각각의 흐름에서 문제를 찾고 이를 해결할 수 있음
  - `리드 타임`이 짧아지면, 계약이 더 많이 일어나므로, **매출 향상**을 기대할 수 있음
- 데이터 패턴에 따라 `리드 타임`을 계산하는 `SQL`

### 같은 레코드에 있는 두 개의 날짜로 계산할 경우
- `숙박 시설`, `음식점` 등의 **예약 사이트**에는
  - 예약 정보를 저장하는 레코드에 `신청일`과 `숙박일`, `방문일`을 한꺼번에 저장
- 따라서 날짜끼리 빼거나 `datediff` 함수를 사용하여, 각 날짜 차이를 계산하면 됨

#### CODE.13.1. 신청일과 숙박일의 리드 타임을 계산하는 쿼리
- `PostgreSQL`, `Hive`, `RedShift`, `BigQuery`, `SparkSQL`
```sql
WITH
reservations(reservation_id, register_date, visit_date, days) As (
  -- PostgreSQL의 경우 VALUES 구문으로 일시 테이블 생성 가능
  -- Hive, Redshift, BigQuery, SparkSQL의 경우 UNION ALL로 대체 가능
  VALUES
      (1, date '2016-09-01', date '2016-10-01', 3)
    , (2, date '2016-09-20', date '2016-10-01', 2)
    , (3, date '2016-09-30', date '2016-11-20', 2)
    , (4, date '2016-10-01', date '2017-01-03', 2)
    , (5, date '2016-11-01', date '2016-12-20', 3)
)
SELECT
  reservation_id
  , register_date
  , visit_date
  -- PostgreSQL, Redshift, 날짜끼리 뺄셈 가능
  , visit_date::date - register_date::date AS lead_time
  -- BigQuery, date_diff
  , date_diff(date(timestamp(visit_date)), date(timestamp(register_date)), day) AS lead_time
  -- Hive, SparkSQL, datediff 함수 사용
  , datediff(to_date(visit_date), to_date(register_date)) AS lead_time
FROM
  reservations
;
```

### 여러 테이블에 있는 여러 개의 날짜로 계산할 경우
- `자료 청구`, `예측`, `계약`처럼 여러 단계가 존재하는 경우
  - 각각의 데이터가 다른 테이블에 저장되는 경우가 많음
- 이럴 때는, 여러 개의 테이블에 있는 날짜를 참조할 수 있도록
  - 테이블을 `JOIN`하고, 이전과 같은 방법으로 날짜의 차이 계산

#### CODE.13.2. 각 단계에서의 리드 타임과 토탈 리드 타임을 계산하는 쿼리
- `PostgreSQL`, `Hive`, `RedShift`, `BigQuery`, `SparkSQL`
```sql
WITH
requests(user_id, product_id, request_date) AS (
  -- PostgreSQL의 경우 VALUES로 일시 테이블 생성 가능
  -- Hive, Redshift, BigQuery, SparkSQL의 경우 UNION ALL로 대체
  VALUES
      ('U001', '1', date '2016-09-01')
    , ('U001', '2', date '2016-09-20')
    , ('U002', '3', date '2016-09-30')
    , ('U003', '4', date '2016-10-01')
    , ('U004', '5', date '2016-01-01')
)
, estimates(user_id, product_id, estimate_date) AS (
  VALUES
      ('U001', '2', date '2016-09-21')
    , ('U002', '3', date '2016-10-15')
    , ('U003', '4', date '2016-09-30')
    , ('U004', '5', date '2016-12-01')
)
, orders(user_id, product_id, order_date) AS (
  VALUES
      ('U001', '2', date '2016-10-01')
    , ('U004', '5', date '2016-12-05')
)
SELECT
  r.user_id
  , r.product_id
  -- PostgreSQL, Redshift, 날짜끼리 뺄셈 가능
  , e.estimate_date::date - r.request_date::date AS estimate_lead_time
  , o.order_date::date    - e.estimate_date::date AS order_lead_time
  , o.order_date::date    - r.request_date::date AS total_lead_time
  -- BigQuery, date_diff
  , date_diff(date(timestamp(e.estimate_date)), date(timestamp(r.request_date)), day) AS estimate_lead_time
  , date_diff(date(timestamp(o.order_date)), date(timestamp(e.estimate_date)), day) AS order_lead_time
  , date_diff(date(timestamp(o.order_date)), date(timestamp(r.request_date)), day) AS total_lead_time
  -- Hive, SparkSQL, datediff
  , datediff(to_date(e.estimate_date), to_date(r.request_date)) AS estimate_lead_time
  , datediff(to_date(o.order_date), to_date(e.estimate_date)) AS order_lead_time
  , datediff(to_date(o.order_Date), to_date(r.request_date)) AS total_lead_time
FROM
  requests AS r
  LEFT OUTER JOIN
    estimates AS e
      ON r.user_id = e.user_id 
      AND r.product_id = e.product_id
  LEFT OUTER JOIN
    orders AS o
      ON r.user_id = o.user_id
      AND r.product_id = o.product_id
;
```

### 같은 테이블의 다른 레코드에 있는 날짜로 계산할 경우
- `EC 사이트`에서 어떤 구매일로부터 다음 구매일까지의 간격을 알고 싶은 경우
  - 데이터가 같은 테이블에 존재할 것
- 이럴 때는 `LAG` 함수를 사용하여 날짜의 차이 계산

#### CODE.13.3. 이전 구매일로부터의 일수를 계산하는 쿼리
- `PostgreSQL`, `Hive`, `RedShift`, `BigQuery`, `SparkSQL`
```sql
WITH
purchase_log(user_id, product_id, purchase_date) AS (
  -- PostgreSQL, VALUES로 일시 테이블 생성
  -- Hive, Redshift, BigQuery, SparkSQL의 경우 UNION ALL로 대체
  VALUES
      ('U001', '1', '2016-09-01')
    , ('U001', '2', '2016-09-20')
    , ('U002', '3', '2016-09-30')
    , ('U001', '4', '2016-10-01')
    , ('U002', '5', '2016-11-01')
)
SELECT
  user_id
  , purchase_date
  -- PostgreSQL, Redshift, 날짜끼리 뺄셈
  , purchase_date::date
    - LAG(purchase_date::date)
      OVER(
        PARTITION BY user_id
        ORDER BY purchase_date
      ) AS lead_time
  -- BigQuery, date_diff 함수
  , date_diff(to_date(purchase_date), 
      LAG(to_date(purchase_date))
        OVER(PARTITION BY user_id ORDER BY purchase_date)) AS lead_time
  -- SparkSQL의 경우 datediff 함수에 LAG 함수의 프레임 지정이 필요
  , datediff(to_date(purchase_date), 
      LAG(to_date(purchase_date))
        OVER(PARTITION BY user_id ORDER BY purchase_date
          ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING)) AS lead_time
FROM
  purchase_log
;
```

### 원포인트
- **리드 타임**을 집계했다면, 사용자의 **데모그래픽**정보를 사용해 비교하기
- `EC사이트`라면, 수도권보다 **지방**이
  - 이전 구매일로부터의 **리드 타임**이 짧거나
  - **연령별**로 구분이 되는 등 다양한 경향이 나타남

## 2. 카트 추가 후에 구매했는지 파악하기
- SQL : `CASE`, `SUM`, `AVG`
- 분석 : 카트 탈락률

### 카트 탈락
- 카트에 얺은 상품을 구매하지 않고, 이탈한 상황
  - 상품 구매까지 절차에 어떤 문제가 있는 경우
  - 예상하지 못한 비용(배송비, 수수료 등)으로 당황
  - 북마크 기능 대신 카트를 사용하는 사람들
- 카트에 상품을 추가했다고 해서, **반드시 구매**로 이어지지 않음
  - **시간에 따라 구매로 이어지는 비율**이 어느정도 인지, 리포트로 만들어 보는 것이 좋음
- 이번 절에서는
  - 카트에 추가된지 `48시간` 이내에 구매되지 않은 상품을 **카트 탈락**이라고 부르고
  - 카트에 추가된 상품 수를 분모로 넣어 구한 비율을 **카트 탈락률**이라 정의

### CODE.13.4. 상품들이 카트에 추가된 시각과 구매된 시각을 산출하는 쿼리
- `PostgreSQL`, `Hive`, `BigQuery`, `SparkSQL`
```sql
WITH
row_action_log AS (
  SELECT
    dt
    , user_id
    , action
    -- 쉼표로 구분된 product_id 리스트 전개하기
    -- PostgreSQL의 경우 regexp_split_to_table 함수 사용 가능
    , regexp_split_to_table(products, ',') AS product_id
    -- Hive, BigQuery, SparkSQL의 경우, FROM 구문으로 전개하고 product_id 추출
    , product_id
    , stamp
  FROM
    action_log
    -- BigQuery
    CROSS JOIN unnest(split(products, ',')) As product_id
    -- Hive, SparkSQL
    LATERAL VIEW explode(split(products, ',')) e AS product_id

, action_time_stats AS (
  -- 사용자와 상품 조합의 카트 추가 시간과 구매 시간 추출
  SELECT
    user_id
    , product_id
    , MIN(CASE action WHEN 'add_cart' THEN dt END) AS dt
    , MIN(CASE action WHEN 'add_cart' THEN stamp END) AS add_cart_time
    , MIN(CASE action WHEN 'purchase' THEN stamp END) AS purchase_time
    -- PostgreSQL, timestamp 자료형으로 변환하여 간격을 구한 뒤, EXTRACT(epoc ~)로 초단위 변환
    , EXTRACT(epoch from
      MIN(CASE action WHEN 'purchase' THEN stamp::timestamp END)
      - MIN(CASE action WHEN 'add_cart' THEN stamp::timestamp END))
    -- BigQuery, unix_seconds 함수로 초 단위 UNIX 시간 추출 후 차이 구하기
    , MIN(CASE action WHEN 'purchase' THEN unix_seconds(timestamp(stamp)) END)
      - MIN(CASE action WHEN 'add_cart' THEN unix_seconds(timestamp(stamp)) END)
    -- Hive, Spark, unix_timestamp 함수로 초 단위 UNIX 시간 추출 후 차이 구하기
    , MIN(CASE action WHEN 'purchase' THEN unix_timestamp(stamp) END)
      - MIN(CASE action WHEN 'add_cart' THEN unix_timestamp(stamp) END)
    AS lead_time
  FROM
    row_action_log
  GROUP BY
    user_id, product_id
)
SELECT
  user_id
  , product_id
  , add_cart_time
  , purchase_time
  , lead_time
FROM
  action_time_stats
ORDER BY
  user_id, product_id
;
```

### CODE.13.5. 카트 추가 후 n시간 이내에 구매된 상품 수와 구매율을 집계하는 쿼리
- `PostgreSQL`, `Hive`, `BigQuery`, `SparkSQL`
```sql
WITH
row_action_log AS (
  ...
)
, action_time_stats AS (
  ...
)
, purchase_lead_time_flag AS (
  SELECT
    user_id
    , product_id
    , dt
    , CASE WHEN lead_time <= 1 * 60 * 60 THEN 1 ELSE 0 END AS purchase_1_hour
    , CASE WHEN lead_time <= 6 * 60 * 60 THEN 1 ELSE 0 END AS purchase_6_hours
    , CASE WHEN lead_time <= 24 * 60 * 60 THEN 1 ELSE 0 END AS purchase_24_hours
    , CASE WHEN lead_time <= 48 * 60 * 60 THEN 1 ELSE 0 END AS purchase_48_hours
    , CASE
        WHEN lead_time IS NULL OR NOT (lead_time <= 48 * 60 * 60) THEN 1
        ELSE 0
      END AS not_purchase
  FROM action_time_stats
)
SELECT
  dt
  , COUNT(*) AS add_cart
  , SUM(purchase_1_hour) AS purhcase_1_hour
  , AVG(purchase_1_hour) AS purhcase_1_hour_rate
  , SUM(purchase_6_hours) AS purchase_6_hours
  , AVG(purchase_6_hours) AS purhcase_6_hours_rate
  , SUM(purchase_24_hours) AS purchase_24_hours
  , AVG(purchase_24_hours) AS purchase_24_hours_rate
  , SUM(purchase_48_hours) AS purchase_48_hours
  , AVG(purchase_48_hours) AS purchase_48_hours)rate
FROM
  purchase_lead_time_flag
GROUP BY
  dt
;
```

### 원포인트
- 카트 탈락 상품이 있는 사용자에게 **메일 매거진** 등의 **푸시**로
  - 쿠폰 또는 할인 정보를 보내주어 구매를 유도하는 방법 등을 고려
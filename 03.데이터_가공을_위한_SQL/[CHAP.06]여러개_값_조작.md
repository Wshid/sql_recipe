## 06. 여러개 값 조작
- 여러개의 값을 집약하여 하나의 값으로 만들거나,
    - 다양한 값을 비교하는 경우가 많다.
- 값을 조작하는 목적과, 레코드에 포함된 다른값을 조합하여 새로운 값을 집계

### 새로운 지표 정의하기
- 페이지뷰
  - 어떤 페이지가 출력된 횟수
- 방문자 수
  - 어떤 페이를 출력한 사용자 수
- 이 둘을 기반으로, `페이지 수 / 방문자 수`를 하면
  - 사용자 한 명이, 페이지를 몇 번이나 방문했는지를 계산할 수 있음
- 방문한 사용자 중에서
  - 특정한 행동(**클릭** 또는 **구매**)를 한 사용자의 비율을 구하여
    - **CTR**(Click Through Rate, 클릭 비율)
    - **CVR**(Conversion Rate, 컨버전 비율)
  - 지표를 정의하고 활용할 수 있음
- 단순하게 숫자료 비교하면, **숫자가 큰 데이터**만 주목함
  - **개인별** 또는 **비율** 등의 지표 사용시,
    - 다양한 관점에서 분석 가능

### 문자열 연결하기
- 주소 연결하기
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    user_id
    
    -- PostgreSQL, Hive, Redshift, BigQuery, SparkSQL 모두 CONCAT 함수 사용 가능
    -- 다만 redshift의 경우는 매개변수를 2개밖에 못받는다
    , CONCAT(pref_name, city_name) AS pref_city
    
    -- PostgreSQL, Redshift의 경우 || 연산자 사용 가능
    , pref_name || city_name AS pref_City
  FROM
    mst_user_location
  ;
  ```
- 대부분의 미들웨어에서, `CONCAT` 사용 가능
- 단 `Redshift`의 경우 `CONCAT`은 매개변수 **2개**의 문자열만 전달 가능하므로,
  - `||`연산자를 사용하면 된다.

### 여러개의 값 비교하기
- SQL
  - **CASE 식**, **SIGN 함수**, **greatest 함수**, **least 함수**, **사칙 연산자**
- 하나의 레코드에 포함된 여러 개의 값을 비교하기
- 분기별 매출 테이블(`q1 ~ q4`), 매출 금액이 지정되지 않을경우 `NULL`을 포함

#### 분기별 매출 증감 판정하기
- 분기별 매출 증가/감소 판단
- 컬럼의 크고 작음 비교시, `CASE`식을 사용하여 조건 사용
- `q1 > q2`의 매출일 일경우, `+`, 같을 경우 ` `, 작은경우 `-` 기입
- 값의 차이를 구하려면 `-`와 같이 컬럼간의 빼기 연산
- `SIGN`함수와의 조합으로, 간단하게 **식의 증감** 판단 가능
- `SIGN` 함수
  - `x>0`, `return 1`
  - `x=0`, `0`
  - `x<0`, `return -1`
- 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    year
    ,q1
    ,q2
    
    -- q1과 q2의 매출변화 평가
    , CASE
      WHEN q1 < q2 THEN '+'
      WHEN q1 = q2 THEN ' '
      ELSE '-'
    END AS judge_q1_q2
    
    -- q1, q2의 매출액 차이 계싼
    , q2 - q1 AS diff_q2_q1
    
    -- q1과 q2의 매출 변화를 1, 0, -1로 표현
    , SIGN(q2 - q1) AS sign_q2_q1
  FROM
    quarterly_Sales
  ORDER BY
    year
  ;
  ```
  
### 연간 최대/최소 4분기 매출 찾기
- 대소 비교시, 컬럼의 수가 많아지면 코드가 복잡해짐
- 최대, 최소값 찾을 때, `greatest/least` 함수를 사용
- 둘 다 SQL 표준에는 포함되지 않으나, 대부분의 SQL 쿼리 엔진에서 구현 됨
- 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    year
    
    -- q1 ~ q4의 최대 매출 구하기
    , greatest(q1, q2, q3, q4) AS greatest_sales
    
    -- q1 ~ q4의 최소 매출 구하기
    , least(q1, q2, q3, q4) AS least_sales
  FROM
    quarterly_sales
  ORDER BY
    year
  ;
  ```

### 연간 평균 4분기 매출 계산하기
- `greatest` 함수 또는 `least`는 기본 제공 함수
- 이를 활용하여 여러 개의 컬럼에 처리하는 경우
  - `q1`에서 `q4`의 매출 평균을 계산하기
- 평균값 구하기
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    year
    , (q1 + q2 + q3 + q4) / 4 AS average
  FROM
    quarterly_sales
  ORDER BY
    year
  ;
  ```
- `NULL` 값을 사칙연산 하려면
  - `COALESCE` 함수를 사용해 적합한 값을 변환
- `NULL`을 포함한 값을 처리한 평균값 구하기
  ```sql
  SELECT
    year
    , (COALESCE(q1, 0) + COALESCE(q2, 0) + COALESCE(q3, 0) + COALESCE(q4, 0)) / 4 AS average
  FROM
    quarterly_sales
  ORDER BY
    year
  ;
  ```
- `NULL`이 아닌 컬럼만을 사용하여 평균값 구하기
  ```sql
  SELECT
    year
  , (COALESCE(q1, 0) + COALESCE(q2, 0) + COALESCE(q3, 0) + COALESCE(q4, 0))
  / (SIGN(COALESCE(q1, 0)) + SIGN(COALESCE(q2, 0)) + SIGN(COALESCE(q3, 0)) + SIGN(COALESCE(q4, 0))) AS average
  FROM
    quarterly_sales
  ORDER BY
    year
  ;
  ```

#### 정리
- **하나의 레코드** 내부 값끼리 연산시
  - 여러 개의 컬럼에 있는 비교/계산 처리가 쉬움
- 하지만, **여러 레코드**에 걸쳐 있는 값 비교 시,
  - **aggregation** 관련 함수로 처리해야 함
  

### 2개의 값 비율 계산하기
- SQL
  - **나눗셈**, **cast 구문**, **CASE 식**, **NULLIF 함수**
- 광고 통계 정보(advertising_stats) 테이블
  - 매일의 **광고 노출 수**와 **클릭 수** 집계

#### 정수 자료형의 데이터 나누기
- 각 광고의 **CTR**(Click Through Rate) 계산
  - `클릭 / 노출 수`
- 나눗셈을 할 때 `PostgreSQL`의 경우
  - `advertising_stats` 테이블의 `clicks`와 `impression`이 **정수형**이므로,
    - 계산 결과도 정수형이 되어 `0`이 출력 됨
  - `CAST` 함수를 사용해, `click`을 `double precision` 자료형으로 변환해야
    - 결과도 `double precision` 자료형으로 리턴 됨
- 결과를 **퍼센트**로 나타낼 때,
  - `ctr * 100`
- `ctr_as_percent` 처럼 `click * 100.0`을 하면,
  - **자료형 변환**이 자동으로 이루어짐
- 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    dt
    , ad_id
  
    -- Hive, Redshift, Bigquery, SparkSQL
    -- 정수를 나눌때, 자동으로 실수형 변환
    , clicks / impressions AS ctr
    
    -- PostgreSQL, 정수 나눌경우, 소수점이 잘리므로, 명시적으로 자료형 변환
    , CAST(clicks AS double precision) / impressions AS ctr
    
    -- 실수를 상수로 앞에 두고 계산하면, 암묵적으로 자료형 변환
    , 100.0 * clicks / impressions AS ctr_as_percent
  FROM
    advertising_stats
  WHERE
    dt='2017-04-01'
  ORDER BY
    dt, ad_id
  ;
  ```

### 0으로 나누는 것 피하기
- `0`으로 나누게 되면 **오류** 발생
- 회피 방법
  - `CASE`식을 사용하여 `impressions = 0`인지 판단
    - `ctr_as_percent_by_case`
      - `impression > 0`일 경우만 계산
      - 해당하지 않을경우 `NULL` 리턴
- `NULL` 전파
  - `0`으로 나누는 것을 회피할 수 있음
  - `NULL`을 포함한 **데이터의 연산 결과**가 모두 `NULL`이 되는 **SQL 성질**
  - `percent_by_null`
    - `NULLIF(impressions, 0)`
      - `impressions = 0`, `return NULL`
      - `NULL` 전반으로 CTR 값도 `NULL`이 되어, `CASE`식을 사용한 방법과 동일한 결과
- 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    dt
    , ad_id
    
    -- CASE 식으로 분모가 0일 경우를 분기, 0으로 나누지 않도록 함
    , CASE
      WHEN impressions > 0 THEN 100.0 * clicks / impressions
    END AS ctr_as_percent_by_case
    
    -- 분모가 0이라면 NULL로 변환하여, 0으로 나누지 않도록 함
    -- PostgreSQL, Redshift, BigQuery, SparkSQL의 경우 NULLIF 함수 사용
    , 100.0 * clicks / NULLIF(impressions, 0) AS ctr_as_percent_by_null
    
    -- Hive의 경우 NULLIF 대신 CASE식 사용하기
    , 100*0 * clicks / 
    CASE WHEN impressions = 0 THEN NULL ELSE impressions END
  FROM
    advertising_stats
  ORDER_BY
    dt, ad_id
  ;
  ```

#### 정리
- 정수로 나누거나, `0`으로 나누는 등의 실수 조심하기

### 두 값의 거리 계산하기
- 거리
  - 두 값을 입력하고, 값이 얼마나 떨어져 있는지 나타내기
  - 예시
    - 시험을 보았을 때, 평균과 얼마나 떨어져 있는가
    - 작년 매출과 올해 매출의 차이
- 어떤 사용자가 있을 때,
  - 해당 사용자와 **구매 성향이 비슷한 사용자**를 뽑는 등의 응용상황에서 사용

#### 숫자 데이터의 절대값, RMS(제곱 평균 제곱근) 계산
- 일차원 위치 정보 테이블(`location_1d`)
  - `x1 INT, x2 INT`
- `ABS`(abstract) 함수
  - **절대값** 계산
- 제곱 평균 제곱근(RMS)
  - 두 값의 차이를 제곱
    - 이 값에 **제곱근** 적용
- 제곱 => `POWER` 함수
- 제곱근 => `SQRT` 함수
- 값이 **1차원**일 경우
  - `절대값 = 제곱 평균 제곱근`
- 1차원 데이터의 제곱 평균 제곱근 계산
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    ABS(x1 -x2) AS abs
    , sqrt(power(x1 - x2, 2)) AS rms
  FROM location_1d
  ```

#### 유클리드 거리 계산
- `xy 평면`위에 있는 두 점 (x1, y1), (x2, y2) 사이의 **유클리드 거리** 계산
- **물리적인 공간**에서 **거리**를 구할 때 사용하는 일반적인 방법
- **제곱 평균 제곱근**을 사용하여 구하면 된다.
- `PostgreSQL`에는 `Point` 자료형 존재
- 이차원 테이블에 대해 **제곱 평균 제곱근(유클리드 거리)**을 구하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    sqrt(power(x1 - x2, 2) + power(y1 - y2)) AS dist
    
    -- PostgreSQL, point 자료형과 거리 연산자 (<->) 사용
    , point(x1, y1) <-> point(x2, y2) AS dist
  FROM
    location_2d
  ;
  ```
- 이는 **유사도 계산** 및 **추천 구현**의 기초가 되는 개념

#### 날짜, 시간 계산하기
- 두 날짜 데이터의 차이 / 시간 데이터를 기준으로 n 시간 뒤 시간 구하기
- 사용자 마스터(smt_users_with_dates) 테이블
  - `user_id STRING, register_stamp TIMESTAMP, birth_date DATE`
- 미래 또는 과거 날짜/시간 계산 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    user_id

    -- PostgreSQL, interval 자료형의 데이터에 사칙 연산 적용
    , register_stamp::timestamp AS register_stamp
    , register_stamp::timestamp + '1 hour'::interval AS after_1_hour
    , register_stamp::timestamp - '30 minutes'::interval AS berfore_30_minutes

    , register_stamp::date AS register_date
    , (register_stamp::date + '1 day'::interval)::date AS after_1_day
    , (register_stamp::date - '1 month'::interval)::date AS before_1_month

    -- Redshift, dateadd 함수 사용
    , register_stamp::timestamp AS register_stamp
    , dateadd(hour, 1 ,register_stamp::timestamp) AS after_1_hour
    , dateadd(monute, -30, register_stamp::timestamp) AS before_30_minutes

    , register_stamp::date register_date
    , dateadd(day, 1, register_stamp::date) AS after_1_day
    , dateadd(month, -1, register_stamp:date) AS before_1_month

    -- BigQuery, timestamp_add/sub, date_add/sub 함수 사용
    , timestamp(register_stamp) AS register_stamp
    , timestamp_add(timestamp(register_stamp), interval 1 hour) AS after_1_hour
    , timestamp_add(timestamp(register_stamp), interval 30 minute) AS before_30_minutes

    -- 타임스탬프 문자열 기반으로 직접 날짜 계산을 할 수 없으므로
    -- 타임 스탬프 자료형 -> 날짜/시간 자료형 변환 뒤 계산
    , date(timestamp(register_stamp)) AS register_date
    , date_add(date(timestamp(register_stamp)), interval 1 day) AS after_1_day
    , date_sub(date(timestamp(register_stamp)), interval 1 month) AS before_1_month

    -- Hive, SparkSQL, 날짜/시각 계산 함수 제공 x
    -- unixtime으로 변환 후, 초단위로 계산 적용뒤 다시 타임스탬프로 변환
    , CAST(register_stamp AS timestamp) AS register_stamp
    , from_unixtime(unix_timestamp(register_stamp) + 60 * 60) AS after_1_hour
    , from_unixtime(unix_timestamp(register_stamp) - 30 * 60) AS before_30_minutes

    --- 타임스탬프 문자열을 날짜 변환시, to_date 함수 사용
    -- 단, hive 2.1.0 이전 버전의 경우, 문자열 자료형 리턴
    , to_date(register_stamp) AS register_date

    -- day/month 계산 시, date_add / date_months 함수 사용
    -- 단, year 계산 함수는 제공되지 않음
    , date_add(to_date(regsiter_stamp), 1) AS after_1_day
    , add_months(to_date(register_stamp), -1) AS before_1_month
  FROM mst_users_with_dates
  ;
  ```

#### 날짜 데이터들의 차이 계산
  - 두 날짜 데이터를 사용하여 날짜 데이터 차이 계산
    - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
    ```sql
    SELECT
      user_id

      -- PostgreSQL, Redshift, 날짜 자료형 끼리 연산 가능
      , CURRENT_DATE as today
      , register_stamp::date ADS register_date
      , CURRENT_DATE - register_stamp::date AS diff_days

      -- BigQuery의 경우 date_diff 함수 사용
      , CURRENT_DATE as today
      , date(timestamp(register_stamp)) AS register_date
      , date_diff(CURRENT_DATE, date(timestamp(register_stamp)), day) AS diff_Days

      -- Hive, SparkSQL의 경우 datediff 함수 사용
      , CURRENT_DATE() as today
      , to_date(register_stamp) AS register_date
      , datediff(CURRENT_DATE(), to_date(register_stamp)) AS diff_days
    FROM mst_users_with_dates
    ;
    ```

#### 사용자의 생년월일로 나이 계산하기
- 윤년 고려
  -  날짜를 365일로 나누어 계산 불가
- 나이를 위한 전용함수는 `PostgreSQL`에서만 제공
  - 날짜 자료형 데이터로 날짜 계산하는 `age`함수 제공
- `age`의 리턴 값은
  - `interval` 자료형의 날짜 단위
  - `EXTRACT` 함수로 `year`부분만 추출해야 함
  - `default`로 현재 나이를 리턴,
    - **특정 날짜를 지정**하면 해당 날짜에서의 나이 리턴
- `age`함수를 사용해 나이를 계산하는 쿼리
  - `PostgreSQL`
  ```sql
  SELECT
    user_id

    -- PostgreSQL, age 함수와 EXTRACT 함수를 이용하여 나이 집계
    , CURRENT_DATE AS today
    , regsiter_stamp::date AS register_date
    , birth_date::date AS birth_date
    , EXTRACT(YEAR FROM age(birth_date::date)) AS current_age
    , EXTRACT(YEAR FROM age(register_stamp::date, birth_date::date)) AS reguster)age
  FROM mst_users_with_dates
  ;
  ```
- `Redshift`의 `datediff` 및 `Bigquery`의 `date_idff` 함수는
  - `day`단위가 아닌 `year`단위로 출력 단위 지정 가능
  - 이를 활용하여 나이 계산
- 하지만 이를 활용하여 계산시, 단순 연의 차이를 계산
  - 해당 연의 생일을 넘었는지에 대한 계산이 포함되지 않음
- 연 부분의 차이를 계산하는 쿼리
  - `PostgreSQL`, `BigQuery`
  ```sql
  SELECT
    user_id

    -- Redshift, datediff 함수로 year을 지정하더라도, 연 부분 차이는 계산 불가
    , CURRENT_DATE AS today
    , register_stamp::date AS register_date
    , birth_date::date AS birth_date
    , datediff(year, birth_date::date, CURRENT_DATE)
    , datediff(year, birth_date::date, register_stamp::date)

    -- BigQuery, date_diff 함수로 year 지정시에도, 연 부분 차이 계산 불가
    , CURRENT_DATE AS today
    , date(timestamp(register_stamp)) AS register_Date
    , date(timestamp(birth_date)) AS birth_date
    , date_diff(CURRENT_DATE, date(timestamp(birth_date)), year) AS current_age
    , date_diff(date(timestamp(register_stamp)), date(timestamp(birth_date)), year) AS register_age
  FROM mst_users_with_dates
  ;
  ```
- 전용 함수를 사용하지 않고 나이를 계산하는 법
  - 날짜를 **고정 자리 수의 정수**로 표현
  - 그 차이를 계산
- 날짜를 정수로 표현해서 나이 계산하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT floor((20160228 - 20000229)/ 10000) AS age;
  ```
- 등록 시점 / 현재 시점의 나이를 문자열로 계산하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    user_id
    , substring(register_stamp, 1, 10) AS register_date
    , birth_date

    -- 등록 시점의 나이 계산
    , floor(
      ( CAST(replace(substring(register_stamp, 1, 10), '-'. '') AS integer)
        - CAST(replace(birth_date, '-', '') AS integer)
        ) / 10000
    ) AS register_age

    -- 현재 시점의 나이 계산
    , floor (
      ( CAST(replace(CAST(CURRENT_DATE as text), '-', '') AS integer)
        - CAST(replace(birth_datey, '-', '') AS integer)
      ) / 10000
    ) AS current_age

    -- BigQuery, text -> string, integer -> int64
    ( CAST(replace(CAST(CURRENT_DATE AS string), '-', '') AS int64)
      - CAST(replace(birth_date, '-', '') AS int64)
    ) / 10000

    -- Hive, SparkSQL, replace -> regexp_replace, text -> string
    -- integer -> int
    -- SparkSQL, CURRENT_DATE -> CURRENT_DATE()
    ( CAST(regexp_replace(CAST(CURRENT_DATE() AS string), '-', '') AS int)
      - CAST(regexp_replace(birth_date, '-', '') AS int)
    ) / 10000
  FROM mst_users_with_dates
  ;
  ```

#### 정리
- 날짜/시간 데이터의 계산은 **미들웨어**간 차이가 큼
  - 실수할 가능성이 높음
- 실무에서는 **날짜/시간 데이터** -> **수치/문자열**로 변환하여 다루는 경우가 많음

### IP 주소 다루기
- 보통 IP주소는 **문자열**로 저장
- IP주소 상호 비교나, 동일 네트워크 IP 계산시
  - 단순 문자열 비교로 어려움

#### IP 주소 자료형 활용
- `PostgreSQL`에는 IP주소를 다루기 위한 `inet` 자료형 제공
- `inet`자료형 비교시, `<`, `>`를 사용
- `inet` 자료형을 사용한 IP 주소 비교 쿼리
  - `PostgreSQL`
  ```sql
  SELECT
    CAST('127.0.0.1' AS inet) < CAST('127.0.0.2' AS inet) AS lt
    , CAST('127.0.0.1' AS inet) > CAST('192.168.0.1' AS inet) AS gt
  ```
  - 조건에 해당할경우 `t`, 아닐 경우 `f` 리턴
  `address/y` 형태의 네트워크 범위에 IP 주소 포함 여부 판정 가능
    - `<<` 또는 `>>` 연산자 사용
- `inet` 자료형을 사용해 IP 주소 범위 다루는 쿼리
  - `PostgreSQL`
  ```sql
  SELECT
    CAST('127.0.0.1' AS inet) << CAST('127.0.0.0/8' AS inet) AS is_contained;
  ```

#### 정수 또는 문자열로 IP 다루기
- IP 주소 전용 자료형이 제공되지 않는 미들웨어의 경우 다른 방법 사용
- IP 주소를 **정수 자료형**으로 변환
  - 숫자 대소/비교 가능
  - IP 주소에서 4개의 **10진수** 부분 추출 쿼리
    - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
    ```sql
    SELECT
      ip

      -- PostgreSQL, Redshift의 경우 splift_part로 문자열 분해
      , CAST(split_part(ip, '.', 1) AS integer) AS ip_part_1
      , CAST(split_part(ip, '.', 2) AS integer) AS ip_part_2
      , CAST(split_part(ip, '.', 3) AS integer) AS ip_part_3
      , CAST(split_part(ip, '.', 4) AS integer) AS ip_part_4

      -- BigQuer, split 함수로 배열 분해, n번째 요소 추출
      , CAST(split(ip, '.')[SAFE_ORDINAL(1)] AS int64) AS ip_part_1
      , CAST(split(ip, '.')[SAFE_ORDINAL(2)] AS int64) AS ip_part_2
      , CAST(split(ip, '.')[SAFE_ORDINAL(3)] AS int64) AS ip_part_3
      , CAST(split(ip, '.')[SAFE_ORDINAL(4)] AS int64) AS ip_part_4

      -- Hive, SparkSQL, split 함수로 배열 분해, n번째 요소 추출
      -- 이때 '.'가 특수문자이므로, \로 escaping
      , CAST(split(ip, '\\.')[0] AS int) AS ip_part_1
      , CAST(split(ip, '\\.')[1] AS int) AS ip_part_2
      , CAST(split(ip, '\\.')[2] AS int) AS ip_part_3
      , CAST(split(ip, '\\.')[3] AS int) AS ip_part_4
    FROM
      (SELECT '192.168.0.1' AS ip) AS t
      
      -- PostgreSQL의 경우 명시적 자료형 변환
      (SELECT CAST('192.168.0.1' AS text) AS ip) AS t
    ;
    ```
  - 추출한 4개의 10진수 부분을
    - `2^24`, `2^16`, `2^8`, `2^0`으로 곱하고 더하면
    - **정수형 자료형 표기**로 변환
  - 이렇게 되면 **대소 비교** 및 **범위 판정**이 가능
- IP 주소를 정수 자료형 표기로 변환하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    ip
    -- PostgreSQL, Redshift의 경우 splift_part로 문자열 분해
    , CAST(split_part(ip, '.', 1) AS integer) * 2^24
      + CAST(split_part(ip, '.', 2) AS integer) * 2^16
      + CAST(split_part(ip, '.', 3) AS integer) * 2^8
      + CAST(split_part(ip, '.', 4) AS integer) * 2^0
    AS ip_integer

    -- BigQuer, split 함수로 배열 분해, n번째 요소 추출
    , CAST(split(ip, '.')[SAFE_ORDINAL(1)] AS int64) * pow(2, 24)
      + CAST(split(ip, '.')[SAFE_ORDINAL(2)] AS int64) * pow(2, 16)
      + CAST(split(ip, '.')[SAFE_ORDINAL(3)] AS int64) * pow(2, 8)
      + CAST(split(ip, '.')[SAFE_ORDINAL(4)] AS int64) * pow(2, 0)
    AS ip_integer

    -- Hive, SparkSQL, split 함수로 배열 분해, nq번째 요소 추출
    -- 이때 '.'가 특수문자이므로, \로 escaping
    , CAST(split(ip, '\\.')[0] AS int) * pow(2, 24)
      + CAST(split(ip, '\\.')[1] AS int) * pow(2, 16)
      + CAST(split(ip, '\\.')[2] AS int) * pow(2, 8)
      + CAST(split(ip, '\\.')[3] AS int) * pow(2, 0)
    AS ip_integer
  FROM
    (SELECT '192.168.0.1' AS ip) AS t
    
    -- PostgreSQL의 경우 명시적 자료형 변환
    (SELECT CAST('192.168.0.1' AS text) AS ip) AS t
    ;
  ```
- IP 주소를 0으로 메우기
  - IP 주소를 비교하는 또다른 방법
  - 각 **10진수**부분을 **3자리 숫자**가 되게 앞부분을 **0**으로 채운다
  - IP 주소를 0으로 메운 문자열 변환 쿼리
    - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
    ```sql
    SELECT
      ip

      -- PostgreSQL, Redshift, lpad 함수로 0 메우기
      , lpad(split_part(ip, '.', 1), 3, '0')
        || lpad(split_part(ip, '.', 2), 3, '0')
        || lpad(split_part(ip, '.', 3), 3, '0')
        || lpad(split_part(ip, '.', 4), 3, '0')
      AS ip_padding

      -- BigQuery, split 함수로 배열 분해, n번째 요소 추출
      , CONCAT(
        lpad(split(ip, '.')[SAFE_ORDINAL(1)], 3, '0')
        , lpad(split(ip, '.')[SAFE_ORDINAL(2)], 3, '0')
        , lpad(split(ip, '.')[SAFE_ORDINAL(3)], 3, '0')
        , lpad(split(ip, '.')[SAFE_ORDINAL(4)], 3, '0')
      ) AS ip_padding

      -- Hive, SparkSQL, split 함수로 배열 분해, n번째 요소 추출
      -- .이 특수문자 이므로 \로 escaping
      , CONCAT(
        lpad(split(ip, '\\.')[0], 3, '0')
        , lpad(split(ip, '\\.')[1], 3, '0')
        , lpad(split(ip, '\\.')[2], 3, '0')
        , lpad(split(ip, '\\.')[3], 3, '0')
      ) AS ip_padding
    FROM
      (SELECT '192.168.0.1' AS ip) AS t

      -- PostgreSQL의 경우 명시적 자료형 변환
      (SELECT CAST('192.168.0.1' AS text) AS ip) AS t
    ;
    ```
    - `lpad` 함수
      - 지정한 문자 수가 되게 문자열을 **왼쪽**으로 메우는 함수
      - 모든 10진수가 `3`자리 수가 되게, 문자열 왼쪽을 `0`으로 메운다
    - 이후 메운 문자열을 `||` 연산자로 연결
    - **고정 길이 문자열**을 만든 이후,
      - **문자열 상태로 대소비교**
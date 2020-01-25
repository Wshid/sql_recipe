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
- `PostgreSQL`에는 `Point`
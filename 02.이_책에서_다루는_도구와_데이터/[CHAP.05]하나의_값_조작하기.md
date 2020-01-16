## 05. 하나의 값 조작하기

### 데이터를 가공 하는 이유
#### 다룰 데이터가 데이터 분석 용도가 아닌 경우
- 업무 데이터를 다루는 경우
  - 데이터베이스에 **코드** 값을 저장 하고
  - 해당 코드의 의미를 **다른 테이블**에서 관리
- 코드를 이용해 리포트 작성시, 무엇을 의미하는지 불명확
- 접근 로그는
  - **어떤 행동을 하나의 문자열**로 표현
  - 여러개의 정보가 **하나의 문자열**로 저장되어 있다면,
    - 이를 SQL에서 다루기는 어려움
- 데이터 분석에 적합한 형태로 미리 가공 및 저장 필요

#### 연산할 때, 비교 가능한 상태로 만들고, 오류를 회피하기 위한 경우
- **로그 데이터**와 **업무 데이터**를 함께 다루는 경우,
  - 각 데이터에 있는 **데이터 형식**이 다를 수 있음
- 두 데이터를 활용해 집계시, **데이터 형식**을 **통일** 하여야 함
- **어떤 값**과 `NULL`을 연산하면
  - 결과가 `NULL`이 나올 수 있음
- 이로 인해, 오류가 발생해 원하는 결과를 얻을 수 없는 경우가 꽤 많기 때문에
- 데이터를 미리 가공하여 `NULL`이 발생하지 않게 만드는 것이 좋음

### 코드 값을 레이블로 변경하기
- 원본 데이터
  ```sql
  -- mysql
  CREATE TABLE `05_01_mst_users` (
    `user_id` varchar(4) NOT NULL,
    `register_date` date DEFAULT NULL,
    `register_device` int(11) DEFAULT NULL,
    PRIMARY KEY (`user_id`)
  ) ENGINE=InnoDB DEFAULT CHARSET=utf8;
  ```
- sql 구문
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    user_id,
    CASE
      WHEN register_device = 1 THEN '데스크톱'
      WHEN register_device = 2 THEN '스마트폰'
      WHEN register_device = 3 THEN '애플리케이션'
      -- default 지정시, ELSE 구문 사용
      ELSE ''
    END AS device_name
  FROM mst_users;
  ```

### URL에서 요소 추출 하기
- 분석 현장에서는 **로그 조건**과 **분석 요건**을 제대로 검토하지 못하고
  - 일단 최소한의 요건으로 `referer`와 `page url`을 저장하는 경우가 있다.
- 이후 **URL**을 기반으로 요소 추출

#### 레퍼러로 어떤 웹페이지를 거쳐 넘어왔는지 판별하기
- 어떤 웹페이지를 거쳐 넘어왔는지 판별 -> **Referer**
- 보통은 **호스트 단위**로 집계한다.
  - **페이지 단위**로 집계시, 밀도가 너무 작아 복잡하기 때문
- `Hive`또는 `BigQuery`에는 **URL**을 다루는 함수 존재
  - 구현되지 않은 **미들웨어**에서는
  - **정규 표현식**으로 **호스트이름의 패턴**을 추출해야 함
- `Redshift`에서는 정규 표현식에서 괄호로 그룹화하는 기능이 없기 때문에
  - 정규식을 복잡하게 작성해야 함
- **Referer** 도메인을 추출하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    stamp
    -- PostgreSQLdml ruddn substring 함수와 정규 표현식 사용
    , substring(referrer from 'https?://([^/]*)') AS referrer_host
    
    -- Redshift의 경우, 정규 표현식에 그룹을 사용할 수 없으므로,
    -- regexp_substr 함수와 regexp-replace 함수를 조합하여 사용
    , regexp_replace(regexp_substr(referrer, 'https?://[^/]*'), 'https?://', '') AS referrer_host

    -- Hive, SparkSQL의 경우, parse_url 함수로 호스트이름 추출
    , parse_url(referrer, 'HOST') AS referrer_host

    -- BigQuery의 경우, host 함수 사용
    , host(referrer) AS referrer_host

  FROM access_log
  ;
  ```

#### URL에서 경로와 요청 매개변수 값 추출하기
- 상품에 관련된 리포트 작성 시
  - 어떤 상품이 열람되는지, 특정하는 ID를 데이터로 따로 저장해두지 않은 경웆 ㅗㄴ재
- 그래도 `URL`을 **로그 데이터**로 저장해 두었다면
  - `URL`을 가공하여 상품 리스트 생성 가능
- `URL` 경로와 `GET` 요청 매개변수에 있는 **특정 키**를 추출하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    stamp
    , url

    -- PostgreSQL의 경우 substring 함수와 정규 표현식 사용
    , substring(url from '//[^/]+([^?#]+)') AS path
    , substring(url from 'id=([^&]*)') AS id

    -- Redshift의 경우 regexp_substr 함수와 regexp_replace 함수를 조합하여 사용
    , regexp_replace(regexp_substr(url, '//[^/]+[^?#]+'), '//[^/]+', '') AS path
    , regexp_replace(regexp_substr(url, 'id=[^&]*'), 'id=', '') AS id

    -- BigQuery의 경우 정규 표현식과 regexp_extract 함수 사용
    , regexp_extract(url, '//[^/]+([^&#]+)') AS path
    , regexp_extract(url, 'id=([^&]*)') AS id

    -- Hive, SparkSQL의 경우 parse_url 함수로 url 경로 / 쿼리 매개변수 추출
    , parse_url(url, 'PATH') AS path
    , parse_url(url, 'QUERY', 'id') As id
  
  FROM access_log
  ;
  ```

#### URL 처리 정리
- 웹 서비스 로그 분석에서 자주 사용되는 기술
- 미들웨어에 따라 **함수 이름**이 다르거나,
  - **정규 표현식**의 작성 방법이 달라 문제 발생

### 문자열을 배열로 분해하기
- 빅데이터 분석에서 가장 많이 사용되는 자료형은 **문자열**
- **문자열 자료형**은 범용적인 자료형이므로,
  - 더 세부적으로 분해해서 사용하는 경우가 많음
- URL 경로를 **슬래시**로 분할하여 계층을 추출하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    stamp
    , url

    -- PostgreSQL의 경우, split_part로 n번째 요소 추출
    , split_part(substring(url from '//[^/]+([^?#]+)'), '/', 2) AS path1
    , split_part(substring(url from '//[^/]+([^?#]+)'), '/', 3) AS path2

    -- Redshift도 split_part로 n번째 요소 추출
    , split_part(regexp_replace(regexp_substr(url, '//[^/]+[^?#]+'), '//[^/]+', ''), '/', 2) AS path1
    , split_part(regexp_replace(regexp_substr(url, '//[^/]+[^?#]+'), '//[^/]+', ''), '/', 3) AS path2

    -- BigQuery의 경우 split 함수를 사용하여 배열로 자름(별도 인덱스 지정 필요)
    , split(regexp_extract(url, '//[^/]+([^&#]+)'), '/')[SAFE_ORDINAL(2)] AS path1
    , split(regexp_extract(url, '//[^/]+([^&#]+)'), '/')[SAFE_ORDINAL(3)] AS path2

    -- Hive, SparkSQL도 split 함수를 사용하여 배열로 자름
    , split(parse_url(url, 'PATH') , '/')[1] AS path1
    , split(parse_url(url, 'PATH') , '/')[2] AS path2
  
  FROM access_log
  ;
  ```

#### 정리
- `Redshift`는 공식적으로 **배열 자료형**의 데이터 처리를 지원하지 않음
  - 하지만 `split_part` 함수를 활용하여 **문자열**을 분할한 뒤 `n`번째 요소 추출 가능
- 배열의 인덱스는 일반적으로 **1**부터 시작하지만,
  - `BigQuery`의 경우, 배열의 값에 접근 방법 상이
    - 배열의 인덱스를 **0**부터 시작하려면, `OFFSET`
    - 배열의 인덱스를 **1**부터 시작하려면, `ORDINAL`을 지정
- 배열 길이 이상의 인덱스에 접근하면
  - 일반적으로는 `NULL`을 리턴하나,
  - `BigQuery`의 경우에는 **오류** 리턴
    - `NULL`을 리턴하게 하려면 `SAFE_OFFSET` 또는 `SAFE_ORDINAL`을 지정

### 날짜와 타임스탬프 다루기
- 미들웨어에 **시간 정보**를 다루는 **자료형**또는 함수에 큰 차이 존재

#### 현재 날짜와 타임스탬프 추출
- `PostgreSQL`
  - `CURRENT_TIMESTAMP`의 리턴 값으로 **타임존**이 적용된 **타임스탬프**의 자료형 리턴
- 이외의 미들웨어는 **타임존 없는 타임스탬프** 반환
- 리턴값의 **자료형**을 맞출 수 있게
  - `PostgreSQL`에서는 `LOCALTIMESTAMP`를 활용하는 것이 좋음
  - `BigQuery`에서는 `UTC` 시간을 리턴
    - 따라서 `CURRENT_TIMESTAMP`로 리턴되는 시간이 한국과 다름
- 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    -- PostgreSQL, Hive, BigQuery의 경우
    CURRENT_DATE AS dt
    , CURRENT_TIMESTAMP AS stamp

    -- Hive, BigQuery, SparkSQL
    CURRENT_DATE() AS dt
    , CURRENT_TIMESTAMP() AS stamp

    -- Redshift, 현재 날짜는 CURRENT_DATE, 현재 타임 스탬프는 GETDATE() 사용
    CURRENT_DATE AS dt
    , GETDATE() AS stamp

    -- PostgreSQL, CURRENT_TIMESTAMP, timezone이 적용된 타임스탬프
    -- 타임존을 적용하고 싶지 않을 때, LOCALTIMESTAMP 사용
    , LOCALTIMESTAMP AS stamp
    ;
  ```

#### 지정된 값의 날짜, 시각 데이터 추출하기
- 현재 시각이 아니라,
  - **문자열**로 지정한 **날짜**와 **시간**을
  - **날짜 자료형**과 **타임 스탬프** 자료형의 데이터를 만드는 경우 존재
- 미들웨어에 따라 다양한 방법 존재,
  - `CAST` 함수를 사용하는 것이 일반적
- 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  -- 문자열을 날짜/타임스탬프로 변환

  SELECT
  -- PostgreSQL, Hive, Redshift, Bigquery, SparkSQL 모두
  -- `CAST(value AS type)` 사용
  CAST('2016-01-30' AS date) AS dt
  , CAST('2016-01-30 12:00:00' AS timestamp) AS stamp

  -- Hive, Bigquery, `type(value)` 사용
  date('2016-01-30') AS dt
  , timestamp('2016-01-30 12:00:00') AS stamp

  -- PostgreSQL, Hive, Redshift, BigQuery, SparkSQL, `type value` 사용
  -- 단, value는 상수이므로, 컬럼 이름 지정 불가능
  date '2016-01-30' AS dt
  , timestamp '2016-01-30 12:00:00' AS stamp

  -- PostgreSQL, Redshift, `value::type` 사용
  '2016-01-30'::date AS dt
  , '2016-01-30 12:00:00'::timestamp AS stamp
  ```

#### 날짜/시각에서 특정 필드 추출
- **타임스탬프** 자료형 데이터에서
  - **년**과 **월** 등의 **특정 필드 값 추출**시, `EXTRACT` 함수 사용
- `EXTRACT` 함수를 `Hive`, `SparkSQL`은 지원하지 않음
  - 별도의 함수 제공
- 타임스탬프 자료형에서 연, 월, 일 등을 추출하는 쿼리
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    stamp
    -- PostgreSQL, Redshift, BigQuery, EXTRACT 함수 사용
    , EXTRACT(YEAR  FROM stamp) AS year
    , EXTRACT(MONTH FROM stamp) AS month
    , EXTRACT(DAY   FROM stamp) AS day
    , EXTRACT(HOUR  FROM stamp) AS hour

    -- Hive, SparkSQL
    , YEAR(stamp) AS year
    , MONTH(stamp) AS month
    , DAY(stamp) AS day
    , HOUR(stamp) AS hour
  FROM
    (SELECT CAST('2020-01-16 22:22:00' AS timestamp) AS stamp) AS t
  ```
- **날짜 자료형**과 **타임 스탬프 자료형**을 사용하지 않아도,
  - 타임스탬프를 **단순한 문자열** 취급하여 필드 추출 가능
- `substring`함수를 사용해 문자열을 추출하는 쿼리
  - 미들웨어에 큰 차이가 없다.
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    stamp

    -- PostgreSQL, Hive, Redshift, SparkSQL, substring 함수 사용
    , substring(stamp, 1, 4) AS year
    , substring(stamp, 6, 2) AS month
    , substring(stamp, 9, 2) AS day
    , substring(stamp, 12, 2) AS hour
    -- 연, 월을 함께 추출
    , substring(stamp, 1, 7) AS year_month

    --- PostgreSQL, Hive, BigQuery, SparkSQL, substr 함수 사용
    , substr(stamp, 1, 4) AS year
    , substr(stamp, 6, 2) AS month
    , substr(stamp, 9, 2) AS day
    , substr(stamp, 12, 2) AS hour
    , substr(stamp, 1, 7) AS year_month
  FROM
    -- PostgreSQL, Redshift의 경우 문자열 자료형(text)
    (SELECT CAST('2020-01-16 22:26:00' AS text) AS stamp) AS t

    -- Hive, BigQuery, SparkSQL의 경우 문자열 자료형(string)
    (SELECT CAST('2020-01-16 22:26:00' AS string) AS stamp) AS t
  ```

#### 정리
- 날짜와 시간 정보는 로그데이터에서 필수적
- **타임존**을 고려해야 하며, 미들웨어의 차이가 존재

### 결손 값을 디폴트 값으로 대체하기
- **문자열** 또는 **숫자**를 다룰 때
  - 중간에 `NULL`이 들어있는 경우를 주의해야 함
- `NULL`과 문자열을 결합 => `NULL`
- `NULL`과 숫자를 사칙연산 => `NULL`
- 처리 대상인 데이터가, 원하는 형태가 아닐 경우, 데이터 가공에는 필수적
- 구매액에서 할인 쿠폰 값을 제외한 매출 금액을 구하는 쿼리
  - 구매액과 `NULL`을 포함하는 쿠폰 금액이 저장된 테이블이 있을 때
  - Table
    - `purcahse_log_with_coupon(purchase_id, amount, coupon)`
  - `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    purchase_id
    , amount
    , coupon
    , amount - coupon AS discount_amount1
    , amount - COALESCE(coupon, 0) AS discount_amount2
  FROM
    purchase_log_with_coupon
  ```
  - `price - coupon` 연산을 할때, `coupon`이 `NULL`일 경우, `NULL`이 된다.
  - `COALESCE`를 사용하여 `0`으로 대치
# 14. 사이트 전체의 특징/경향 찾기
- 웹 사이트 전체의 특징과 경향을 찾기 위한 리포트와 SQL
- **접근 해석 도구**를 도입한 서비스에서는 이미 파악한 값일 수 있음
- **접근 분석 도구**로 집계한 결과 및 리포트도, 어느정도는 필터링이 가능하겠으나,
  - 제공되는 기능 이외의 리포트는 작성할 수 없음
  - 리포트의 설계가 **접근 분석 도구**의 기능으로 인해, 제한 될 수 있음
- 위 제한을 넘어, 자유롭게 리포트를 설계할 수 있다는 점이
  - 빅데이터 기반을 도입하는 장점
- **접근 분석 도구**에서 제공되는 결과에서
  - 추가로 복잡한 조건으로 **드릴 다운**해서 상세를 찾으려면,
  - 접근 분석 도구에서는 **최저한의 상태**로 집계할 항목을 추출해야 함
- 또한, 접근 분석 도구에서 활용할 수 없는
  - **사용자 데이터** 또는 **상품 데이터**등의 업무 데이터와 조합하면,
  - 작성할 수 없었던 리포트 작성 가능

### 샘플 데이터
- access_log : stamp / short_session / long_session / url / referer
- purchase_log : stamp / short_session / long_session / purchase_id / amount

## 1. 날짜별 방문자 수 / 방문 횟수 / 페이지 뷰 집계하기
- `COUNT`, `COUNT(DISTINCT ~ )`
- 방문자 수, 방문 횟수, 페이지 뷰
- 웹 사이트에서는
  - `방문자 수`, `방문 횟수`, `페이지 뷰 집계`가 기본
  - 각 지표가 무엇을 집계하는지 이해하고 활용할 것
- 각 용어의 의미
  - 방문자 수
    - 브라우저를 꺼도 사라지지 않는 **쿠키의 유니크 수**
    - `e.g. 한명의 사용자가 1일에 3회 방문에도, 1회로 집계`
  - 방문 횟수
    - 브라우저를 껏을 때 **사라지는** 쿠키의 유니크 수
    - `e.g. 한명의 사용자가 1일에 3회 방문하면, 3회로 집계`
  - 페이지 뷰
    - 페이지를 출력한 로그의 줄 수
- **1회 방문당 페이지 뷰**를 날짜별로 집계하기
  - 일자 / 방문자 수 / 방문 횟수 / 페이지 뷰 / 1번 방문당 페이지 뷰

### CODE.14.1. 날짜별 접근 데이터를 집계하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
```sql
SELECT
  -- PostgreSQL, Hive, Redshift, SparkSQL, subString으로 날짜 부분 추출
  substring(stamp, 1, 10) as dt
  -- PostgreSQL, Hive, BigQuery, SparkSQL, substr 사용
  , substr(stamp, 1, 10) AS dt
  -- 쿠키 계산하기
  , COUNT(DISTINCT long_session) AS access_users
  -- 방문 횟수 계산
  , COUNT(DISTINCT short_session) AS access_count
  -- 페이지 뷰 계산
  , COUNT(*) AS page_view

  -- 1인당 페이지 뷰 수
  -- PostgreSQL, Redshift, BigQuery, SparkSQL, NULLIF 함수 사용
  , 1.0 * COUNT(*) / NULLIF(COUNT(DISTINCT long_session), 0) AS pv_per_user
  -- Hive, NULLIF 함수 대신 CASE 사용
  , 1.0 * COUNT(*)
      / CASE
          WHEN COUNT(DISTINCT long_session) <> 0 THEN COUNT(DISTINCT long_session)
        END
    AS pv_per_user
FROM
  access_log
GROUP BY
  -- PostgreSQL, Redshift, BigQuery
  -- SELECT 구문에서 정의나 별칭을 GROUP BY에 지정할 수 있음
  dt
  -- PostgreSQL, Hive, Redshift, SparkSQL
  -- SELECT 구문에서 별칭을 지정하기 이전의 식을 GROUP BY에 지정할 수 있음
  substring(stamp, 1, 10)
ORDER BY
  dt
;
```
- 사이트에서 로그인 기능을 제공하는 경우
  - 추가로 **로그인 UU**와 **비로그인 UU**도 동시에 집계하면 좋음
    - **CHAP.11.1** 참고

### 원포인트
- **페이지 뷰** 정보만 필요하다 해도,
  - 그 이외의 지표를 집계할 때마다, 별도의 `SQL`을 작성해서, 실행하기엔 번거로움
- 따라서 한번에 모든 정보를 추출할 수 있는 SQL을 만들어 사용하기

## 2. 페이지별 쿠키 / 방문 횟수 / 페이지 뷰 집계하기
- 로그 데이터에는 URL이 포함된 경우가 많음
  - **URL**을 집계하면, 각 페이지의 방문 횟수, 페이지 뷰 등 집계 가능
- 로그 데이터의 URL에는
  - **화면 제어**를 위한 매개변수
  - **광고 계측**(클릭 수 계산 등)을 위한 매개변수 등이 포함되는 경우가 많음
- 따라서 **한 페이지**를 가리키는 여러 **URL**이 존재하므로,
  - 단순한 방법으로는 페이지 뷰를 구하기 어려움

### CODE.14.2. URL별로 집계하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
```sql
SELECT
  url
  , COUNT(DISTINCT short_session) AS access_count
  , COUNT(DISTINCT long_session) AS access_users
  , COUNT(*) AS page_view
FROM
  access_log
GROUP BY
  url
;
```

### CODE.14.3. 경로별로 집계하는 쿼리
- 상세 페이지를 구분하지 않음
- 요청 매개변수를 생략하고, **경로**만으로 집계
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
```sql
WITH
access_log_with_path AS (
  -- URL에서 경로 추출
  SELECT
    -- PostgreSQL의 경우, 정규 표현식으로 경로 부분 추출
    , substring(url from '//[^/]+([^?#]+)') AS url_path
    -- Redshift의 경우, regexp_substr 함수와 regexp_replace 함수를 조합하여 사용
    , regexp_replace(regexp_substr(url, '//[^/]+[^?#]+'), '//[^/]+', '') AS url_path
    -- BigQuery의 경우 정규 표현식에 regexp_extract 사용
    , regexp_extract(url, '//[^/]+([^?#]+)') AS url_path
    -- Hive, SparkSQL, parse_url 함수로 url 경로 부분 추출
    , parse_url(url, 'PATH') AS url_path
  FROM access_log
)
SELECT
  url_path
  , COUNT(DISTINCT short_session) AS access_count
  , COUNT(DISTINCT long_session) AS access_users
  , COUNT(*) AS page_view
FROM
  access_log_with_path
GROUP BY
  url_path
;
```

### CODE.14.4. url에 의미 부여하여 집계
- 카테고리/리스트 페이지로 그룹핑 하기
  - `/list/newly/` -> 신상품 리스트 페이지로 의미 부여
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
```sql
WITH
access_log_with_path AS (
  ...
)
, access_log_with_split_path AS (
  -- 경로의 첫 번째 요소와 두 번째 요소 추출하기
  SELECT *
    -- PostgreSQL, Redshift, split_part로 n번째 요소 추출하기
    , split_part(url_path, '/', 2) AS path1
    , split_part(url_path, '/', 3) AS path2
    -- BigQuery, split 함수로 배열로 분해하고 추출하기
    , split(url_path, '/')[SAFE_ORDINAL(2)] AS path1
    , split(url_path, '/')[SAFE_ORDINAL(3)] AS path2
    -- Hive, SparkSQL, split 함수로 배열로 분해하고 추출하기
    , split(url_path, '/')[1] AS path1
    , split(url_path, '/')[2] AS path2
  FROM access_log_with_path
)
, access_log_with_page_name AS (
  -- 경로를 /로 분할하고, 조건에 따라 페이지 이름 붙이기
  SELECT *
  , CASE
      WHEN path1 = 'list' THEN
        CASE
          WHEN path2 = 'newly' THEN 'newly_list'
          ELSE 'category_list'
        END
      -- 이외의 경우에는 경로 그대로 사용
      ELSE url_path
    END AS page_name
  FROM access_log_with_split_path
)
SELECT
  page_name
  , COUNT(DISTINCT short_session) AS access_count
  , COUNT(DISTINCT long_session) AS access_users
  , COUNT(*) FROM page_view
FROM access_log_with_page_name
GROUP BY page_name
ORDER BY page_name
;
```
- 전체 페이지 뷰를 세가지로 구분
  - 최상위 페이지
  - 카테고리/리스트 페이지
  - 상세 페이지
- `category_list`와 `newly_list`는
  - `CASE` 식으로 정의한 명칭 이외의 경우는
  - 경로를 기반으로 만들어진 이름이므로, `/`가 포함되어 있음
- 구하고자 하는 리포트의 용도에 따라 **집계의 밀도**를 정하고 집계할 것

### 원포인트
- 로그를 저장할 때, 해당 페이지에 어떤 의미가 있는지 알려주는 **페이지 이름**을 따로 전송하면,
  - 집계할 때 매우 용이함
- 로그를 설계할 때 **이후의 집계 작업**을 설계하고 검토할 것


## 3. 유입원별로 방문 횟수 또는 CVR 집계하기
- 주요 개념
  - SQL : 정규 표현식, URL 함수
  - 분석 : 유입원, URL 생성도구, CVR
- 웹 사이트 접근시, **직접 URL** 접근 외에
  - 다른 사이트의 링크 등을 클릭하는 방법도 존재
- 주요 유입 경로
  - 검색 엔진
  - 검색 연동형 광고
  - 트위터, 페이스북 등의 소셜 미디어
  - 제휴 서비스
  - 광고 네트워크(ad network)
  - 다른 웹사이트에 붙은 링크(블로그 소개 기사 등)
- 유입 경로를 **개별적**으로 집계하면
  - 웹사이트의 방문자가 **어떤 행동**을 해서, 우리 웹사이트를 방문하는 것인지 알 수 있음
- 추가로 검색 연동형 광고, 제휴, 광고 네트워크 등을 관리하는
  - 마케팅 부문의 효과를 시각적으로 표현하는 목적으로도 사용할 수 있음

### 유입원 판정
- 직전 페이지의 URL을 레퍼러(referer)라고 부름
- 이러한 레퍼러 등을 활용하면, 다음과 같은 방법으로 `URL`판정이 가능
  - URL **매개변수** 기반 판정
  - **레퍼러 도메인**과 **랜딩 페이지**를 사용한 판정

#### URL 매개변수 기반 판정
- 광고 담당자는 광고가 얼마나 퍼졌는지 집계하고자
  - `URL`에 광고 계측 전용 매개변수를 설정
- 대표적으로 `GA`(Google Analytics)를 도입하면,
  - URL 생성 도구를 기반으로 **매개변수를 추가**하는 방법을 사용하는 경우가 많음
  - 매개변수를 사용해, 유입 수를 리포트 하는 기능 제공
- URL 생성 도구
  - GA에서 사용하는 `URL 생성도구`(Compaign URL Builder)은
    - 웹사이트의 `URL` 또는 `Compaign Source`등
      - 필요한 **매개변수**를 입력하면, URL을 생성
  - `Compaign URL Builder`를 사용한 URL
    - `www.~~~.com?utm_source=google&utm_medium=cp&utm_compaign=spring_sale`
  - `Compaign URL Builder`의 속성
    **URL 생성 도구 항목**|**URL 매개변수**|**용도**
    -----|-----|-----
    Campain Source(필수)|utm\_source|The referer; 사이트 이름 또는 매체의 명칭
    Compaign Medium|utm\_medium|Marketing medium; 유입 분류
    Compaign Name|utm\_compaign|Product, promo code, or slogan; 캠페인 페이지의 이름
    Compaign Term|utm\_term|Identity the paid keywords; 유입 키워드
    Compaign Content|utm\_content|Use to differentiate ads; 광고를 분류하는 식별자
  - 예시
    - `utm_source > utm_medium > utm_compaign` 계층구조
      ```log
      yahoo > cpc > 201608
      yahoo > cpc > 201609
      yahoo > banner > 2016spring_sale
      ```
- URL 매개변수를 사용하면, 다양한 유입 계측이 가능
  - 예를 들어 `URL`에 여러 가지 **매개 변수**를 지정하고, 이를 활용해 QR코드를 만들면,
    - QR코드를 전단지와 잡지에 넣어, 오프라인 광고 효과 계측이 가능

##### 레퍼러 도메인과 랜딩 페이지를 사용한 판정
- 검색 엔진, 개인 블로그, 트위터 처럼 **광고 담당자**가 조작할 수 없는 영역은
  - URL 매개변수를 활용할 수 없음
- 이러한 경우, `referer`를 사용해야만, 어느 곳에서 유입되는지 확인 가능
- 추가적으로 **전단지**와 **잡지**등에서
  - 회사 이름과 상품 이름이 아닌 **특징적인 어구**를 넣고,
    - 이를 검색하게 유도한 뒤, 이벤트 사이트로 들어오게 해서
    - 오프라인 광고의 효과를 계측할 수 있음

### 유입원별 방문 횟수 집계하기
- `레퍼러가 있는 경우`와 `자신의 사이트가 아닌 경우`라는 두가지 조건을 만족할 때
  - 다음과 같은 로직으로 유입원 별 방문 횟수 집계

**유입 경로**|**판정 방법**|**집계 방법**
-----|-----|-----
검색 연동 광고|URL 생성 도구로 만들어진 매개 변수|URL에 utm\_source, utm\_medium이 있을 경우, 이러한 두 개의 문자열을 결합하여 집계
제휴 마케팅 사이트|URL 생성 도구로 만들어진 매개 변수|URL에 utm\_source, utm\_medium이 있을 경우, 이러한 두 개의 문자열을 결합하여 집계
검색 엔진|도메인|레퍼러의 도메인이 다음과 같은 검색엔진일 때 <br />  - search.naver.com <br /> - www.google.co.kr
소셜 미디어|도메인|레퍼러의 도메인이 다음과 같은 소셜 미디어일 때 <br /> - twitter.com <br /> - www.facebook.com
기타 사이트|도메인|위에 언급한 도메인이 아닐 경우, other라는 이름으로 집계

#### CODE.14.5. 유입원별로 방문 횟수를 집계하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
```sql
WITH
access_log_with_parse_info AS (
  -- 유입원 정보 추출하기
  SELECT *
    -- PostgreSQL, 정규 표현식으로 유입원 정보 추출하기
    , substring(url from 'https?://([^\]*)') AS url_domain
    , substring(url from 'utm_source=([^&]*)') AS url_utm_source
    , substring(url from 'utm_medium=([^&]*)') AS url_utm_medium
    , substring(referer from 'https?://([^/]*)') AS referer_domain
    -- Redshift, regexp_substr 함수와 regexp_replace 함수를 조합하여 사용
    , regexp_replace(regexp_substr(url, 'https?://[^/]*'), 'https?://', '') AS url_domain
    , regexp_replace(regexp_substr(url, 'utm_source=[^&]*'), 'utm_source=', '') AS url_utm_source
    , regexp_replace(regexp_substr(url, 'utm_medium=[^&]*'), 'utm_medium=', '') AS url_utm_medium
    , regexp_replace(regexp_substr(referer, 'https?://[^/]*'), 'https?://', '') AS referer_domain
    -- BigQuery, 정규표현식 regexp_extract 사용
    , regexp_extract(url, 'https?://([^/]*)') AS url_domain
    , regexp_extract(url, 'utm_source=([^&]*)') AS url_utm_source
    , regexp_extract(url, 'utm_medium([^&]*)') AS url_utm_medium
    , regexp_extract(referer, 'https?://([^/]*)') AS referer_domain
    -- Hive, SparkSQL, parse_url 함수 사용
    , parse_url(url, 'HOST') AS url_domain
    , parse_url(url, 'QUERY', 'utm_source') AS url_utm_source
    , parse_url(url, 'QUERY', 'utm_medium') AS url_utm_medium
    , parse_url(referer, 'HOST') AS referer_domain
  FROM access_log
)
, access_log_with_via_info AS (
  SELECT *
    , ROW_NUMBER() OVER(ORDER BY stamp) AS log_id
    , CASE
        WHEN url_utm_source <> '' AND url_utm_medium <> ''
          -- PostgreSQL, Hive, BigQuery, SpaerkSQL, concat 함수에 여러 매개변수 사용 가능
          THEN concat(url_utm_source, '-', url_utm_medium)
          -- PostgreSQL, Redshift, 문자열 결합에 || 연산자 사용
          THEN url_utm_source || '-' || url_utm_medium
        WHEN referer_domain IN ('search.naver.com', 'www.google.co.kr') THEN 'search'
        WHEN referer_domain IN ('twitter.com', 'www.facebook.com') THEN 'social'
        ELSE 'other'
      -- ELSE referer_domain으로 변경하면, 도메인별 집계 가능
      END AS via
  FROM access_log_with_parse_info
  -- 레퍼러가 없는 경우와 우리 사이트 도메인의 경우 제외
  WHERE COALESCE(referer_domain, '') NOT IN ('', url_domain)
)
SELECT via, COUNT(1) AS access_count
FROM access_log_with_via_info
GROUP BY via
ORDER BY access_count DESC;
```

### 유입원별로 CVR 집계하기
- 위 쿼리 결과 에서, 각 방문에서 구매한 비율(CVR) 집계하는 쿼리를 작성하기
  - 구매 비율을 집계하되, `WITH` 구문의 `access_log_with_purchase_amount` 테이블 수정시,
  - 다양한 액션의 비율을 집계할 수 있음

#### CODE.14.6. 각 방문에서 구매한 비율(CVR)을 집계하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
```sql
WITH
access_log_with_parse_info AS (
  ...
)
, access_log_with_via_info AS (
  ...
)
, access_log_with_purchase_amount AS (
  SELECT
    a.log_id
    , a.via
    , SUM(
        CASE
          -- PostgreSQL, interval 자료형의 데이터로 날짜와 시간 사칙연산 가능
          WHEN p.stamp::date BETWEEN a.stamp::date AND a.stamp::date + '1 day'::interval
          -- Redshift, dateadd 함수
          WHEN p.stamp::date BETWEEN a.stamp::date AND dateadd(day, 1, a.stamp::date)
          -- BigQuery, date_add 함수
          WHEN date(timestamp(p.stamp))
            BETWEEN date(timestamp(a.stamp))
              AND date_add(date(timestamp(a.stamp)), interval 1 day)
          -- Hive, SparkSQL, date_add
          -- *BigQuery와 서식이 조금 다름
          WHEN to_date(p.stamp)
            BETWEEN to_date(a.stamp) AND date_add(to_date(a.stamp), 1)

            THEN amount
        END
      ) AS amount
  FROM
    access_log_with_via_info AS a
    LEFT OUTER JOIN
      purchase_log AS p
      ON a.long_session = p.long_session
  GROUP BY a.log_id, a.via
)
SELECT
  via
  , COUNT(1) AS via_count
  , count(amount) AS conversions
  , AVG(100.0 * SIGN(COALESCE(amount, 0))) AS cvr
  , SUM(COALESCE(amount, 0)) AS amount
  , AVG(1.0 * COALESCE(amount, 0)) AS avg_amount
FROM
  access_log_with_purchase_amount
GROUP BY via
GROUP BY cvr DESC
```
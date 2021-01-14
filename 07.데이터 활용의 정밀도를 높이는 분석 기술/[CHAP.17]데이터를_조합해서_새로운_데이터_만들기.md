# 17. 데이터를 조합해서 새로운 데이터 만들기
- 데이터 분석시, 원하는 데이터가 없는 경우가 존재
- db를 아무리 잘 설계하더라도, 시간이 지나면서, 분석할 데이터가 분석해질 수 있음
  - 처음부터 **서비스 운영**을 목적으로 데이터를 설계하고,
  - 분석을 고려하지 않았다면, 문제가 더 커질 수 있음
- 이 때, **새로운 데이터**를 준비할 수 있다면,
  - 완전히 새로운 데이터 분석이 가능함
- 최근에는 `국가`, `대학`, `연구 기관` 등에서 공개하는 오픈 데이터를 쉽게 활용할 수 있음
  - 직접 구하기 어려운 데이터는 **오픈 데이터**를 활용해 **서비스와 결합**할 것
- 이번 절의 주요 내용
  - 외부 데이터를 읽어 들이는 방법과, 데이터 가공하는 방법
  - 이를 통해 분석의 폭을 넓히고 **데이터의 정밀도**를 높이는 방법 소개

## 1. IP 주소를 기반으로 국가와 지역 보완하기
- 개념
  - SQL : inet 자료형
  - 분석 : 국가와 지역 판정
- 사용자 로그에 **IP 주소**가 있다면 **국가**와 **지역** 보완 가능
- 이를 활용할 경우,
  - 사용자를 **지역별**로 구분하거나 **타임존**에 따라 구분해서 분석 가능
- 무료로 사용가능한 **Geolocation database**를 사용하여
  - `IP 주소를 기반으로 국가와 지역 보완`
- 코드 예는 `PostgreSQL`만 사용

### DATA.17.1. IP 주소를 포함하는 액션 로그(action_log_with_ip) 테이블
**session**|**user\_id**|**action**|**ip**|**stamp**
-----|-----|-----|-----|-----
0CVKaz|U001|view|216.58.220.238|2015.11.3 18:00:00
1QceiB|U002|view|98.139.183.24|2016.11.3 19:00:00

### GeoLite2 내려받기
- `GeoLite2`는 `MaxMind` 업체에서 제공하는 **무료 지오로케이션 데이터베이스**
- 다양한 C에서 활용할 수 있는 **바이너리 파일**, **csv** 파일 형식을 제공
- 일반적으로 한달에 한 번 데이터 업데이트
- 데이터 종류
  - GeoLite2 Country : 국가 정보만 필요할 경우
  - GeoLite2 City : 도시 정보까지 필요한 경우

### CODE.17.1. GeoLite2의 CSV 데이터를 로드하는 쿼리
- `PostgreSQL`
  ```sql
  DROP TABLE IF EXISTS mst_city_ip;
  CREATE TABLE mst_city_ip (
    network                           inet PRIMARY KEY
    , geoname_id                      integer
    , registered_country_geoname_id   integer
    , represented_country_geoname_id  integer
    , is_anonymous_proxy              boolean
    , is_satellite_provider           boolean
    , postal_code                     varchar(255)
    , latitude                        numeric
    , longitude                       numeric
    , accuracy_radius                 integer
  );

  DROP TABLE IF EXISTS mst_locations;
  CREATE TABLE mst_locations (
    , geoname_id                      integer PRIMARY KEY
    , locale_code                     varchar(255)
    , contient_code                   varchar(10)
    , contient_name                   varchar(255)
    , country_iso_code                varchar(10)
    , country_name                    varchar(255)
    , subdivision_1_iso_code          varchar(10)
    , subdivision_1_name              varchar(255)
    , subdivision_2_iso_code          varchar(10)
    , subdivision_2_name              varchar(255)
    , city_name                       varchar(255)
    , metro_code                      integer
    , time_zone                       varchar(255)
  );

  COPY mst_city_ip FROM '/path/to/GeoLite2-City-Blocks-IPv4.csv' WITH CSV HEADER;
  COPY mst_locations FROM '/path/to/GeoLite2-City-Locations-en.csv' WITH CSV HEADER;
  ```

### IP 주소로 국가와 지역 정보 보완하기
- `GeoLite2` 데이터 로드가 완료되면
  - **액션 로그**의 `IP 주소`와 결합해서, **국가**와 **지역 정보** 보완
- 다음 코드는
  - `IP 주소`를 기반으로 지역, 국가, 도시, 타임존 등의 지역 정보를 보완하는 쿼리

### CODE.17.2. 액션 로그의 IP 주소로 국가와 지역 정보를 추출하는 쿼리
- `PostgreSQL`
  ```sql
  SELECT
    a.ip
    , l.contient_name
    , l.country_name
    , l.city_name
    , l.time_zone
  FROM
    action_log AS a
    LEFT JOIN
      mst_city_ip AS i
      ON a.ip::inet << i.network
    LEFT JOIN
      mst_locations AS l
      ON i.geoname_id = l.geoname_id
  ```

### 원포인트
- `PostgreSQL` 이외의 미들웨어에서 `IP 주소`를 다루는 방법은 **6.6** 참고
  - 네트워크 범위의 최소/최대 IP를 계산하고,
    - **비교 가능한 형식으로 변환**해야, 다양하게 활용 가능
- `CSV` 파일의 데이터를 전처리 해주는 `GeoIP2 CSV Format Converter`등의 도구도 살펴볼 것

## 2. 주말과 공휴일 판단하기
- 개념
  - 분석 : 주말/공휴일
- 일반적인 서비스의 경우 **주말**과 **공휴일**에
  - **방문 횟수**와 `CV`가 늘어남
  - 물론 반대의 서비스도 존재
- 월별로 목표를 세울 때,
  - 해당 연도와 해당 월에 있는 **주말**과 **공휴일**이 얼마나 되는지 계산하면
  - 더 정확한 목표를 세울 수 있음
- 로그 데이터가 **주말**과 **공휴일**에 기록된 것인지 판정하는 방법

### DATA.17.4. 접근 로그(access_log) 테이블
>**session**|**user\_id**|**action**|**stamp**
>-----|-----|-----|-----
>98900e|U001|view|2016-10-01 18:00:00
>98900e|U001|view|2016-10-01 20:00:00
>1cf768|U002|view|2016-10-04 23:00:00

### 공휴일 정보 테이블
- **날짜 데이터**를 사용하면 쉽게 **공휴일**과 **일요일** 파악 가능
- 하지만 공휴일(설날, 추석)은 판정할 수 없음
- **공휴일**을 판정하려면 **별도의 데이터**가 필요 함
- 일반적으로는 공휴일을 저장할 수 있도록 **공휴일 정보 테이블**을 만들어서 사용

#### CODE.17.3. PostgreSQL에서의 주말과 공휴일을 정의하는 방법
- PostgreSQL
  ```sql
  CREATE TABLE mst_calendar(
    year            integer
    , month         integer
    , day           integer
    , dow           varhcar(10)
    , dow_num       integer
    , holiday_name  varchar(255)
  );
  ```
- 테이블 데이터
  >**year**|**month**|**day**|**dow**|**dow\_num**|**holiday\_name**
  >-----|-----|-----|-----|-----|-----
  >2015|1|1|Thu|4|신정
  >2015|1|2|Fri|5| 

### 주말과 공휴일 판정하기
- 전처리를 생략하고 `JOIN` 조건으로 바로 날짜를 결합
  - 실제 운용시에는 성능을 위해 **미리 날짜 전용 컬럼 생성**이 필요

#### CODE.17.4. 주말과 공휴일을 판정하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    a.action
    , a.stamp
    , c.dow
    , c.holiday_name
    -- 주말과 공휴일 판정
    , c.dow_num IN (0, 6) -- 토요일과 일요일 판정하기
    OR c.holiday_name IS NOT NULL -- 공휴일 판정하기
    AS is_day_off
  FROM
    access_log AS a
    JOIN
      mst_calendar AS c
      -- 액션 로그의 타임스탬프에서 연, 월, 일을 추출하고 결합하기
      -- PostgrdSQL, Hive, Redshift, SparkSQL의 경우 다음과 같이 사용
      -- BigQuery의 경우 substring을 substr, int를 int64로 수정하기
      ON  CAST(substring(a.stamp, 1, 4) AS int) = c.year
      AND CAST(substring(a.stamp, 6, 2) AS int) = c.month
      AND CAST(substring(a.stamp, 9, 2) AS int) = c.day
  ;
  ```

## 3. 하루 집계 범위 변경하기
- 개념
  - SQL : 날짜/시간 함수
  - 분석 : 날짜 범위
- 날짜별로 데이터를 그냥 집계하게 되면
  - 자정(`0시`)기준으로 데이터가 처리됨
- 대부분의 서비스는 **자정 전/후**의 사용비율이 꽤 많은 편
  - 이를 다른 날짜로 구분할 경우 **사용자의 패턴 분석이 어려울 수 있음**
- **지속률**을 계산하는 경우에도, 등록 시간이 `23:59`이라면
  - 날짜가 바뀌면서 한 번만 사용해도 **2일간 사용자**로 취급 됨
- 이번 절에서는
  - 사용자의 활동이 가장 적은 **오전 4시**정도를 기준으로
  - 오전 4시부터 다음 날 오전 **오전 3시 59분 59초**까지를 하루로 집계할 수 있게 데이터를 가공하기

### 샘플 데이터
- 액션 로그를 사용하여 `0시 ~ 4시`까지의 데이터를 **이전 날의 데이터**로 취급
  - `session, user_id, action, stamp`의 컬럼을 가짐

### 하루 집계 범위 변경
- **하루 집계 범위**를 **오전 4시**에서 시작하게 변경하고 싶다면,
  - 타임 스탬프의 시간을 **4시간 당기면 됨**
- `raw_date`와 `mod_date`(4시간 당긴 날짜)를 구하는 예시
- `11월 4일 오전 0시 ~ 3시 59분`에 있는 로그가 `11월 3일`로 판정된 것을 볼 수 있음

#### CODE.17.5. 날짜 집계 범위를 오전 4시로부터 변경하는 쿼리
```sql
WITH
action_log_with_mod_stamp AS (
  SELECT *
  -- 4시간 전의 시간 계산하기
  -- PostgreSQL의 경우 interval 자료형의 데이터를 사용해 날짜를 사칙연산 할 수 있음
  , CAST(stamp::timestamp - '4 hours'::interval AS text) AS mod_stamp
  -- Redshift의 경우 dateadd 함수 사용하기
  , CAST(dateadd(hour, -4, stamp::timestamp) AS text) AS mod_stamp
  -- BigQuery의 경우 timestamp_sub 함수 사용하기
  , CAST(timestamp_sub(timestamp(stamp), interval 4 hour) AS string) AS mod_stamp
  -- Hive, SparkSQL의 경우 한 번 unixtime으로 변환한 뒤 초 단위로 계산하고, 다시 타임스탬프로 변환하기
  , from_unixtime(unix_timestamp(stamp) -4 * 60 * 60) AS mod_stamp
  FROM action_log
)
SELECT
  session
  , user_id
  , action
  , stamp
  -- raw_date와 4시간 후를 나타내는 mod_date 추출하기
  -- PostgreSQL, Hive, Redshift, SparkSQL의 경우 다음과 같이 사용
  , substring(stamp, 1, 10) As raw_date
  , subsing(mod_stamp, 1, 10) As mod_date
  -- BigQuery의 경우 substring을 substr로 수정하기
  , substr(stamp, 1, 10) AS raw_date
  , substr(mod_stamp, 1, 10) AS mod_date
FROM action_log_with_mod_stamp;
```

### 원포인트
- 하루 집계 범위를 모든 리포트에 적용한다면
  - **로그 수집**과 **데이터 집계**에 관련한 배치 처리 시간을
  - **집계 범위**에 맞게 변경해야 함
- 날짜를 기반으로 무언가를 **집계/변경**하는 경우
  - 원본 데이터를 꼭 따로 **백업** 한 뒤 가공해야 함
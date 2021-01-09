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
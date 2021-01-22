# 19. 데이터 중복 검출하기
- RDB의 경우 적절하게 **유니크 키**를 설정했을 경우
  - 키가 중복되면, 자동으로 **오류**가 발생하여, **데이터의 무결성**이 보장됨
- 하지만 `Hive`, `BigQuery`와 같은 `RDB`가 아닌 데이터베이스에서
  - **데이터 중복**을 사전에 확인하는 기능은 없음
- 마스터 데이터 또는 로그 데이터의 **중복**을 확인하고,
  - **중복 데이터를 제외**하는 법

## 1. 마스터 데이터의 중복 검출하기
- 개념
  - SQL : `COUNT` 함수, `COUNT(DISTINCT ~)`, `HAVING` 구문, `COUNT` 윈도 함수
- 마스터 데이터에 **중복**이 존재하는 경우,
  - 마스터 데이터를 **로그 데이터**와 결합하면
  - 로그가 여러 레코드로 카운팅 되어 **잘못된 분석 결과**를 만들 수 있음
- 따라서 **중복**이 없도록 해야 함

### 키가 중복되는 데이터의 존재 확인하기
- 마스터 데이터의 레코드에 **중복**이 발생했다면 여러가지 이유가 있을 수 있음
  - 데이터를 로드할 때, 실수로 **여러번 로드**되어, 같은 데이터를 가진 레코드가 중복 생성
  - 마스터 데이터의 값을 **갱신**할 때 문제가 발생하여, 오래된 데이터와 새로운 데이터가 서로 다른 레코드로 분리된 경우
  - 운용상의 실수로 같은 ID를 다른 데이터에 재사용한 경우
- 어떤 경우든, **마스터 데이터**에 **중복**이 존재하는지 확인하는 것부터 시작해야 함
- `mst_categories` 테이블을 **샘플 데이터**로 활용
  - `id`, `name`, `stamp`
- 테이블 내부에 **중복**이 발생하는지 확인하려면
  - 테이블 전체의 **레코드 수**와 **유니크한 키의 수**를 세서 비교하면 됨

#### CODE.19.1. 키의 중복을 확인하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    COUNT(1) AS total_num
    , COUNT(DISTINCT id) AS key_num
  FROM mst_categories
  ;
  ```

### 키가 중복되는 레코드 확인하기
- 테이블 내부에 키의 중복이 **존재**하는 것을 알았다면
  - 구체적으로 어떤 키의 **레코드**가 중복되는지 확인해야 함
- 중복되는 ID를 확인하려면 ID 기반으로 
  - `GROUP BY`를 통해 집약하고, 
  - `HAVING` 구문을 사용해 레코드의 수가 1보다 큰 그룹을 찾으면 됨 
- 이러한 ID의 레코드 값을 확인하려면
  - `PostgreSQL`, `BigQuery`에서는 `string_agg` 함수,
  - `Redshift`에서는 `listagg` 함수
  - `Hive`, `SparkSQL`에서는 `collect_list`, `concat_ws`를 사용하여
  - 중복된 값을 **배열로 집약**할 수 있음

#### CODE.19.2. 키가 중복되는 레코드의 값 확인하기
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    id
    , COUNT(*) AS record_num

    -- 데이터를 배열로 집약하고, 쉼표로 구분된 문자열로 변환하기
    -- PostgreSQL, BigQuery; string_agg
    , string_agg(name, ',') AS name_list
    , string_agg(stamp, ',') AS stamp_list
    -- Redshift; listagg
    , listagg(name, ',') AS name_list
    , listagg(stamp, ',') AS stamp_list
    -- Hive, SparkSQL; collect_list, concat_ws
    , concat_ws(',', collect_list(name)) AS name_list
    , concat_ws(',', collect_list(stamp)) AS stamp_list
  FROM
    mst_categories
  GROUP BY id
  -- 중복된 ID 확인하기
  HAVING COUNT(*) > 1
  ;
  ```

#### CODE.19.3. 윈도 함수를 사용해서 중복된 레코드를 압축하는 쿼리
- `string_agg` 함수처럼 **배열**로 만드는 집약함수를 사용할 수 없는 경우이거나,
  - 원래 레코드 형식을 그대로 출력하고 싶은 경우 사용
- **윈도 함수**를 사용하여, ID 중복을 세고,
  - 해당 값을 기반으로 중복을 찾기
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  mst_categories_with_key_num AS (
    SELECT
      *
      -- id 중복 세기
      , COUNT(1) OVER(PARTITION BY id) AS key_num
    FROM
      mst_categories
  )
  SELECT
    *
  FROM
    mst_categories_with_key_num
  WHERE
    key_num > 1 -- ID가 중복되는 경우 확인
  ;
  ```

### 정리
- 마스터 데이터의 ID가 중복되는 경우는 기본적으로
  - 어떤 **실수** 또는 **오류**가 원인일 수 있음
  - 이럴때는 일단 마스터 데이터를 **클렌징** 하기
- 여러번의 데이터 로드가 원인이라면,
  - **데이터 로드 흐름 수정** 또는 매번 데이터를 지우고, 새로 저장하는 방법을 사용하여
    - 여러번 데이터를 로드해도 **같은 실행 결과**가 보증되게 만들기
- 마스터 데이터가 도중에 **갱신**되어, 새로운 데이터와 오래된 데이터가 중복된 경우
  - 새로운 데이터만 남기거나, `timestamp`를 포함하여 **유니크 키**를 구성하는 방법도 고려하기

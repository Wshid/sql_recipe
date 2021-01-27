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

## 2. 로그 중복 검출하기
- 개념
  - SQL : `GROUP BY` 함수, `MIN` 함수, `ROW_NUMBER` 함수, `LAG` 함수
- 로그 데이터는 `정상적으로 저장된 데이터도 중복되는 경우` 존재
  - 사용자가 버튼을 2회 연속 클릭
  - 페이지 새로고침
- 샘플 데이터
  - `dup_action_log` 테이블
  - `session`, `user_id`, `action`, `products`, `action_time`

### 중복 데이터 확인하기
- **19.1**과 동일하게, 사용자와 상품의 조합을 만들고
  - 그를 기반으로 중복 레코드 확인

#### CODE.19.4. 사용자와 상품의 조합에 대한 중복을 확인하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    user_id
    , products
    -- 데이터를 배열로 집약하고, 쉼표로 구분된 문자열로 변환
    -- PostgreSQL, BigQuery의 경우는 string_agg 사용하기
    , string_agg(session, ',') AS session_list
    , string_agg(stamp, ',') AS stamp_list
    -- Redshift의 경우 listagg 사용하기
    , listagg(session, ',') AS session_list
    , listagg(stamp, ',') AS stamp_list
    -- Hive, SparkSQL의 경우 collect_list, concat_ws 사용
    , concat_ws(',', collect_list(session)) AS session_list
    , concat_ws(',', collect_list(stamp)) AS stamp_list
  FROM
    dup_action_log
  GROUP BY
    user_id, products
  HAVING
    COUNT(*) > 1
  ;
  ```
- 이 때, 중복된 로그가 발생 하더라도
  - session이 다르면서, stamp 차이가 난다면, 정상적으로 별도 액션으로 처리해도 무방
  - session이 같으면서, stamp 차이가 얼마 나지 않는다면, 중복 및 배제 필요

### 중복 데이터 배제하기
- `같은 세션 id`, `같은 상품`일 때, `타임 스탬프가 제일 오래된 데이터 남기기`로 정의
- 중복 배제시에, `GROUP BY`로 집약하고,
  - `MIN(stamp)`으로 가장 오래된 **타임스탬프**만 추출하는 방법 사용

#### CODE.19.5. GROUP BY와 MIN을 사용해 중복을 배제하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    session
    , user_id
    , action
    , products
    , MIN(stamp) AS stamp
  FROM
    dup_action_log
  GROUP BY
    session, user_id, action, products
  ;
  ```
- 타임스탬프 이외의 컬럼도 활용해 **중복 제거**가 필요하므로
  - `MIN`함수나 `MAX`함수 등의 단순 집약 함수를 적용할 수 없는 경우 이 방법 사용 불가
- 더 일반적인 방법으로는
  - 원래 데이터에 `ROW_NUMBER` 윈도 함수를 사용하여, 중복 데이터에 **순번**을 부여하고
  - 부여된 순번을 사용하여 **중복 레코드 중 하나**만 남기게 만드는 방법 존재

#### CODE.19.6. ROW_NUMBER를 사용해 중복을 배제하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  dup_action_log_with_order_num AS (
    SELECT
      *
      -- 중복된 데이터에 순번 붙이기
      , ROW_NUMBER()
          OVER(
            PARTITION BY session, user_id, action, products
            ORDER BY stamp
          ) AS order_num
    FROM
      dup_action_log
  )
  SELECT
    session
    , user_id
    , action
    , products
    , stamp
  FROM
    dup_action_log_with_order_num
  WHERE
    order_num = 1 -- 순번이 1인 데이터(중복된 것 중에서 가장 앞의 것)만 남기기
  ;
  ```
- 이번 샘플 로그는 `session_id`를 저장하므로
  - 같은 사용자와 상품의 조합이라도, **세션이 다를경우 다른 로그**
- 하지만 `session_id`등을 사용할 수 없는 경우 존재
  - 이 경우, **타임 스탬프 간격 확인**

#### CODE.19.7. 이전 액션으로부터의 경과 시간을 계산하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  dup_action_log_with_lag_seconds AS (
    SELECT
      user_id
      , action
      , products
      , stamp
      -- 같은 사용자와 상품 조합에 대한 이전 액션으로부터의 경과 시간 계산하기
      -- PostgreSQL의 경우 timestamp 자료형으로 변환하고 차이를 구한 뒤
      -- EXTRACT(epoc ~)를 사용해 초 단위로 변경하기
      , EXTRACT(epoch from stamp::timestamp - LAG(stamp::timestamp)
          OVER(
            PARTITION BY user_id, action, products
            ORDER BY stamp
          )) AS lag_seconds
      -- Redshift의 경우 datediff 함수를 초 단위로 지정
      , datediff(second, LAG(stamp::timestamp)
          OVER(
            PARTITION BY user_id, action, products
            ORDER BY stamp
          ), stamp::timestamp) AS lag_seconds
      -- BigQuery의 경우 unix_seconds 함수로 초 단위 UNIX 타임으로 변환하고 차이 구하기
      , unix_seconds(timestamp(stamp)) - LAG(unix_seconds(timestamp(stamp)))
          OVER(
            PARTITION BY user_id, action, products
            ORDER BY stamp
      -- SparkSQL의 경우 다음과 같이 프레임 지정 추가
          ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING
          ) AS lag_seconds
    FROM
      dup_action_log
  )
  SELECT
    *
  FROM
    dup_action_log_with_lag_seconds
  ORDER BY
    stamp;
  ```
- `lag_seconds`
  - 중복 되지 않은 레코드에서는 `NULL`
  - `사용자 ID`와 `상품 ID`가 중복되는 레코드의 경우
    - 타임스탬프를 기반으로 이전 액션에서의 경과 시간 계산
- 산출한 경과 시간이 일정 시간보다 적을 경우 **중복으로 취급**하여 배제

#### CODE.19.8. 30분 이내의 같은 액션을 중복으로 보고 배제하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  dup_action_log_with_lag_seconds AS (
    -- CODE.19.7
  )
  SELECT
    user_id
    , action
    , products
    , stamp
  FROM
    dup_action_log_with_lag_seconds
  WHERE
    (lag_seconds IS NULL OR lag_seconds >= 30 * 60)
  ORDER BY
    stamp
  ;
  ```
- `session_id`를 사용하지 않아도, **타임스탬프**를 활용하면
  - 일정 시간 이내의 로그를 중복으로 취급 및 배제 가능

### 정리
- 로그 데이터에 **중복**이 포함된 상태로 리포트를 작성하면
  - 실제와 다른 값으로 리포트가 만들어져 **잘못된 판단**을 할 수 있음
- 따라서 분석하는 데이터를 잘 확인하고
  - 어떤 방침으로 이러한 **중복 제거**를 진행할지 검토해야 함
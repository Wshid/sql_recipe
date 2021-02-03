# 20. 여러 개의 데이터셋 비교하기
- 집계 작업에는
  - 여러 개의 **데이터셋**을 비교하여, 어떤 판단을 내리는 경우가 많음
- 여러 개의 데이터 셋을 비교하여
  - **데이터의 변화한 부분**을 확인하거나
  - 어떤 차이가 있는지 확인할 때 사용할 수 있는 SQL 소개

### 데이터의 차이를 추출하는 경우
- **같은 성질의 데이터** 또는 같은 SQL로 만들어진 **다른 기간의 집계 결과**를 비교해서
  - 추가/변경이 없는지, 삭제/결손이 없는지 확인하는 상황
- 대량의 마스터 데이터에서 변경점을 확인하기엔 불가능함
  - SQL을 사용해야 하는 경우

### 데이터의 순위를 비교하는 경우
- 웹 사이트 내 `인기 기사 순위`또는 `적절한 검색 조건`을 출력하는 모듈
  - 인기 기사 순위를 화면에 출력해도, 변화가 거의 없다면
  - 자주 사용하는 사용자에게는 흥미가 떨어지는 정보
    - `순위 집계 기간`과 `조건`을 적절하게 변경해야 함
- **집계 기간** 또는 **출력 조건**을 변경했을 때는
  - **과거 로직**과 비교하여 어떤 차이가 있었는지 등을 확실하게 확인해야 함
- 이런 것들을 **수치화**하지 않고, 눈으로 대충 확인한다면
  - 모호한 판단만이 가능
- 따라서 **로직의 결과**로 출력된 순위의 유사도를 **수치화**하여
  - 새로운 로직과 과거 로직 변화를 정량적으로 설명해야 함

## 1. 데이터의 차이 추출하기
- 개념
  - SQL : `LEFT OUTER JOIN`, `RIGHT OUTER JOIN`, `FULL OUTER JOIN`, `IS DISTINCT FROM` 연산자
- 마스터 데이터를 사용해서, 특정 데이터를 분석할 때는
  - 분석 시점에 따라 **마스터 데이터**의 내용이 달라지므로
  - **시점**에 따라 분석 결과가 달라질 수 있음
- 이러한 때 분석 결과가 왜 달라지는지 설명하려면
  - 어떤 부분이 **변경**되었는지 확인해야 함
- 이번 절에서는
  - **다른 시점**의 마스터 데이터를 사용해, 데이터의 차이를 계산하는 법
- 샘플 데이터
  - 두 개의 마스터 테이블
  - `2016.12.01`, `2017.01.01` 시점의 상품 마스터 테이블
  - `product_id`, `name`, `price`, `updated_at`

### 추가된 마스터 데이터 추출하기
- 두 개의 마스터 테이블에서
  - 한쪽에만 존재하는 레코드를 추출할 때는 `OUTER JOIN`을 사용
- 새로운 마스터 테이블에만 존재하는 레코드를 추출할 때는
  - **새로운 테이블**을 기준으로 **오래된 테이블**을 `LEFT OUTER JOIN`하고
  - 오래된 테이블의 컬럼이 `NULL`인 레코드를 추출하면 됨

#### CODE.20.1. 추가된 마스터 데이터를 추출하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    new_mst.*
  FROM
    mst_products_20170101 AS new mst
    LEFT OUTER JOIN
      mst_products_20161201 AS old_mst
      ON
        new_mst.product_id = old_mst.product_id
  WHERE
    old_mst.product_id IS NULL
  ```

### 제거된 마스터 테이블 추출하기
- 제거된 마스터 데이터를 추출하는 쿼리
- 이전 예시에서 `LEFT OUTER JOIN`의 순서를 반대로 돌리거나,
  - 테이블 순서는 그대로 유지하되, `RIGHT_OUTER_JOIN`을 사용

#### CODE.20.2. 제거된 마스터 데이터를 추출하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    old_mst.*
  FROM
    mst_products_20170101 AS new_mst
    RIGHT OUTER JOIN
      mst_products_20161201 AS old_mst
      ON
        new_mst.product_id = old_mst.product_id
  WHERE
    new_mst.product_id IS NULL
  ```

### 갱신된 마스터 데이터 추출하기
- **오래된 테이블**과 **새로운 테이블**에 모두 존재하고
  - **특정 컬럼의 값이 다른 레코드** 추출
- 내부 결합으로 두개의 마스터 결합
  - 타임 스탬프 값이 다른 레코드를 추출
- 코드 예에서는 값이 변경될 때, 반드시 **타임스탬프 갱신**이 일어난다는 것을 전제로 함

#### CODE.20.3. 변경된 마스터 데이터를 추출하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    new_mst.product_id
    , old_mst.name AS old_name
    , old_mst.price AS old_price
    , new_mst.name AS new_name
    , new_mst.price AS new_price
    , new_mst.updated_at
  FROM
    mst_products_20170101 AS new_mst
    JOIN
      mst_products_20161201 AS old_mst
      ON
        new_mst.product_id = old_mst.product_id
  WHERE
    -- 갱신 시점이 다른 레코드만 추출하기
    new_mst.updated_at <> old_mst.updated_at
  ```

### 변경된 마스터 데이터 모두 추출하기
- `추가`, `제거`, `갱신`된 레코드를 모두 추출
- `LEFT OUTER JOIN`, `RIGHT OUTER JOIN`, `(INNER) JOIN`이 모두 포함되어야 하므로
  - `FULL OUTER JOIN`을 사용
- 차이가 발생한, 변경된 레코드의 경우
  - 양쪽의 **타임스탬프**가 일치하지 않는 것이므로
  - `new_stamp <> old.stamp` 조건만으로 충분하다고 할 수 있지만,
  - `OUTER JOIN`의 경우 한쪽의 데이터가 있을때, 다른쪽이 `NULL`이 되기 때문에
    - `<>`로는 정상 비교가 어려움
- 한쪽에만 `NULL`이 있을때 정상 비교를 하려면
  - `IS DISTINCT FROM`연산자를 사용
  - 단, `IS DISTINCT FROM`연산자가 구현되지 않은 미들웨어의 경우 
  - `COALESCE`함수로 `NULL`을 배제하고 `<>`로 비교해야 함
- 다음 코드는 변경된 레코드를 추출한 뒤
  - `SELECT`구문 내부에서 `added`, `deleted`, `updated`를 판정
  - 오래된 테이블의 값이 `NULL`이라면 **추가**
  - 새로운 테이블의 값이 `NULL`이라면 **삭제**
  - 이외의 경우(타임스탬프가 다른 경우) **갱신**

#### CODE.20.4. 변경된 마스터 데이터를 모두 추출하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    COALESCE(new_mst.product_id, old_mst.product_id) AS product_id
    , COALESCE(new_mst.name, old_mst.name) AS name
    , COALESCE(new_mst.price, old_mst.price) AS price
    , COALESCE(new_mst.updated_at, old_mst.updated_at) AS updated_at
    , CASE
        WHEN old_mst.updated_at IS NULL THEN 'added'
        WHEN new_mst.updated_at IS NULL THEN 'deleted'
        WHEN new_mst.updated_at <> old_mst.updated_at THEN 'updated'
      END AS status
  FROM
    mst_products_20170101 AS new_mst
    FULL OUTER JOIN
      mst_products_20161201 AS old_mst
      ON
        new_mst.product_id = old_mst.product_id
  WHERE
    -- PostgreSQL의 경우 IS DISTINCT FROM 연산자를 사용해 NULL을 포함한 비교 가능
    new_mst.updated_at IS DISTINCT FROM old_mst.updated_at
    -- Redshift, BigQuery의 경우
    -- IS DISTINCT FROM 구문 대신 COALESCE 함수로 NULL을 default 값으로 변환하고 비교하기
    COALESCE(new_mst.updated_at, 'infinity')
    <> COALESCE(old_mst.updated_at, 'infinity')
  ;
  ```

## 2. 두 순위의 유사도 계산하기
- 개념
  - SQL : `RANK` 함수, `SUM` 함수, `COUNT` 함수
  - 분석 : 스피어만 상관계수(Spearman's Rank Correlation Coefficient)
- 데이터 분석에서는 **14.2**에서 소개한 3개의 지표
  - `방문 횟수`, `방문자 수`, `페이지 뷰`
    - 로 **페이지의 순위**를 작성하는 경우가 많음
  - 어떻게 조합해야 가장 적합한 점수 매기기가 가능할지
- 순위들의 **유사도**를 계산하여
  - 어떤 순위가 **효율적**인지 순위를 **정량적**으로 평가하는 방법 소개
- 샘플 데이터 : 웹 페이지 접근 로그
  - stamp, short_session, long_session, path

### 지표들의 순위 작성하기
- `방문 횟수`, `방문자 수`, `페이지 뷰`를 기반으로 **순위**를 작성하는 쿼리

#### CODE.20.5. 3개의 지표를 기반으로 순위를 작성하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQueyr`, `SparkSQL`
  ```sql
  WITH
  path_stat AS (
    -- 경로별 방문 횟수, 방문자 수, 페이지 뷰 계산
    SELECT
      path
      , COUNT(DISTINCT long_session) AS access_users
      , COUNT(DISTINCT short_session) AS access_count
      , COUNT(*) AS page_view
    FROM
      access_log
    GROUP BY
      path
  )
  , path_ranking AS (
    -- 방문 횟수, 방문자 수, 페이지 뷰별로 순위 붙이기
    SELECT 'access_user' AS type, path, RANK() OVER(ORDER BY access_users DESC) AS rank
      FROM path_stat
    UNION ALL
      SELECT 'access_count' AS type, path, RANK() OVER(ORDER BY access_count DESC) AS rank
      FROM path_stat
    UNION ALL
      SELECT 'page_view' AS type, path, RANK() OVER(ORDER BY page_view DESC) AS rank
      FROM path_stat
  )
  SELECT *
  FROM
    path_ranking
  ORDER BY
    type, rank
  ;
  ```
- 위 코드의 결과를 보면 `type` 컬럼에 지표의 종류인
  - `access_count`, `access_user`, `page_view`가 저장되어야 함
- 각 지표별로 `/top`, `/detail`, `/search`라는 **3개의 페이지 순위**를 출력

### 경로별 순위들의 차이 계산하기
- 위 출력 결과를 기반으로 **경로와 순위 조합**을 만들고 **순위 차이** 계산
- **순위 차이**(diff)는 **양쪽 순위 차이의 제곱**으로 정의

#### CODE.20.6. 경로들의 순위 차이를 계산하는 쿼리
- **경로**를 키로 삼아 **순위**를 자기 결합하고
  - 두 지표 기반의 **순위 차이**를 계산
- `PostgreSQL`, `Hive`, `Redshift`, `BigQueyr`, `SparkSQL`
  ```sql
  WITH
  path_stat AS (
    -- CODE.20.5.
  )
  , path_ranking AS (
    -- CODE.20.5.
  )
  , pair_ranking AS (
    SELECT
      r1.path
      , r1.type AS type1
      , r1.rank AS rank1
      , r2.type AS type2
      , r2.rank AS rank2
      -- 순위 차이 계산하기
      , POWER(r1.rank - r2.rank, 2) AS diff
    FROM
      path_ranking AS r1
      JOIN
        path_ranking AS r2
        ON r1.path = r2.path
  )
  SELECT
    *
  FROM
    pair_ranking
  ORDER BY
    type1, type2, rank1
  ;
  ```
- 실행결과 분석 방법
  - 같은 지표를 기반으로 `diff`를 계산시, `0`
- `방문 횟수`와 `방문자 수` 비교에서는
  - `/top`이 각각 **1위**와 **3위**이므로 `(1-3)^2 = 4`

### 스피어만 상관계수 계산하기
- **경로별 순위 차이** 계산이 아닌, **순위 전체의 유사도 계산**
- `Spearman's Rank Correlation Coefficient`로 **순위의 유사도**를 계산
  - 두 개의 순위가 **완전 일치**시 `1.0`, 완전히 일치하지 않는 경우 `-1.0`을 가짐

#### CODE.20.7. 스피어만 상관계수로 순위의 유사도를 계산하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQueyr`, `SparkSQL`
  ```sql
  WITH
  path_stat AS (
    -- CODE.20.5.
  )
  , path_ranking AS (
    -- CODE.20.5.
  )
  , pair_ranking AS (
    -- CODE.20.6.
  )
  SELECT
    type1
    , type2
    -- 스피어만 상관계수 계산
    , 1 - (6.0 * SUM(diff) / (POWER(COUNT(1), 3) - COUNT(1))) AS spearman
  FROM
    ir_ranking
  GROUP BY
    type1, type2
  ORDEr BY
    type1, spearman DESC
  ;
  ```
- 해석 방법
  - 같은 지표의 순위 비교의 경우, 순위가 완전히 일치하기 때문에 `1.0` 리턴
  - **방문자 수**와 **페이지 뷰**의 순위 비교에서는, 순위의 순위 자체가 다르므로
    - 스피어만 상관계수 값이 `-1.0`으로 나타냄

### 정리
- 두 순위의 유사도를 **수치**로 표현할 수 있다면
  - 서로 다른 점수를 기반으로 만든 순위가 어떤 **유사성**을 갖는지 확인 가능
- 이를 활용하면, 순위를 기반으로 **다른 지표**와의 유사성을 구하고
  - 이를 기반으로 **이상적인 순위**를 자동 생성할 수 있음
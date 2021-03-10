# 22. 데이터 마이닝
- 데이터 마이닝
  - 대량의 데이터에서 **특정 패턴**또는 **규칙** 등 유용한 지식을 추출하는 방법을 전반적으로 나타내는 용어
  - **상관 규칙 추출**, **클러스터링**, **상관 분석** 등이 존재
- 데이터 마이닝의 방법 대부분은
  - `재귀 처리`와, `휴리스틱 처리` 필요
  - 단순한 `SQL`로 처리하기 어려움
- 일반적으로 `R`이나 `Python`을 활용하는 경우가 많음
- 이번 절에서는 **상관 규칙 추출 방법**중 하나인 **어소시에이션 분석**을 다루고
  - 이 내용을 SQL로 구현
- 샘플데이터
  - 구매상세로그(purchase_detail_log)
  - stamp, session, purchase_id, product_id

## 1. 어소시에이션 분석
- 개념
  - SQL : `COUNT` 윈도 함수
  - 분석 : 지지도, 확신도, 리프트
- **상관 규칙 추출**의 대표적인 방법
- 예시
  - `상품 A를 구매한 사람의 60%는 상품 B도 구매 했다` 등의 상관 규칙을 데이터에서 찾아내는 것
- 상관 규칙이란
  - `상품 A와 상품 B가 동시에 구매되는 경향이 있다`가 아닌
  - `상품 A를 구매했다면, 상품 B도 구매한다`는
    - **시간척 차이**와 **인과 관계**를 가지는 규칙
  - `상품 A를 구매했다면, 상품 B도 구매한다` => true,
    - `상품 B를 구매했다면, 상품 A도 구매한다` => ? (참으로 성립하지 않을 수 있음)
- 본래 어소시에이션 분석은
  - **상품 조합**(A, B, C, ...)를 구매한 사람의 `n%`는
    - **상품 조합**(X, Y, Z, ...)도 구매한다
  - 위와 같이 **상품 조합**과 관련한 상관 규칙도 발견할 때 활용할 수 있는 **범용적인 알고리즘**
- 하지만, 범용적인 알고리즘을 만들기 위해서는
  - **재귀적인 반복 처리**등을 포함한 많은 처리가 필요,
  - 책에서는 **2개의 상품 간의 발생하는 상관 규칙**으로 단순화

### 어소시에이션 분석에 사용되는 지표
- 지지도(Support)
- 확신도/신뢰도(Confidence)
- 리프트(Lift)

#### 지지도(Support)
- 상관 규칙이 어느 정도의 **확률**로 발생하는지 나타내는 값
- 예시
  - 100개의 구매 로그에서 `상품 X와 상품 Y를 같이 구매한 로그`가 20개 있다면,
  - 이 규칙에 대한 지지도는 `20%`
    - 이때, 구매로그는 동시에 구매한 여러 개의 상품이 하나의 로그에 있다고 가정

#### 확신도/신뢰도(Confidence)
- 어떤 결과가, 어느정도의 확률로 발생하는지를 의미
- 예시
  - 100개의 구매 로그에,
    - `상품 X 구매 레코드 50개`
    - `상품 Y 구매 레코드 20개`
    - 확신도는 `20/50 = 40%`

#### 리프트(Lift)
- 어떤 조건을 만족하는 경우의 확률(`확신도`)를
  - 사전 조건 없이 `해당 결과가 일어날 확률`로 나눈 값
- 예시
  - 100개의 구매로그에
    - `상품 X : 50개`
    - `상품 X & 상품 Y : 20개`
    - `상품 Y : 20개`
    - 확신도 : `20/50 = 40%`
    - 상품 Y 구매 확률 : `20/100 = 20%`
    - 리프트 : `40% / 20% = 2.0`
  - 상품 X를 구매한 경우, 상품 Y를 구매할 확률이 2배라는 의미
- 보통 리프트 값이 `1.0`이상이면 좋은 규칙

### 두 상품의 연관성을 어소시에이션 분석으로 찾기
- 어소시에이션 분석으로 지표를 계산하여, 두 상품의 연관성 찾기
- `상품 A`와 `상품 B`의 어소시에이션 분석에 필요한 정보
  - 구매로그 총 수
  - 상품 A의 구매 수
  - 상품 B의 구매 수
  - 상품 A & 상품 B의 동시 구매 수

#### CODE.22.1. 구매 로그 수와 상품별 구매 수를 세는 쿼리
- `구매 로그 총 수`, `상품 X의 구매 수`를 각각 집계하여, 모든 레코드에 전개
- 각 상품 ID를 사용하여 아래 값을 집계
  - 구매 로그 총 수(purchase_count)
  - 상품 구매 수(product_count)
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  purchase_id_count AS (
    -- 구매 상세 로그에서 유니크한 구매 로그 수 계산하기
    SELECT COUNT(DISTINCT purchase_id) AS purchase_count
    FROM purchase_detail_log
  )
  , purchase_detail_log_with_counts AS (
    SELECT
      d.purchase_id
      , p.purchase_count
      , d.product_id
      -- 상품별 구매 수 계산하기
      , COUNT(1) OVER(PARTITION BY d.product_id) AS product_count)
    FROM
      purchase_detail_log AS d
      CROSS JOIN
        -- 구매 로그 수를 모든 레코드 수와 결합하기
        purchase_id_count AS p
  )
  SELECT
    *
  FROM
    purchase_detail_log_with_counts
  ORDER BY
    product_id, purchase_id
  ;
  ```

#### CODE.22.2. 상품 조합별로 구매 수를 세는 쿼리
- 위 결과에 `구매한 상품 페어`를 생성하고, 조합별로 `구매 수`를 세는 쿼리
- 동시에 구매한 상품 페어를 생성하기 위해
  - 구매 ID에서 **자기 결합**을 한 뒤, 같은 상품 페어는 제외
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  purchase_id_count AS (
    -- CODE.22.1.
  )
  , purchase_detail_log_with_counts AS (
    -- CODE.22.1.
  )
  , product_pair_with_stat AS (
    SELECT
      l1.product_id AS p1
      , l2.product_id AS p2
      , l1.product_count AS p1_count
      , l2.product_count AS p2_count
      , COUNT(1) AS p1_p2_count
      , l1.purchase_count AS purchase_count
    FROM
      purchase_detail_log_with_counts AS l1
      JOIN
        purchase_detail_log_with_counts AS l2
        ON l1.purhcase_id = l2.purchase_id
    WHERE
      -- 같은 상품 조합 제외하기
      l1.product_id <> l2.product_id
    GROUP BY
      l1.product_id
      , l2.product_id
      , l1.product_count
      , l2.product_count
      , l1.purchase_count
  )
  SELECT
    *
  FROM
  product_pair_with_stat
  ORDER BY
    p1, p2
  ;
  ```
- 어소시에이션 분석에 필요한 다음 지표를 계산함
  - 구매 로그 총 수, 상품 A의 구매 수, 상품 B의 구매 수, 상품 A와 B 동시 구매수

#### CODE.22.3. 지지도, 확산도, 리프트를 계산하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  purchase_id_count AS (
    -- CODE.22.1.
  )
  , purchase_detail_log_with_counts AS (
    -- CODE.22.1.
  )
  , product_pair_with_stat AS (
    -- CODE.22.2.
  )
  SELECT
    p1
    , p2
    , 100.0 * p1_p2_count / purchase_count AS support
    , 100.0 * p1_p2_count / p1_count AS confidence
    , (100.0 * p1_p2_count / p1_count)
      / (100.0 * p2_count / purchase_count) AS lift
  FROM
    product_pair_with_stat
  ORDER BY
    p1, p2
  ;
  ```

### 정리
- 이번 절의 **어소시에이션 분석**은
  - 두 상품의 **상관 규칙**을 주목한 한정적인 분석
- 이 쿼리로 **상관 규칙 추출**을 이해했다면,
  - 다른 분석 방법도 찾아 도전해볼 것
# 24. 점수 계산하기
- 어떤 값을 기반으로 **정렬한 결과** -> **순위**
- **순위**는 다양한 분석 보고서에 활용하며, 검색 결과 정렬과 **상품 추천**에도 사용
- 여러 지표들을 조합하여,
  - 범위를 변화 시켜 순위를 구할때 활용할 수 있는 다양한 계산 방법 소개

## 1. 여러 값을 균형있게 조합해서 점수 계산하기
- 개념
  - SQL : `SUM(CASE~)`, `ROW_NUMBER` 함수
  - 분석 : 산술 평균, 기하 평균, 조화 평균, 가중 평균
- 데이터 분석시에는 **여러 지표**를 비교하여 종합적으로 판단 필요
  - **재현율**과 **적합률**의 균형 잡기
  - `CTR`, `CTR`의 관계
  - `지지도`와 `확산도` 등
- 여러 값을 **조합**한 뒤, **집약**해서 쉽게 비교, 검토하는 방법 소개
  - **재현율**과 **적합률**을 조합하고
  - 이를 기반으로 검색 엔진의 **정밀도**를 정량적으로 평가
- 데이터
  - 세로로 저장한 데이터
    - `path`, `recall`, `precision`
  - 가로로 저장한 데이터 
    - `path`, `index`, `value`

### 세 종류의 평균
- 여러 값을 집약하는 **가장 대표적인 방법**
- **산술 평균**
  - 각각의 모두 값을 더하여, 전체 개수로 나누기
  - 가장 기본적인 평균
- **기하 평균**
  - 각각의 값을 곱한 뒤, 개수만큼 제곱근을 한 값
  - `n제곱근`을 걸기 때문에, 실수값을 얻기 위해, 각 값은 양수여야 함
  - **성장률**과 **이율**등을 구할 때 사용
  - 여러 지표를 여러번 **곱해야하는 경우**, 기하평균이 적합함
- **조화 평균**
  - 각 값의 **역수의 산술평균**을 구하고, 다시 **역수**를 취한 값
  - 역수를 취해야 하기 때문에, 각 값이 `0`이 되면 안됨
  - **비율을 나타내는 값의 평균**을 계산할 때 유용하게 활용할 수 있음
  - **평균 속도 계산**에서 많이 사용

### 평균 계산하기
- 세로 데이터의 평균을 구하는 쿼리
- **기하 평균**과 **조화 평균**을 동시에 구할 수 있도록
  - `WHERE`구문에서 **재현율**과 **적합률**이 `0`이 아니고, 곱했을 때 `0`보다 큰 것만 추출하도록 함
- **기하 평균** 계산시에 `SQRT`를 사용하여 제곱근을 구해도 되지만,
  - 3개의 값 이상의 평균에서도 대응할 수 있도록 `POWER`함수에 매개변수 `1/2`를 넣어 계산

#### CODE.24.1. 세로 데이터의 평균을 구하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    *
    -- 산술 평균
    , (recall + precision) / 2 AS arithmetic_mean
    -- 기하 평균
    , POWER(recall * precision, 1.0 / 2) AS geometric_mean
    -- 조화 평균
    , 2.0 / ((1.0 / recall) + (1.0 / precision)) AS harmonic_mean
  FROM
    search_evaluation_by_col
    -- 값이 0보다 큰 것만으로 한정하기
  WHERE recall * precision > 0
  ORDER BY path
  ;
  ```
- **재현율**과 **적합률**이 모두 `50.0`으로 같은 값이라면
  - **산술 평균**도 모두 `50.0`
- 두 값이 다를 경우는 반드시 `산술 평균 > 기하 평균 > 조화 평균`순
- 가로 데이터의 산술 평균, 기하 평균, 조화 평균을 구하려면
  - 값이 `0`이 되는 `poath`를 제외할 수 있게 `WHERE` 구문으로 `value > 0`으로 한정
  - `HAVING` 구문으로, 값의 결손이 없는 `path`로 한정
- **산술 평균**은 일반적인 집약 함수인 `AVG` 사용
  - **조화 평균**도 `AVG`와 **역수 계산**만 유의하여 사용
- **기하 평균**은 별도의 함수를 제공하지 않음
  - `log`를 활용하여 **값의 곱셈 -> 로그 덧셈**으로 변환 
  ```bash
  # 2와 8의 기하 평균
  b(log_b(2) + log_b(8))/2
  ```

#### CODE.24.2. 가로 기반 데이터의 평균을 계산하는 쿼리 
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    path
    -- 산술 평균
    , AVG(value) AS arithmetic_mean
    -- 기하 평균(대수의 산술 평균)
    -- PostgreSQL, Redshift, 상용 로그로 log함수 사용하기
    , POWER(10, AVG(log(value))) AS geometric_mean
    -- Hive, BigQuery, SparkSQL, 상용 로그로 log10 함수 사용
    , POWER(10, AVG(log10(value))) AS geometric_mean
    -- 조화 평균
    , 1.0 / (AVG(1.0 / value)) AS harmonic_mean
  FROM
    search_evaluation_by_row
    -- 값이 0보다 큰 것만으로 한정
  WHERE value > 0
  GROUP BY path
  -- 빠진 데이터가 없게 path로 한정
  HAVING COUNT(*) = 2
  GROUP BY path
  ;
  ```
- **재현율**과 **적합률**의 조화 평균은
  - 두 값을 **통합적으로 평가**할 때 자주 사용
  - `F 척도`(f-measure)또는 `F1 스코어`라는 별도의 이름으로 지칭

### 가중 평균
- **재현율**과 **적합률**에 가중치의 비율을 다르게 하기
  - 검색 결과의 포괄성을 고려하면서도,
  - 검색 결과 상위에 있는 요소들의 타당성을 중요시하여
  - 검색 엔진을 평가하기위해
    - `재현율 : 적합률 = 3:7`
- 가중치가 부여된 평균 계산 방법
  - `산술 평균` = 각 값에 가중치 곱, 가중치의 합으로 나누기
  - `기하 평균` = 가중치만큼 제곱, 합계만큼 제곱근
  - `조화 평균` = 각 값의 역수에 가중치 곱하여 산술 평균, 이후 역수
- 가중치의 합이 `1.0`이 되면
  - 가중치의 합으로 나누거나, 제곱근 구하는 등의 계산이 **필요 없음**
- 가중치의 합계를 `1.0`으로 설정할 수 있게,
  - 가중치 : `재현율 = 0.3, 적합률 = 0.7`

#### CODE.24.3. 세로 기반 데이터의 가중 평균을 계산하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    *
    -- 가중치가 추가된 산술 평균
    , 0.3 * recall + 0.7 * preicision AS weighted_a_mean
    -- 가중치가 추가된 기하 평균
    , POWER(recall, 0.3) * POWER(precision, 0.7) AS weighted_g_mean
    -- 가중치가 추가된 조화 평균
    , 1.0 / ((0.3) / recall) + (0.7 / precision)) AS weighted_h_mean
  FROM
    search_evaluation_by_col
  -- 값이 0보다 큰 것만으로 한정
  WHERE recall * precision > 0
  ORDER BY path
  ;
  ```
- 가로 기반 테이블 데이터의 가중 평균 계산
  - 무게를 `SELECT ... CASE`로 나타낼 수 있지만, 쿼리가 길어지므로,
    - 임시 테이블`(weight)`을 사용하여, **무게 마스터 테이블**을 생성하여
    - 이를 결합하는 방식으로 사용
- `AVG`를 사용하면, **레코드 행 수로 나눈 평균**을 구하므로
  - 대신 `SUM`함수를 사용

#### CODE.24.4. 가로 기반 테이블의 가중 평균을 계산하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  weights AS (
    -- 가중치 마스터 테이블(가중치의 합계가 1.0이 되도록 설정)
              SELECT 'recall'   AS index, 0.3 AS weight
    UNION ALL SELECT 'precision' AS index, 0.7 AS weight
  )
  SELECT
    e.path
    -- 가중치가 추가된 산술 평균
    , SUM(w.weight * e.value) AS weighted_a_mean
    -- 가중치가 추가된 기하 평균
    -- PosgreSQL, Redshift, log 사용
    , POWER(10, SUM(w.weight * log(e.value))) AS weighted_g_mean
    -- Hive, BigQuery, SparkSQL, log10 함수 사용
    , POWER(10, SUM(w.weight * log10(e.value))) AS weighted_g_mean

    -- 가중치가 추가된 조화 평균
    , 1.0 / (SUM(w.weight / e.value)) AS weighted_h_mean
  FROM
    search_evaluation_by_row AS e
    JOIN
      weights AS w
      ON e.index = w.index
    -- 값이 0보다 큰 것만으로 한정하기
  WHERE e.value > 0
  GROUP BY e.path
  -- 빠진 데이터가 없도록 path로 한정
  HAVING COUNT(*) = 2
  ORDER BY e.path
  ;
  ```

### 정리
- `3개 이상`의 값에 대해서도 같은 쿼리로 평균을 구할 수 있음
- `3`개의 평균을 사용하면, 다양한 지표의 데이터 활용에 도움이 됨

## 2. 값의 범위가 다른 지표를 정규화해서 비교 가능한 상태로 만들기
- 개념
  - SQL : `MIN` 윈도 함수, `MAX` 윈도 함수
  - 분석 : `Min-Max` 정규화, 시그모이드 함수
- 값의 범위(스케일)이 다른 지표를 결합할 때는
  - 정규화라는 **전처리**를 해주어야 함
- 샘플
  - 상품 열람 수, 구매 수 데이터
    - `user_id, product, view_count, purchase_count`
  - 각각의 값을 정규화 해야함
  - 상품 구매수는 `0 | 1`이지만, 열람 수는 `21, 49`와 같은 비교적 큰 수
- `열람 수`와 `구매 수`를 모두 `0 ~ 1`의 실수로 변환해야 함

### Min-Max 정규화
- 서로 다른 값의 **폭**을 가지는 각 지표를 `0~1`의 스케일로 정규화 하는 방법
- 각 지표의 `min`과 `max`를 구하고, 변환후의 최소 최대를 `0.0 ~ 1.0`이 되게 반영
- 각각의 지표를 `min`으로 감산한 뒤,
  - `max - min`을 나누는 방식
- `열람 수`와 `구매 수`에 `Min-Max` 정규화를 적용하는 쿼리
- 레코드 전체의 `min`, `max`를 구하기 위해 `min/max 윈도 함수 사용`

#### CODE.24.5. 열람 수와 구매 수에 Min-Max 정규화를 적용하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    user_id
    , product
    , view_count AS v_count
    , purchase_count AS p_count
    , 1.0 * (view_count - MIN(view_count) OVER())
      -- PostgreSQL, Redshift, BigQuery, SparkSQL의 경우 `NULLIF`로 0으로 나누는 것 피하기
      / NULLIF((MAX(view_count) OVER() - MIN(view_count) OVER()), 0)
      -- Hive의 경우 NULLIF 대신 CASE 사용
      / (CASE
          WHEN MAX(view_count) OVER() - MIN(view_count) OVER() = 0 THEN NULL
          ELSE MAX(view_count) OVER() - MIN(view_count) OVER()
          END
        )
      AS norm_view_count
    , 1.0 * (purchase_count - MIN(purchase_count) OVER())
      -- PostgreSQL, Redshift, BigQuery, SparkSQL의 경우 `NULLIF`로 0으로 나누는 것 피하기
      / NULLIF((MAX(purchase_count) OVER() - MIN(purchase_count) OVER()), 0)
      -- Hive의 경우 NULLIF 대신 CASE 사용
      / (CASE
          WHEN MAX(purchase_count) OVER() - MIN(purchase_count) OVER() = 0 THEN NULL
          ELSE MAX(purchase_count) OVER() - MIN(purchase_count) OVER()
          END
        )
      AS norm_p_count
  FROM action_counts
  ORDER BY user_id, product;
  ```
- `norm_v_count` : 정규화 후의 열람 수
- `norm_p_count` : 정규화 후의 구매
  - 구매 수의 경우, 최솟값과 최대값이 기존 `0, 1`이므로, 값이 변화되지 않음

### 시그모이드 함수로 변환하기
- `Min-Max 정규화`의 문제
  - **모집단**의 변화에 따라, **정규화 후**의 값이 모두 변경 됨 
    - 최댓값 또는 최소값이 변하는 경우
  - 정규화 계산시, 데이터의 수에 따라 계산 비용이 의존적
- 이 문제를 극복하기 위해 **결과값이 0~1**인 함수를 사용하여, 데이터를 변환하는 정규화 방법 사용
- `0`이상의 값을 `0~1`의 범위로 변환해주는
  - `S`자 곡선을 그리는 함수
- 그 중에서도 `y=1/(1+e^(-ax))` 함수가 많이 사용됨
  - 이 때, `a=1`인 함수를 **표준 시그모이드 함수**로 부름
- 이러한 시그모이드 함수를 
  - `x=0`일 때, `y=0`
  - `x=unlimited`, `y=1`이 되도록 변경
  - 매개변수 `a`(gain)에 적당한 값을 넣어 지표 조정
- 시그모디으 함수를 사용해 변환하는 쿼리
  - 열람수에는 적당하게 `gain 0, 1`
  - 구매수에는 적당하게 `gain 10`을 적용
- `gain`을 크게 적용할경우,
  - **입력 값**이 조금 커지는 것만으로도, 변환후의 값이 `1`에 가까워짐

#### CODE.24.6. 시그모이드 함수를 사용해 변환하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    user_id
    , product
    , view_count AS v_count
    , purchase_count AS p_count
    -- gain을 0.1로 사용한 sigmoid 함수
    , 2.0 / (1 + exp(-0.1 * view_count)) - 1.0 AS sigm_v_count
    -- gain을 10으로 사용한 sigmoid 함수
    , 2.0 / (1 + exp(-10 * purchase_count)) - 1.0 AS sigm_p_count
  FROM action_counts
  ORDER BY user_id, product;
  ```
- `sigmoid`를 적용한
  - **열람 수**
    - `10 -> 0.4621`
    - `49 -> 0.9852`
  - **구매 수**
    - gain이 `10`이므로, 기존 값(`0 ~ 1`)과 큰 차이가 없음
- `Min-Max 정규화`와 다르게 값이 `2`배가 된다고, 변환된 값이 `2`배가 되는 것이 아님
  - sigmoid 자체가 곡선이므로,
  - `0`에 가까울 수록 **변환 후**의 값에 미치는 **영향이 큼**
- 위 특징은 **열람 수**를 다룰 때 매우 좋음
  - 열람 수가 `1`회인 상품과, `10`회인 상품은 **관심도**의 차이가 매우 크지만
  - 열람 수가 `10001`회인 상품과, `1010`회인 상품은, 크게 관심도 차이가 나지 않음
- 즉, **sigmoid** 함수를 사용해 **비선형 변환**을 하는 것이 더 직관적으로 이해할 수 있음

### 정리
- **값의 범위**(스케일)이 다른 지표를 다루는 여러가지 방법
- 어떤 방법이 적합할지는 **데이터의 특성**에 따라 다름
- 직접 사용해보면서, 어느쪽이 더 효율적일지 확인 필요
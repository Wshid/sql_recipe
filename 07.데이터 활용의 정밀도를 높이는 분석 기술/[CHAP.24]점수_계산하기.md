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

## 3. 각 데이터의 편차값 계산하기
- 개념
  - SQL : `stddev_pop` 함수
  - 분석 : 표준편차, 정규값, 편차값
- 데이터의 **분포**가 **정규 분포**에 가깝다는게 어느정도 확실하다면,
  - **정규값**과 **편차값**을 이용하는 경우가 많음
- 일반적인 **편차값**의 정의와
  - 이를 `SQL`로 구하는 방법

### 표준편차, 정규값, 편차값
#### 표준편차
- 데이터의 **쏠림 상태**을 나타내는 값
  ```bash
  # 모집단의 표준편차를 구할 때
  # SQL : stddev_pop
  표준편차 = v((각 데이터 - 평균)^2을 모두 더한 것 / 데이터의 수)

  # 모집단의 일부를 기반으로 표준편차를 구할 때
  # SQL : sqddev
  표준편차 = v((각 데이터 - 평균)^2를 모두 더한것 / 데이터의 개수 - 1)
  ```
- 표준편차의 `min = 0`
- 표준편차가 커질수록 **데이터에 쏠림**이 존재

#### 정규값
- 평균으로부터 **얼마나 떨어져 있는지**
- 데이터의 **쏠림 정도**를 기반으로 **점수의 가치**를 평가하기 쉽게
  - 데이터를 변화하는 것 -> **정규화**
- 정규화된 데이터를 **정규값**이라고 함
  ```bash
  정규값 = (각각의 데이터 - 평균) / 표준편차
  ```
- 정규값의 평균은 `0`, 표준 편차는 `1`
- 이 특징을 활용하면
  - 서로 다른 데이터, 단위가 다른 데이터(`e.g. CTR, CVR`)도 쉽게 비교 가능

#### 편차값
- **조건**이 다른 데이터를 쉽게 비교하고 싶을 때
- **정규값**을 사용하여 계산
  ```bash
  편차값 = 각 정규화된 데이터 * 10 + 50
  ```
- 편차값의 평균은 `50`, 표준편차는 `10`

#### 편차 계산하기
- 편차값은 `10 * (각각의 값 - 평균 값 / 표준편차 + 50)`으로 계산 가능
- **평균값**과 **표준편차**를 사용해 각 값을 계산하므로
  - **윈도 함수**를 사용해 구해야 함
- 윈도 함수를 사용하여 과목들의 **표준편차**, **기본값/편차값**을 계산하는 쿼리

##### CODE.24.7. 표준편차, 기본값, 편차값을 계산하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `SparkSQL`
  ```sql
  SELECT
    subject
    , name
    , score
    -- 과목별로 표준편차 구하기
    , stddev_pop(score) OVER(PARTITION BY subject) AS stddev_pop
    -- 과목별 평균 점수 구하기
    , AVG(score) OVER(PARTITION BY subject) AS avg_score
    -- 점수별로 기준 점수 구하기
    , (score - AVG(score) OVER(PARTITION BY subject))
      / stddev_pop(score) OVER(PARTITION BY subject) AS std_value
    -- 점수별로 편차값 구하기
    , 10.0 * (score - AVG(score) OVER(PARTITION BY subject))
      / stddev_pop(score) OVER(PARTITION BY subject) + 50 AS deviation
  FROM exam_scores
  ORDER BY subject, name;
  ```
- **BigQuery**의 경우 `stddev_pop`함수를 **윈도 함수**로 사용할 수 없으므로,
  - 다른 테이블에 **표준편차**를 계산해두고, **각 레코드와 결합**

##### CODE.24.8. 표쥰편차를 따로 계산하고, 편차값을 계산하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  exam_stddev_pop AS (
    -- 다른 테이블에서 과목별로 표준편차 구해두기
    SELECT
      subject
      , stddev_pop(score) AS stddev_pop
    FROM exam_scores
    GROUP BY subject
  )
  SELECT
    s.subject
    , s.name
    , s.score
    , d.stddev_pop
    , AVG(s.score) OVER(PARTITION BY s.subject) AS avg_score
    , (s.score - AVG(s.score) OVER(PARTITION BY s.subject)) / d.stddev_pop AS std_value
    , 10.0 * (s.score - AVG(s.score) OVER(PARTITION BY s.subject)) / d.stddev_pop + 50 AS deviation
  FROM
    exam_scores AS s
    JOIN
      exam_stddev_pop AS d
      ON s.subject = d.subject
  ORDER BY s.subject, s.name;
  ```

### 정리
- **기본값**과 **편차값** 등의 지표를 `SQL`로 계산하는 방법은
  - **윈도 함수**의 등장 덕분에, 매우 쉬워짐
- **지표 정의**와 **의미**를 확실하게 확인하여, **의미 있는 데이터 분석**을 할 것

## 4. 거대한 숫자 지표를 직감적으로 이해하기 쉽게 가공하기
- 개념
  - SQL : `log` 함수
  - 분석 : 록 
- 사람이 직감적으로 이해하기 쉬운 값으로 변형하는
  - **비선형적인 함수**로 **로그**를 사용하는 방법
- `사용자 액션 수`, `경과일 수`처럼
  - 값이 크면 클수록 **값 차이**가 큰 영향을 주지 않는경우,
  - **로그**를 사용해 데이터를 변환하면, 이해하기 쉽게 가공 가능
- 샘플
  - `dt`, `user_id`, `product`, `v_count`, `p_count`
    - 사용자별 상품 열람 수(`v_count`)
    - 구매 수(`p_count`)
- **23.2**에서는
  - 모든 기간의 `열람 수 + 구매 수`를 하여, **상품 점수**를 계산
  - 이번절에서는
    - **오래된** 날짜의 액션일수록, **가중치를 적게** 주는 방법 사용
- 사용자별로 **최종 액션일**, **각각의 레코드와 최종 액션일과의 날짜 차이** 계산

### CODE.24.9. 사용자들의 최종 접근일과 각 레코드와의 날짜 차이를 계산하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  action_counts_with_diff_date AS (
    SELECT *
      -- 사용자별로 최종 접근일과 각 레코드의 날짜 차이 계산
      -- PostfreSQL, Redshift의 경우 날짜끼리 빼기 연산 가능
      , MAX(dt::date) OVER(PARTITION BY user_id) AS last_access
      , MAX(dt::date) OVER(PARTITION BY user_id) - dt::date AS diff_date
      -- BigQuery의 경우 date_diff 함수 사용하기
      , MAX(date(timestamp(dt))) OVER(PARTITION BY user_id) AS last_access
      , date_diff(MAX(date(timestamp(dt))) OVER(PARTITION BY user_id), date(timestamp(dt)), date) AS diff_date
      -- Hive, SparkSQL의 경우 datediff 함수 사용
      , MAX(to_date(dt)) OVER(PARTITION BY user_id) AS last_access
      , datediff(MAX(to_date(dt)) OVER(PARTITION BY user_id), to_date(dt)) AS diff_date
    FROM
      action_counts_with_date
  )
  SELECT *
  FROM action_counts_with_diff_date;
  ```
- `최종 액션일`과의 차이가 크면 클수록, 가중치를 낮게 하는 방법
  - 날짜 차이에 **역수** 취하기?
    - 단순하게 역수를 취하게 되면 `2일 전, 1/2`, `30일 전, 1/30`이 되기 때문에, 극단적으로 작아짐
    - 따라서 **날짜 차이**에 **로그**를 취해
      - 날짜가 증가하도록 가중치가 **완만하게 감소**하게 변경
  - 추가로, 날짜 차이가 `0`일 때, 가중치의 최댓값이 `1`이 될 수 있게 하고
  - 추가적인 **매개 변수**를 사용해, **가중치의 변화율**을 조정할 수 있도록 매개변수 a를 둔다
- 수식
  ```
  y = 1/(log2(ax+2)) {0 <= x}
  ```
- 매개변수가 `0`에 가까워질수록, 그래프가 **완만**해지며
- `a=0`이 되어버리면, 가중치가 항상 `1`이 된다
- 매개 변수를 크게 할수록 **가중치의 감소가 눈에 띄게 변화**

### CODE.24.10. 날짜 차이에 따른 가중치를 계산하는 쿼리
```sql
WITH
action_counts_with_diff_date AS (
  -- CODE.24.9.
), action_counts_with_weight AS (
  SELECT *
    -- 날짜 차이에 따른 가중치 계산하기(a = 0.1)
    -- PostgreSQL, Hive, SparkSQL, log(<밑수>, <진수>) 함수 사용
    , 1.0 / log(2, 0.1 * diff_date + 2) AS weight
    -- Redshift의 경우 log는 상용 로그만 존재, `log2`로 나눠주기
    , 1.0 / ( log(CAST(0.1 * diff_date + 2 AS double precision)) / log(2) ) AS weight
    -- BigQuery의 경우, log(<진수>, <밑수>) 함수 사용
    , 1.0 / log(0.1 * diff_date + 2, 2) AS weight
  FROM action_counts_with_diff_date
)
SELECT
  user_id
  , product
  , v_count
  , p_count
  , diff_date
  , weight
FROM action_counts_with_weight
ORDER BY
  user_id, product, diff_date DESC
;
```
- 최종 접근일과의 날짜 차이가 `0`일때, 가중치가 `1.0`으로 가장 크고,
  - 최종 접근일과의 날짜 차이가 커질수록, 가중치가 완만하게 줄어듦
- 지금까지 구한 과정으로 구한 **가중치**를
  - **열람 수**와 **구매 수**에 곱해 점수를 계산
- 날짜별로 **열람 수**와 **구매 수**에 해당 날짜의 **가중치**를 곱한 뒤 더해 점수 계산

### CODE.24.11. 일수차에 따른 중첩을 사용해 열람 수와 구매 수 점수를 계산하는 쿼리
```sql
WITH
action_counts_with_date AS (
  -- CODE.24.9.
)
, action_counts_with_weight AS (
  -- CODE.24.10.
)
, action_scores AS (
  SELECT
    user_id
    , product
    , SUM(v_count) AS v_count
    , SUM(v_count * weight) AS v_score
    , SUM(p_count) AS p_count
    , SUM(p_count * weight) AS p_score
  FROM action_counts_with_weight
  GROUP BY
    user_id, product
)
SELECT *
FROM action_scores
ORDER BY
  user_id, product;
```
- 날짜에 따라 가중치가 적용된 점수 산출 가능
- 같은 구매 수의 상품이더라도, 날짜가 오래되었다면
  - 점수가 낮게 측정

### 정리
- `열람 수`와 `구매 수`처럼 **어떤 것을 세어 집계한 숫자**는
  - 점수르 계산할 때, **로그**를 취해서 값의 변화를 **완만하게** 표현 가능
- 이를 활용하면 사람이 더 **직감적으로** 쉽게 값 변화 인지 가능
- 로그는 수학적 요소지만, **그 활용범위가 넓음**

## 5. 독자적인 점수 계산 방법을 정의해서 순위 작성하기
- 분석 : 가중 평균
- 예시 데이터
  - 월별 상품 매출 수(`monthly_sales`)
  - `year_month, item, amount`
- **순서 생성** 방침은
  - 1년 동안의 계절마다 **주기적으로 팔리는 상품**과
  - **최근 트랜드 상품**이 상위에 오도록 순위를 구성
- 따라서 `1년 전의 매출과`, `최근 1개월`의 매출 값으로 **가중 평균**을 내서 점수를 사용
- `4분기별 상품 매출액과 합계`를 구하는 쿼리
  - 상품별 `2016 1q`, `4q`의 매출액을 사용하여 순위를 구함

### CODE.24.12. 분기별 상품 매출액과 매출 합계를 집계하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  item_sales_per_quarters AS (
    SELECT item
      -- 2016.1q의 상품 매출 모두 더하기
      , SUM(
        CASE WHEN year_month IN ('2016-01', '2016-02', '2016-03') THEN amount ELSE 0 END
      ) AS sales_2016_q1
      -- 2016.4q의 상품 매출 모두 더하기
      , SUM(
        CASE WHEN year_month IN ('2016-10', '2016-11', '2016-12') THEN amount ELSE 0 END
      ) AS sales_2016_q4
    FROM monthly_sales
    GROUP BY item
  )
  SELECT
    item
    -- 2016.1q의 상품 매출
    , sales_2016_q1
    -- 2016.1q의 상품 매출 합계
    , SUM(sales_2016_q1) OVER() AS sum_sales_2016_q1
    -- 2016.4q의 상품 매출
    , sales_2016_q4
    -- 2016.4q의 상품 매출 합계
    , SUM(sales_2016_q4) OVER() AS sum_sales_2016_q4
  FROM item_sales_per_quaters
  ;
  ```
- 위 출력 결과에서 `2016.1q`와 `4q`의 합계 매출액을 확인하면,
  - 서비스 전체의 매출이 상승 경향이 있는 것을 알 수 있음(각각 `254,000`, `550,000`)
- 따라서 1분기 매출액과 4분기 매출액이 같다고 해도, **의미가 다름**
- 다음 코드는 `1q`와 `4q`의 매출액을 정규화하여 점수를 계산하는 쿼리
  - 분모가 다른 2개의 지표를 비교할 수 있도록 `Mix-Max 정규화` 사용

### CODE.24.13. 분기별 상품 매출액을 기반으로 점수를 계산하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  item_sales_per_quarters AS (
    -- CODE.24.12
  )
  , item_scores_per_quarters AS (
    SELECT
      item
      , sales_2016_q1
      , 1.0
        * (sales_2016_q1 - MIN(sales_2016_q1) OVER())
        -- PostgreSQL, Redshift, BigQuery, SparkSQL, NULLIF로 divide 0 회피
        / NULLIF(MAX(sales_2016_q1) OVER() - MIN(sale_2016_q1) OVER(), 0)
        -- Hive, CASE 식 사용
        / (CASE
            WHEN MAX(sales_2016_q1) OVER() - MIN(sales_2016_q1) OVER() = 0 THEN NULL
            ELSE MAX(sales_2016_q1) OVER() - MIN(sales_2016_q1) OVER() 
          END)
        AS score_2016_q1
      , sales_2016_q4
      , 1.0
        * (sales_2016_q4 - MIN(sales_2016_q4) OVER())
        -- PostgreSQL, Redshift, BigQuery, SparkSQL, NULLIF로 divide 0 회피
        / NULLIF(MAX(sales_2016_q4) OVER() - MIN(sales_2016_q4) OVER(), 0)
        -- Hive, CASE 식 사용
        / (CASE
            WHEN MAX(sales_2016_q4) OVER() - MIN(sales_2016_q4) OVER() = THEN NULL
            ELSE MAX(sales_2016_q4) OVER() - MIN(sales_2016_q4) OVER()
          END)
        AS score_2016_q4
    FROM item_sales_per_quarters
  )
  SELECT *
  FROM item_scores_per_quarters
  ;
  ```
- 다음으로, `1q`와 `4q`의 매출을 기반으로 **정규화된 점수**를 계산했다면,
  - `1q`와 `4q`의 **가중 평균**을 구하고, 하나의 점수로 집약하기
- `1q : 4q = 7 : 3`으로 지정하고 가중 평균을 계산할때 다음과 같이 순위 생성

### CODE.24.14. 분기별 상품 점수 가중 평균으로 순위를 생성하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  item_sales_per_quarters AS (
    -- CODE.24.12
  ),
  item_scores_per_quarters AS (
    -- CODE.24.13
  )
  SELECT
    item
    , 0.7 * score_2016_q1 + 0.3 * score_2016_q4 AS score
    , ROW_NUMBER()
        OVER(ORDER BY 0.7 * score_2016_q1 + 0.3 * score_2016_q4 DESC)
      AS rank
  FROM item_scores_per_quarters
  ORDER BY rank
  ;
  ```

### 정리
- **주기 요소가 있는 상품**과 **트렌드 상품**을 균형 있게 조합한 **순위**를 생성할 수 있음
- 추가로 **여러 가지 점수 계산 방식**을 조합하면 다양한 분야에 활용할 수 있음
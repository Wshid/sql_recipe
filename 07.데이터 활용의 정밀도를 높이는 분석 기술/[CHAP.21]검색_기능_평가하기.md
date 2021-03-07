# 21. 검색 기능 평가하기
- 운영하는 서비스에서 **내부 검색 기능**을 제공할 경우
  - 사용자가 어떤 **검색 쿼리**를 입력하고, **어떤 결과**를 얻는지 분석하느 작업이 매우 중요
- 이번 절에서는
  - **검색 관련 행동 로그**와
  - 미리 입력하여 준비한 **평가 전용 데이터**로
    - 검색 기능을 **정량적**으로 평가하고, 개선하는 방법을 소개

### 검색하는 사용자의 행동
- 검색하는 사용자의 **행동 패턴** 정의
  - 사용자가 **특정 검색 쿼리**를 입력하면
  - 사용자는 해당 **검색 결과**를 **출력**하는 화면으로 이동
    - 사용자는 검색 결과로 `원하는 정보`가 나오면, 해당 정보의 **상세 화면**으로 이동
    - 원하는 정보가 없다면, `다시 검색` / `이탈`
- 사용자가 무엇을 검색하고
  - 그 검색 결과에 대해 **어떤 행동**을 취하는지 구분이 가능하다면
  - 검색 기능을 담당하고 있는 **엔지니어**에게 기능 개선 요청이 가능

### 검색 기능 개선 방법
- 검색 키워드의 **흔들림**을 흡수할 수 있도록 **동의어 사전 추가**
- 검색 키워드를 **검색 엔진**이 이해할 수 있게 **사용자 사전 추가**
- 검색 결과가 사용자가 원하는 순서대로 나오도록 **정렬 순서 조정**

#### 동의어 사전 추가
- 사용자가 언제나 상품 이름을 `정확하게 입력하지 않음`
  - 상품의 단축된 이름 또는 별칭을 사용하는 경우 존재
  - 알파벳 이름을 한글로 검색 등
- 검색의 흔들림을 흡수하려면
  - 공식 상품 이름과 동일한 의미를 가질 수 있는 단어를 **동의어 사전**에 추가해야 함

#### 사용자 사전 추가하기
- **검색 엔진**에 따라서는
  - 단어를 **독자적인 알고리즘**으로 분해하고,
  - 이를 기반으로 **인덱스**를 만들어 데이터 검색에 활용
- 위와 같이 작업할 경우
  - **서비스 제공 측**이 의도한대로, 단어가 분해된다는 보장이 없음
- 예를 들어 `위스키`라고 검색했으나,
  - `위`와 `스키`로 분해할 수 있음
- 이런 결과를 막기 위해 `위스키`를 하나의 단어로 **사용자 사전**에 추가

#### 정렬 순서 조정하기
- **검색 엔진**은 조건에 맞은 아이템을 **출력**할 뿐만 아니라
  - **검색 쿼리**와의 연관성을 **수치화**하여
  - 점수가 높은 순서대로 정렬해주는 **순위 기능**을 가짐
- **관련도 점수**를 계산하려면
  - 검색 키워드와 아이템에 포함된 문장의
    - `출현 위치`와 `빈도`
    - `아이템의 갱신 일자`, `접속 수`
      - 등의 다양한 요소를 조합해야 함
- 이처럼 데이터를 조합하여 **최적의 순서**로 결과를 출력하려면
  - 적절한 평가 지표와 방법이 있어야 함

### 검색 기능을 평가하기 위한 데이터
- **21.1 ~ 21.5**에서는 `DATA.21.1`을 사용
  - `검색 결과`와 `상세 페이지 로그 데이터`
- 위 데이터를 활용하여 **사용자 행동 지표** 분석
- 검색 결과 페이지가 많아서, 여러 페이지가 나올 경우
  - 두 번째 이후 페이지를 눌러도, 하나의 레코드만 로그로 저장하게 데이터를 가공
  - 이는, `검색 결과의 두번째 페이지를 열람`하는 것과 **새로운 검색**을 구분하기 위함
- 텍스트 박스에 **검색 키워드**를 입력하여 검색하는
  - Free Keyword Search를 전제로 **리포트**와 **쿼리** 소개
- 카테고리 검색에 활용할 수 있는 리포트도 존재함
- 데이터 컬럼
  - stamp, session, action, keyword, url, referer, result_num

## 1. NoMatch 비율과 키워드 집계하기
- 개념
  - SQL : `SUM(CASE~)`, `AVG(CASE~)`
  - 분석 : NoMatch 비율, NoMatch 키워드
- 사용자가 검색했을 때, 원하는 결과가 나오지 않으면 `부정적인 인상`을 받음
- 사용자가 다시 다른 조건으로 검색하면 좋겠으나, **이탈할 가능성** 존재
- 검색 엔진이 **키워드**를 제대로 파악하지 못해, 흔들림을 흡수하지 못했다면
  - 여러 가지 대책을 통해 결과가 나오도록 변경해야 함

### NoMatch 비율 집계하기
- 이 책에서는 `검색 총 수` 중에서 `검색 결과를 0`으로 리턴하는 검색 결과 비율을
  - `NoMatch 비율`이라고 정의
- 공식
  - `NoMatch 비율 = 검색 결과가 0인 수(NoMatch 수) / 검색 총 수`
- 검색 엔진이 제대로 **키워드**를 파악하지 못하거나,
  - 검색 키워드의 **흔들림**을 흡수하지 못할 가능성을 알아보려면,
  - 검색 결과가 `0`인 비율(`NoMatch 비율`)을 집계하는 것이 좋음

#### CODE.21.1 NoMatch 비율을 집계하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    -- PostgreSQL, Hive, Redshift, SparkSQL의 경우 substring으로 날짜 추출
    substring(stamp, 1, 10) AS dt
    -- PostgreSQL, Hive, BigQuery, SparkSQL의 경우 substr 사용
    substr(stamp, 1, 10) AS dt
    , COUNT(1) AS search_count
    , SUM(CASE WHEN result_num = 0 THEN 1 ELSE 0 END) AS no_match_count
    , AVG(CASE WHEN result_num = 0 THEN 1.0 ELSE 0.0 END) AS no_match_rate
  FROM
    access_log
  WHERE
    action = 'search'
  GROUP BY
    -- PostgreSQL, Redshift, BigQuery
    -- SELECT 구문에서 정의한 별칭을 GROUP BY에서 지정 가능
    dt
    -- PostgreSQL, Hive, Redshift, SparkSQL
    -- SELECT 구문에서 별칭을 지정하기 이전의 식을 GROUP BY로 지정할 수 있음
    substring(stamp, 1, 10)
  ;
  ```

### NoMatch 키워드 집계하기
- **CODE.21.2**는 `NoMatch`가 될 때의 **검색 키워드**를 추출하는 쿼리
- 어떤 키워드가 `0`개의 결과를 내는지 집계한 뒤,
  - **동의어 사전**과 **사용자 사전**의 추가하다고 판단되면, 이를 추가하여 **검색 결과** 개선

#### CODE.21.2. NoMatch 키워드를 집계하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  search_keyword_stat AS (
    -- 검색 키워드 전체 집계 결과
    SELECT
      keyword
      , result_num
      , COUNT(1) AS search_count
      , 100.0 * COUNT(1) / COUNT(1) OVER() AS search_share
    FROM
      access_log
    WHERE
      action = 'search'
    GROUP BY
      keyword, result_num
  )
  -- NoMatch 키워드 집계 결과
  SELECT
    keyword
    , search_count
    , search_share
    , 100.0 * search_count / SUM(search_count) OVER() AS no_match_share
  FROM
    search_keyword_stat
  WHERE
    -- 검색 결과가 0개인 키워드만 추출
    result_num = 0
  ```

### 원포인트
- 이번 절에서는 `Free Keyword Search`를 전제로 설명 했지만
  - 조건을 선택하는 **카테고리 검색**에서도 `NoMatch` 비율이 중요한 지표가 될 수 있음
- 어떤 경우에도 검색 결과가 `0`이 나오지 않도록, 여러가지 대책을 세우기


## 2. 재검색 비율과 키워드 집계하기
- 개념
  - SQL : `LEAD`함수, `SUM(CASE~)`, `AVG(CASE~)`
  - 분석 : 재검색 비율, 재검색 키워드
- 검색 결과에 만족하지 못해, **새로운 키워드**로 검색한 사용자의 행동은
  - 검색을 어떻게 **개선**하면 좋을지 좋은 지표가 됨

### 재검색 비율 집계하기
- **재검색 비율**이란
  - 사용자가 검색 결과의 **출력**과 관계 엾이
  - 어떤 결과도 클릭하지 않고, 새로 검색을 **실행한 비율**을 나타냄
- **재검색 비율**을 **사용자의 행동 로그**에서 어떻게 집계할지 생각 필요
- 검색 결과 출력 로그(`action=search`)와 검색 화면에서
  - 상세 화면으로의 이동 로그(`action=detail`)를 시계열 순서로 나열해서
  - 각각의 줄에 다음 줄의 액션을 기록
- 다음 줄의 액션을 **윈도 함수**로 추출하는 쿼리

#### CODE.21.3. 검색 화면과 상세 화면의 접근 로그에 다음 줄의 액션을 기록하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  access_log_With_next_action AS (
    SELECT
      stamp
      , session
      , action
      , LEAD(action)
      -- PostgreSQL, Hive, Redshift, BigQuery의 경우
      OVER(PARTITION BY session ORDER BY stamp ASC)
      -- SparkSQL, Frame 지정 필요
      OVER(PARTITION BY session ORDER BY stamp ASC
        ROWS BETWEEN 1 FOLLOWING AND 1 FOLLOWING)
          AS next_action
    FROM
      access_log
  )
  SELECT *
  FROM access_log_with_next_Action
  ORDER BY
    session, stamp
  ;
  ```
- `action`과 `next_action` 모두가 `search`인 레코드는 **재검색 수**를 의미
- 추가로 `action=search`인 레코드를 집계하고
  - 이를 검색 총 수로 보면, 두 값을 통해 재검색 비율 집계 가능

#### CODE.21.4. 재검색 비율을 집계하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  access_log_with_next_action AS (
    -- CODE.21.3.
  )
  SELECT
    -- PostgreSQL, Hive, Redshift, SparkSQL, substring으로 날짜 부분 추출
    substring(stamp, 1, 10) AS dt
    -- PostgreSQL, Hive, BigQuery, SparkSQL, substr 사용
    , substr(stamp, 1, 10) AS dt
    , COUNT(1) AS search_count
    , SUM(CASE WHEN next_action = 'search' THEN 1 ELSE 0 END) AS retry_count
    , AVG(CASE WHEN next_action = 'search' THEN 1.0 ELSE 0.0 END) AS retry_rate
  FROM
    access_log_with_next_action
  WHERE
    action = 'search'
  GROUP BY
    -- PostgreSQL, Redshift, BigQuery
    -- SELECT 구문에서 정의한 별칭을 GROUP BY 지정 가능
    dt
    -- PostgreSQL, Hive, Redshift, SparkSQL의 경우
    -- SELECT 구문에서 별칭을 지정하기 이전의 식을 GROUP BY 지정 가능
    substring(stamp, 1, 10)
  ORDER BY
    dt
  ;
  ```

### 재검색 키워드 집계하기
- **재검색 키워드**를 집계하면
  - **동의어 사전**이 `흔들림을 잡지 못하는 범위 확인`이 가능
- 추가로 컨텐츠 명칭의 **새로운 흔들림**과
  - 사람들이 일반적으로 해당 컨텐츠를 **부르는 이름**을 찾는 척도가 됨
- 사용자들이
  - 어떤 검색어로 **재검색** 했는지 확인 및 집계 이후
  - 대처해야 하는 부분이 있다면 **동의어 사전 반영** 필요

#### CODE.21.5. 재검색 키워드를 집계하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  access_log_with_next_search AS (
    SELECT
      stamp
      , session
      , action
      , keyword
      , result_num
      , LEAD(action)
        -- PostgreSQL, Hive, Redshift, BigQuery
        OVER(PARTITION BY session ORDER BY stamp ASC)
        -- SparkSQL, 프레임 지정
        OVER(PARTITION BY session ORDER BY stamp ASC
          ROWS BETWEEN 1 FOLLOWING AND 1 FOLLOWING)
        AS next_action
      , LEAD(keyword)
        -- PostgreSQL, Hive, Redshift, BigQuery
        OVER(PARTITION BY session ORDER BY stamp ASC)
        -- SparkSQL, 프레임 지정
        OVER(PARTITION BY session ORDER BY stamp ASC
          ROWS BETWEEN 1 FOLLOWING AND 1 FOLLOWING)
        AS next_keyword
      , LEAD(result_num)
        -- PostgreSQL, Hive, Redshift, BigQuery
        OVER(PARTITION BY session ORDER BY stamp ASC)
        -- SparkSQL, 프레임 지정
        OVER(PARTITION BY session ORDER BY stamp ASC
          ROWS BETWEEN 1 FOLLOWING AND 1 FOLLOWING)
        AS next_result_num
    FROM
      access_log
  )
  SELECT
    keyword
    , result_num
    , COUNT(1) AS retry_count
    , next_keyword
    , next_result_num
  FROM
    access_log_with_next_search
  WHERE
    action = 'search'
    AND next_action = 'search'
  GROUP BY
    keyword, result_num, next_keyword, next_result_num
  ```

### 원포인트
- 동의어 사전을 사람이 직접 관리하는 것은 매우 힘들며,
  - 담당자의 실수로 `흔들림 제거`가 되지 않을 수 있음
- 따라서 **재검색 키워드**를 집계하고
  - 검색 시스템이 **자동으로 흔들림 제거하도록 개선**하는 방법이 좋음

## 3. 재검색 키워드를 분류해서 집계하기
- 개념
  - SQL : `LIKE`
  - 분석 : 재검색 킹워드
- 사용자가 **재검색** 했다는 것은
  - 검색 결과에 만족하지 못했으며,
  - 다른 **동기**가 있다는 것을 의미
- 특정 패턴을 볼 때
  - 사용자가 어떤 키워드로 검색/변경 여부를 확인할 수 있다면
  - 이를 **동의어 사전**과 **사용자 사전 추가 후보**로 확인 가능
    - 검색 개선 가능성 존재

### 재검색의 동기
- `NoMatch`에서의 조건 변경
  - 검색 결과가 `0`개 이므로, **다른 검색어**로 검색
- 검색 필터링
  - 검색 결과가 너무 많으므로, 단어를 필터링
- 검색 키워드 변경
  - 검색 결과가 나오기는 했으나, **다른 검색어**로 재검색

### NoMatch에서의 조건 변경
- `NoMatch`에서 조건을 변경했다면,
  - 해당 키워드는 **동의어 사전**과 **사용자 사전에 추가할 키워드 후보**를 의미

#### CODE.21.6. NoMatch에서 재검색 키워드를 집계하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  access_log_with_next_search AS (
    -- CODE.21.5
  )
  SELECT
    keyword
    , result_num
    , COUNT(1) AS retry_count
    , next_keyword
    , next_result_num
  FROM
    access_log_with_next_search
  WHERE
    action = 'search'
    AND next_Action = 'search'
    -- NoMatch 로그만 필터링하기
    AND result_num = 0
  GROUP BY
    keyword, result_num, next_keyword, next_result_num
  ```

### 검색 결과 필터링
- 재검색한 **검색 키워드**가 원래의 검색 키워드를 **포함** 하고 있다면
  - 검색을 조금 더 **필터링**하고 싶은 의미일 수 있음
- 자주 사용되는 **필터링 키워드**가 있다면
  - 이를 원래 키워드로 검색 했을 때
  - **연관 검색어** 등으로 출력하여, 사용자가 요구하는 컨텐츠로 빠르게 유도 가능

#### CODE.21.7. 검색 결과 필터링 시의 재검색 키워드를 집계하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  access_log_with_next_search AS (
    -- CODE.21.5.
  )
  SELECT
    keyword
    , result_num
    , COUNT(1) AS retry_count
    , next_keyword
    , next_result_num
  FROM
    access_log_with_next_search
  WHERE
    action = 'search'
    AND next_action = 'search'
    -- 원래 키워드를 포함하는 경우만 추출하기
    -- PostgreSQL, Hive, BigQuery, SparkSQL, concat 함수 사용
    AND next_keyword LIKE concat('%', keyword, '%')
    -- PostgreSQL, Redshift, || 연산자 사용
    AND next_keyword LIKE '%' || keyword || '%'
  GROUP BY
    keyword, result_num, next_keywrod, next_result_num
  ;
  ```

### 검색 키워드 변경
- 완전히 다른 검색 키워드를 이용하여 재검색 했을 경우
  - 원래 검색 키워드를 사용한 검색 결과에 **원하는 내용이 없음**
- **동의어 사전**이 정상 동작하지 않음
- 완전히 다른 검색 키워드로 검색했을 경우
  - 분명 **이유**가 있을 것
  - 집계하여 확인

#### CODE.21.7. 검색 키워드를 변경 때 재검색을 집계하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  access_log_with_next_search AS (
    -- CODE.21.5.
  )
  SELECT
    keyword
    , result_num
    , COUNT(1) AS retry_count
    , next_keyword
    , next_result_num
  FROM
    access_log_with_next_search
  WHERE
    action = 'search'
    AND next_action = 'search'
    -- 원래 키워드를 포함하지 않는 검색 결과만 추출
    -- PostgreSQL, Hive, BigQuery, SparkSQL, concat 함수 사용
    AND next_keyword NOT LIKE concat('%', keyword, '%')
    -- PostgreSQL, Redshift, || 연산자 사용
    AND next_keyword NOT LIKE '%' || keyword || '%'
  GROUP BY
    keyword, result_num, next_keyword, next_result_num
  ;
  ```

### 정리
- 위 세가지 경우 모두 매우 다른 결과를 노출함
  - 세가지를 모두 **집계**하여 활용할 것

## 4. 검색 이탈 비율과 키워드 집계하기
- 개념
  - SQL : `SUM(CASE ~)`, `AVG(CASE ~)`
  - 분석 : 검색이탈률, 검색 이탈 키워드
- 검색 결과가 출력된 이후
  - 어떤 액션도 취하지 않고, 이탈한 사용자는 **검색에 만족하지 못한 경우**
- 검색 결과 화면에서 사용자가 이탈한다면
  - 검색 결과가 `0개`이거나
  - 원하는 결과가 나오지 않은 경우
- 이러한 사용자가 얼마나 존재하는지 집계하고
  - 그 때의 **키워드**를 확인하면, 여러가지 개선 가능

### 검색 이탈 집계 방법
- **CODE.21.3**의 결과를 활용
  - `action=search, next_action is NULL` : 검색 이탈
- `action=search`임을 **검색 총 수**로 활용하여
  - `검색 이탈 수 / 검색 총 수 = 검색 이탈 비율` 추산 가능

#### CODE.21.10. 검색 이탈 비율을 집계하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  access_log_With_next_action AS (
    -- CODE.21.9
  )
  SELECT
    -- PostgreSQL, Hive, Redshift, SparkSQL, substring으로 날짜 추출
    substring(stamp, 1, 10) AS dt
    -- PostgreSQL, Hive, BigQuery, SparkSQL, substr 사용
    substr(stamp, 1, 10) AS dt
    , COUNT(1) AS search_count
    , SUM(CASE WHEN next_action IS NULL THEN 1 ELSE 0 END) AS exit_count
    , AVG(CASE WHEN next_action IS NULL THEN 1.0 ELSE 0.0 END) AS exit_rate
  FROM
    access_log_with_next_action
  WHERE
    action = 'search'
  GROUP BY
    -- PostgreSQL, Redshift, BigQuery
    -- SELECT 구문에서 정의한 별칭을 GROUP BY에 지정 가능
    dt
    -- PostgreSQL, Hive, Redshift, SparkSQL
    -- SELECT 구문에서 별칭을 지정하기 이전의 식을 GROUP BY에 지정 가능
    substring(stamp, 1, 10)
  ORDER BY
    dt
  ;
  ```

### 검색 이탈 키워드 집계하기
- 검색 이탈이 발생했을 때 **검색한 키워드**를 추출하는 쿼리
- 검색 이탈 키워드를 활용하면
  - **동의어 사전 추가**등의 여러 가지 조치를 취해 검색 개선 가능

#### CODE.21.11. 검색 이탈 키워드를 집계하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  access_log_with_next_search AS (
    -- CODE.21.5
  )
  SELECT
    keyword
    , COUNT(1) AS search_count
    , SUM(CASE WHEN next_action IS NULL THEN 1 ELSE 0 END) AS exit_count
    , AVG(CASE WHEN next_action IS NULL THEN 1.0 ELSE 0.0 END) AS exit_rate
    , result_num
  FROM
    access_log_with_next_search
  WHERE
    action='search'
  GROUP BY
    keyword, result_num
    -- 키워드 전체의 이탈률을 계산한 후, 이탈률이 0보다 큰 키워드만 추출하기
  HAVING
    SUM(CASE WHEN next_action IS NULL THEN 1 ELSE 0 END) > 0
  ```

### 원포인트
- 검색에서 이탈한 사용자가 검색 결과에 만족하지 못하는 이유는 다양함
  - e.g. 원하는 상품이 **상위에 표시되지 않음** 등의 출력 순서 관련 이슈
- 참고로 **정렬 순서 평가 지표**에 대한 내용은 **21.6** 참고

## 5. 검색 키워드 관련 지표의 집계 효율화 하기
- `NoMatch` 비율, `재검색` 비율, `검색 이탈` 비율
- 각각의 지표과 **검색 키워드**를 산출하기 위해
  - 매번 비슷한 쿼리를 작성하는 것은 비효율적
- 따라서 다음과 같이 **집계 효율화**를 위한
  - **중간 데이터 생성 SQL**을 사용

### CODE.21.12. 검색과 관련된 지표를 집계하기 쉽게 중간 데이터를 생성하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  access_log_with_next_search AS (
    -- CODE.21.5
  )
  , search_log_with_next_action (
    SELECT *
    FROM
      access_log_with_next_search
    WHERE
      action = 'search'
  )
  SELECT *
  FROM search_log_with_next_action
  ORDER BY
    session, stamp
  ;
  ```

### 원포인트
- 앞의 출력 결과를 사용하면 `NoMatch` 수, `재검색` 수, `검색 이탈 수`를 포함해
  - 키워드 등을 간단하게 집계 가능
- 이러한 출력 결과를 테이블로 저장하거나,
  - 검색과 관련된 지표를 **전처리**하는, 정형화된 `WITH`구문으로 활용하면
  - 작업 효율을 높일 수 있음

## 6. 검색 결과의 포괄성을 지표화하기
- 개념
  - SQL : `FULL OUTER JOIN`, `SUM` 윈도 함수
  - 분석 : 재현율
- **검색 키워드**에 대한 지표를 사용하여
  - **검색 엔진 자체의 정밀도**를 평가하는 방법
- 이를 활용하면, 정밀도를 사람이 하나하나 평가하지 않아도 
  - 기계적으로 **자동화** 가능
- 샘플 데이터
  - 검색 키워드에 대한 검색 결과 순위(**DATA.21.2**) : search_result
    - keyword, rank, item
  - 검색 키워드에 대한 정답 아이템(**DATA.21.3**) : correct_result
    - keyword, item
    - 검색 키워드에 대해 올바른 결과를 나타낸 아이템을 미리 정리한 것

### 재현율(Recall)을 사용해 검색의 포괄성 평가하기
- 검색 엔진을 **정량적**으로 평가하는 대표적인 지표로
  - 재현율(`Recall`)과 정확률(`Precision`)이 존재
- 재현율
  - 어떤 키워드의 검색 결과에서
    - 미리 준비한 **정답 아이템**이 얼마나 나왔는지 비율로 표기
  - 특정 키워드 10개의 검색 결과가 나왔으면 할때
    - 실제 4개만 나온다면, `40%`의 재현율
- 재현율을 집계하려면
  - 일단 **검색 결과**와 **정답 아이템**을 결합하고
  - 어떤 아이템이 **정답 아이템**에 포함되는지 판단해야 함

#### CODE.21.13. 검색 결과와 정답 아이템을 결합하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  search_result_with_correct_items AS (
    SELECT
      COALESCE(r.keyword, c.keyword) AS keyword
      , r.rank
      , COALESCE(r.item, c.item) AS item
      , CASE WHEN c.item IS NOT NULL THEN 1 ELSE 0 END AS correct
    FROM
      search_result AS r
      FULL OUTER JOIN
        correct_result AS c
        ON r.keyword = c.keyword
        AND r.item = c.item
  )
  SELECT *
  FROM
    search_Result_with_correct_items
  ORDER BY
    keyword, rank
  ;
  ```
- `correct` 컬럼의 플래그가 `1`인 아이템이
  - **정답 아이템**에 포함된 아이템
- `OUTER JOIN`을 사용해
  - 검색 결과에 포함되지 않은 **정답 아이템 레코드**를 남긴다
    - 재현율을 계산하려면 **정답 아이템의 총 수**를 구해야하기 때문

#### CODE.21.14. 검색 상위 n개의 재현율을 계산하는 쿼리
- 상위 `n`개의 정답 아이템 히트 수는, `SUM` 윈도 함수를 사용하여
  - 누계 정답 아이템 플래그를 모두 합하여 계산
  - 이 값을 **정답 항목 총 수**로 나누면 **재현율**을 구할 수 있음
- 검색 결과에 포함되지 않은 아이템의 레코드는
  - 재현율을 계산할 수 없음 -> 편의상 `0`으로 출력
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  search_result_with_correct_items AS (
    -- CODE.21.13.
  )
  , search_result_with_recall AS (
    SELECT
      *
      -- 검색 결과 상위에서, 정답 데이터에 포함되는 아이템 수의 누계 계산
      , SUM(corret)
      -- rank=NULL, 아이템의 정렬 순서에 마지막에 위치
      -- 편의상 가장 큰 값으로 변환
        OVER(PARTITION BY keyword ORDER BY COALESCE(rank, 100000) ASC
          ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS cum_correct
      , CASE
        -- 검색 결과에 포함되지 않은 아이템은 편의상 적합률을 0으로 다루기
        WHEN rank IS NULL THEN 0.0
        ELSE
          100.0
          * SUM(correct)
              OVER(PARTITION BY keyword ORDER BY COALESCE(rank, 100000) ASC
                ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
              / SUM(correct) OVER(PARTITION BY keyword)
          END AS recall
      FROM
        search_Result_with_correct_items
  )
  SELECT *
  FROM
    search_result_with_recall
  ORDER BY
    keyword, rank
  ;
  ```

### 재현율의 값을 집약해서 비교하기 쉽게 만들기
- 위 코드에서 구한 재현율은 **검색 결과 전체 레코드**에 대한 재현율
  - 이 값으로 **검색 엔진 평가**는 어려움
- 검색 엔진을 **정량적**으로 평가하고
  - 여러 개의 **검색 엔진 비교**하려면
  - 하나의 **대표값**으로 집약하는 것이 바람직
- **검색 엔진**의 일반적인 인터페이스는
  - 검색 결과 `상위 n개`를 순위 형식으로 출력
  - 각 검색 키워드에 대한 **재현율**을 사용자에게 출력되는 **아이템 개수**로 한정해서 구하는 것이 좋음
    - 첫번째 페이지에 재현되는 아이템이 몇개인지를 구하는 것이 좋다는 의미

#### CODE.21.15. 검색 결과 상위 5개의 재현율을 키워드별로 추출하는 쿼리
- 검색 결과의 출력 결과가 `default=5`로 가정하여
  - 재현율을 **키워드**별 계산
- 검색 결과가 `5`개 이상이라면, 상위 `5`개로 재현율을 구하고
  - 검색 결과가 `5`개보다 적을 경우, **검색 결과 전체**를 기반으로 재현율을 구함
- 검색 결과가 없다면 재현율은 `0`
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  search_result_with_correct_items AS (
    -- CODE.21.13
  )
  , search_result_with_recall AS (
    -- CODE.21.14
  )
  , recall_over_rank_5 AS (
    SELECT
      keyword
      , rank
      , recall
      -- 검색 결과 순위가 높은 순서로 번호 붙이기
      -- 검색 결과에 나오지 않는 아이템은 편의상 0으로 다루기
      , ROW_NUMBER()
          OVER(PARTITION BY keyword ORDER BY COALESCE(rank, 0) DESC)
        AS desc_number
    FROM
      search_result_with_recall
    WHERE
      -- 검색 결과 상위 5개 이하 또는 검색 결과에 포함되지 않은 아이템만 출력
      COALESCE(rank, 0) <= 5
  )
  SELECT
    keyword
    , recall AS recall_at_5
  FROM recall_over_rank_5
  -- 검색 결과 상위 5개 중에서 가장 순위가 높은 레코드 추출하기
  WHERE desc_number = 1
  ;
  ```

#### CODE.21.16. 검색 엔진 전체의 평균 재현율을 계산하는 쿼리
- 위 코드에서 검색 키워드의 재현율 집약 이후, 마지막으로 전체적인 **재현율 평균** 구하기
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  search_result_with_correct_items AS (
    -- CODE.21.13
  )
  , search_result_with_recall AS (
    -- CODE.21.14
  )
  , recall_over_rank_5 AS (
    -- CODE.21.15
  )
  SELECT
    avg(recall) AS average_recall_at_5
  FROM recall_over_rank_5
  -- 검색 결과 상위 5개 중에서 가장 순위가 높은 레코드 추출하기
  WHERE desc_number = 1
  ;
  ```

### 정리
- 재현율은 **정답 아이템**에 포함되는 아이템을
  - 어느 정도 망라할 수 있는지를 나타내는 지표
- 재현율은 **의학** 또는 **법** 관련 정보를 다룰 때도 많이 사용됨

## 7. 검색 결과의 타당성을 지표화하기
- 개념
  - SQL : `FULL OUTER JOIN`, `SUM` 윈도 함수
  - 분석 : 정확률
- 검색 결과를 평가할 때 사용하는 지표중 하나인 **정확률**(Precision)
  - 간단하게 **정밀도**라고 부르기도 함
- **정확률**은
  - 검색 결과에 포함되는 아이템 중, **정답 아이템**이 어느 정도 비율로 포함되는가를 나타냄
  - e.g. 검색 결과 상위 10개에 5개의 정답 아이템이 포함되어 있다면, `정확률은 50%`

### 정확률(Precision)을 사용해 검색의 타당성 평가하기
- 검색 결과 상위 `n`개의 정확률을 계산하는 쿼리
- 재현율과 거의 유사하나
  - 분모 부분만 **검색 결과 순위까지의 누계 아이템 수**로 변경
  - 이전과 동일하게, 검색 결과에 포함되지 않은 아이템의 레코드도
    - 편의상 정확률을 `0`으로 계산

#### CODE.21.17. 검색 결과 상위 n개의 정확률을 계산하는 쿼리
```sql
WITH
search_result_with_Correct_items AS (
  -- CODE.21.13
)
, search_result_with_precision AS (
  SELECT
    *
    -- 검색 결과의 상위에서 정답 데이터에 포함되는 아이템 수의 누계 구하기
    , SUM(correct)
      -- rank가 NULL이라면 정렬 순서의 마지막에 위치하므로
      -- 편의상 굉장히 큰 값으로 변환하기
      OVER(PARTITION BY keyword ORDER BY COALESCE(rank, 100000) ASC
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS cum_correct
    , CASE
      -- 검색 결과에 포함되지 않은 아이템은 편의상 적합률을 0으로 다루기
        WHEN rank IS NULL THEN 0.0
        ELSE
          100.0
          * SUM(correct)
              OVER(PARTITION BY keyword ORDER BY COALESCE(rank, 100000) ASC
                ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
          -- 재현률과 다르게, 분모에 검색 결과 순위까지의 누계 아이템 수 지정하기
          / COUNT(1)
              OVER(PARTITION BY keyword ORDER BY COALESCE(rank, 100000) ASC
                ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
        END AS precision
  FROM
    search_result_with_correct_items
)
SELECT *
FROM
  search_result_with_precision
ORDER BY
  keyword, rank
;
```

### 정확률 값을 집약해서 비교하기 쉽게 만들기
- 키워드마다 값을 집약하기
- 검색 결과 상위 5개의 정확률을 키워드로 추출
  - 재현율을 계산할 때 사용한 쿼리와 거의 같음
- 검색 결과 상위 `n`개의 정확률은 `P@n`이라고 표기

#### CODE.21.18. 검색 결과 상위 5개의 정확률을 키워드별로 추출한 쿼리
```sql
WITH
search_result_with_correct_items AS (
  -- CODE.21.13
)
, search_result_with_precision AS (
  -- CODE.21.17
)
, precision_over_rank_5 AS (
  SELECT
    keyword
    , rank
    , precision
    -- 검색 결과 순위가 높은 순서로 번호 붙이기
    -- 검색 결과에 나오지 않는 아이템은 편의상 0으로 다루기
    , ROW_NUMBER()
        OVER(PARTITION BY keyword ORDER BY COALESCE(rank, 0) DESC) AS desc_number
  FROM
    search_result_with_precision
  WHERE
    -- 검색 결과의 상위 5개 이하 또는 검색 결과에 포함되지 않는 아이템만 출력하기
    COALESCE(rank, 0) <= 5
)
SELECT
  keyword
  , precision AS precision_at_5
FROM precision_over_rank_5
  -- 검색 결과의 상위 5개 중에서 가장 순위가 높은 레코드만 추출하기
  WHERE desc_number = 1;
```

#### CODE.21.19. 검색 엔진 전체의 평균 정확률을 계산하는 쿼리
- 검색 키워드들의 정확률을 추출한 이후
  - 정확률의 **평균**을 구하고,
  - 검색 엔진 전체의 **평균 정확률**을 구하기
```sql
WITH
search_Result_With_correct_items AS (
  -- CODE.21.13
)
, search_Result_with_precision AS (
  -- CODE.21.17
)
, preceision_over_rank_5 AS (
  -- CODE.21.18
)
SELECT
  AVG(precision) AS average_precision_at_5
FROM precision_over_rank_5
-- 검색 결과 상위 5개 중에서 가장 순위가 높은 레코드만 추출하기
WHERE desc_number=1
;
```

### 정리
- 정확률은 **검색 결과 상위**에 출력되는
  - **아이템의 타당성**을 나타내는 지표
- 이 지표는 **웹 검색**등
  - 방대한 데이터 검색에서 **적절한 검색 결과**를 빠르게 찾고 싶은 경우에 중요한 지표
  

## 8. 검색 결과 순위와 관련된 지표 계산하기
- 개념
  - SQL : `AVG` 함수
  - 분석 : `MAP`(Mean Average Precision)
- **재현율**과 **정확률**으로 실제 검색 엔진 검토시, 복잡한 조건 검토하게 되면 부족한 점 존재
  - 검색 결과의 순위는 고려하지 않음
  - 정답과 정답이 아닌 아이템을 `0`과 `1`이라는 두가지 값으로밖에 표현할 수 없음
  - 모든 아이템에 대한 정답을 미리 준비하는 것은 사실 **불가능**에 가까움
- 검색 결과의 순위는 고려하지 않음 예시
  - 검색 상위 10개의 **적합률**이 `40%`인 검색엔진 A, B가 존재할 때,
  - A는 `1, 4`번째에 정답 아이템 존재, 검색엔진 B는 `7, 10`번째 정답 아이템 존재
  - 당연히 `1`이 더 좋은 검색 엔진이라 할 수 있으나, `P@10`을 구하면, 두 검색엔진은 **같은 성능**으로 판단
  - 검색 엔진을 고려한 지표
    - MAP(Mean Average Precision)
    - MRR(Mean Reciprocal Rank)
- 정답과 정답이 아닌 아이템이라는 두가지로 밖에 표현하지 못함
  - 이전 절까지 다룬 정답 아이템의 경우, 어떤 아이템이 **검색에 히트**되어도 **같은 것으로 취급**
  - 경우에 따라 **굉장히 관련 있는 아이템**, **조금 관련있는 아이템**처럼, 단계쩍인 점수를 사용해 정답 아이템을 다루고 싶은 경우 존재
  - 예시
    - 사용자의 별점(5단계) 리뷰 데이터가 존재할 때,
    - 해당 점수를 정답 데이터를 사용하여 **적합률**과 **재현률**을 구할 경우
      - 사용자로부터 입력받은 **별점**을 활용할 수 없음
  - 단계적인 점수를 고려하여 정답 아이템을 다루는 지표
    - DCG(Discounted Cumulated Gain)
    - NDCG(Normalized DCG)
- 모든 아이템에 대해 정답 아이템을 준비할 수 없음
  - 검색 엔진이 필요한 서비스의 경우, 검색 대상 아이템이 많을 수 있음
  - 사용자 **행동 로그**등에서, 기계적으로 정답 아이템을 만들 수 있겠으나,
    - 사용자가 각각 확인할 수 있는 아이템의 수는 많지 않음
  - **재현율**과 **적합률**을 적용하더라도, 극단적으로 낮은 값이기 때문에 사용하기 어려움
  - 검색 대상 아이템 수에서, 정답 아이템 수가 한정된 경우에는 **BPREF**(Binary Preference)를 활용
  
  ### MAP으로 검색 결과의 순위를 고려해 평가하기
  - 검색 결과의 순위를 고려한 평가
  - 검색 결과 상위 `n`개의 **적합률** 평균
  - 정답 아이템이 순위에 아예 없는경우, 편의상 적합률을 `0`으로 정의
  - 예시
    - 정답 아이템 수가 `4`개라고 할때, `P@10 = 40%`
    - 상위 `1~4번째`가 모두 정답 아이템
      - MAP 값은 `100 * ((1/1) + (2/2) + (3/3) + (4/4))/4 = 100`으로 계산
      - 상위 `7~10`번째가 정답 아이템이라면, `P@10 = 40%`
        - `MAP = 100 * ((1/7) + (2/8) + (3/9) + (4/10))/4 = 28.15`
          - 이전보다 낮게 평가함
          - MAP을 계산하려면, **정답 아이템**별로 **정확률** 추출 필요
            - 이전 절의 쿼리를 활용하여 `connect`컬럼의 플래그가 `1`인 레코드를 추출하면 됨
            
#### CODE.21.20. 정답 아이템별로 적합률을 추출하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  search_result_with_correct_items AS (
  -- CODE.21.13.
  )
  , search_result_with_precision AS (
  -- CODE.21.17
  )
  SELECT
    keyword
    , rank
    , precision
  FROM
    search_result_with_precision
  WHERE
    correct = 1
  ;
  ```

#### CODE.21.21. 검색 키워드별로 정확률의 평균을 계산하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  search_result_with_correct_items AS (
  -- CODE.21.13
  )
  , search_result_with_precision AS (
  -- CODE.21.17
  )
  , average_precision_for_keywords AS (
  SELECT
    keyword
    , AVG(precision) AS average_precision
  FROM
    search_result_with_precision
  WHERE
    correct = 1
  GROUP BY
    keyword
  )
  SELECT *
  FROM
    average_precision_for_keywords
  ;
  ```
  
#### CODE.21.22. 검색 엔진의 MAP을 계산하는 쿼리
- 검색 키워드별로 정확률의 평균을 추출한 이후, 검색 엔진 자체의 MAP을 계산
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  search_result_with_correct_itmes AS (
  -- CODE.21.13
  )
  , search_result_with_precision AS (
  -- CODE.21.17
  )
  , average_precision_for_keywords AS (
  -- CODE.21.21
  )
  SELECT
    AVG(average_precision) AS mean_average_precision
  FROM
    average_precision_for_keywords
  ;
  ```

### 검색 평가와 관련한 다른 지표들
#### 정확률(Precision)
- 검색 결과 순위 중에서, 얼마나 많은 비율의 아이템이 제대로 출력되는가
- 웹 검색에서 처럼, 방대한 **중복**을 포함한 정보에서
  - 관련 정보를 재빠르게 추출해야할 때 사용

#### 재현율(Recall)
- 사용자가 원할 것으로 보이는 아이템 중에서
  - 얼마나 많은 비율의 아이템이 제대로 출력되는가
- `법률계` 또는 `의학계`를 대상으로 하는 검색 처럼
  - 결과에 **누수**가 있으면 안되는 경우에 중요

#### P@n
- 순위 상위 `n`개를 기반으로 측정한 Precision

#### MAP(Mean Average Precision)
- 각각의 쿼리에 대한 Average Precision의 평균값

#### MRR(Mean Reciprocal Rank)
- 순위 중에서 **처음부터 정답 아이템이 나오는 순위**의 **역수**를 취한 뒤 평균을 구한 값
- 평균역순위

#### E-Measure
- `Precision`과 `Recall`을 합쳐 만든 단일 지표
- `Precision`과 `Recall`을 1:1로 합치면 `F-Measure`가 됨

#### F-Measure
- `Precision`과 `Recall`의 조회 평균

#### NDCG(Normalized Discounted Cumulated Gain)
- 아이템의 평가를 **여러 레벨**로 설정하고,
  - 순위 내부에서 **순위를 추가**한 지표
- 정답 데이터를 만들기 어렵지만, **신뢰도가 매우 높음**

#### BPREF(Binary Preferences)
- 아이템 사이의 **상대적인 관심도**를 기준으로 하는 지표
- 아이템 공간이 굉장히 거대해서
  - 평가 데이터를 만들기 힘든 경우에 사용
  
### 정리
- 검색 엔진을 **정량적**으로 평가하기 위한 지표는
  - 위 정리 내용 외에 많은 것이 존재
- NDCG와 BPREF와 같은 다른 지표에 대해서도
  - 정의와 적용 범위를 확인 이후
  - 어떻게 쿼리를 작성하면 구할수 있을지 고민할 것
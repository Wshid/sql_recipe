# 15. 사이트 내의 사용자 행동 파악하기
- 웹 사이트의 특징적인 지표, 리포트를 작성하는 SQL
  - 방문자 수, 방문 횟수, 직귀율, 이탈률
- **접근 분석 도구**를 사용하는 서비스라면, 위 지표를 이미 파악했을 것
- 하지만 **빅데이터 기반**으로 데이터 추출 시,
  - **사용자** 또는 **상품 데이터** 등의 각종 **업무 데이터**와 조합하여 상세히 분석 가능
- **직귀율**과 **이탈률**은 비슷해 보이나, 실제 지표의 정의가 다름
- 샘플 데이터 : 구인/구직 데이터
  - 사이트 맵이 정해져 있음
  - `검색 결과 목록` 페이지에서 `지역`과 `직종`을 조건으로 지정할 수 있음
  - `지역`은 여러 depth 가 있음
  - `직종`은 `인사`, `영업`, `기술` 지정 가능
  - `검색 결과 목록 페이지`에서 검색을 하면 로그가 저장됨

## 1. 입구 페이지와 출구 페이지 파악하기
- 개념
  - SQL : `FIRST_VALUE`, `LAST_VALUE`
  - 분석 : 입구페이지, 출구페이지
- 입구 페이지 : 사이트 방문시, **처음 접근한 페이지**
  - **랜딩 페이지**라고도 부름
- 출구 페이지 : 마지막 접근 페이지
  - **이탈 페이지**라고도 부름

### 입구 페이지와 출구 페이지 집계하기
- 접근 로그의 각 **세션**에서, 입구페이지와 출구페이지의 URL을 집계하려면
  - `FIRST_VALUE`, `LAST_VALUE`의 **윈도 함수**를 사용해야 함
- `OVER` 구문 지정에는 `ORDER BY stamp ASC`로, 타임스탬프를 오름 차순으로 지정
  - `ORDER BY`를 지정한 윈도 함수의 **파티션**은
    - 기본적으로 `첫 행 ~ 현재 행`
    - `ROWS BETWEEN ~`을 사용해서 각 세션 내부의 **모든 행**을 대상으로 지정

#### CODE.15.1. 각 세션의 입구 페이지와 출구 페이지 경로를 추출하는 쿼리
```sql
WITH
activity_log_with_landing_exit AS (
  SELECT
    session
    , path
    , stamp
    , FIRST_VALUE(path)
      OVER(
        PARTITION BY session
        ORDER BY stamp ASC
          ROWS BETWEEN UNBOUNDED PRECEDING
                    AND UNBOUNDED FOLLOWING
      ) AS landing
    , LAST_VALUE(path)
      OVER(
        PARTITION BY session
        ORDER BY stamp ASC
          ROWS BETWEEN UNBOUNDED PRECEDING
                    AND UNBOUNDED FOLLOWING
      ) AS exit
  FROM activity_log
)
SELECT *
FROM
  activity_log_with_landing_exit
;
```
- 각 세션의 **입구 페이지**와 **출구페이지**의 URL이 추가된 것을 확인할 수 있음
- 다음 코드처럼, **입구 페이지**와 **출구 페이지**의 URL을 기반으로
  - 유니크한 **세션**의 수를 집계
  - **방문 횟수**를 구할 수 있음

#### CODE.15.2. 각 세션의 입구 페이지와 출구 페이지를 기반으로 방문 횟수를 추출하는 쿼리
- `PostgreSQL`, `Hive`, `Redshit`, `BigQuery`, `SparkSQL`
```sql
WITH
activity_log_with_landing_exit AS (
  ...
)
, landing_count AS (
  -- 입구 페이지의 방문 횟수 집계
  SELECT
    landing AS path
    , COUNT(DISTINCT session) AS count
  FROM
    activity_log_with_landing_exit
  GROUP BY landing
)
, exit_count AS (
  -- 출구 페이지의 방문 횟수 집계
  SELECT
    exit AS path
    , COUNT(DISTINCT session) AS count
  FROM
    activity_log_with_landing_exit
  GROUP BY exit
)
-- 입구 페이지와 출구 페이지 방문 횟수 결과 한번에 출력
SELECT 'landing' AS type, * FROM landing_count
UNION ALL
SELECT 'exit' AS type, * FROM exit_count
;
```

### 어디에서 조회를 시작해서, 어디에서 이탈하는지 집계하기
- 위 코드에서 입구 페이지와 출구 페이지의 **방문 횟수**를 파악할 수 있으나,
  - 어떤 페이지에서 조회하기 시작하여,
  - 어떤 페이지에서 이탈하는지 확인 불가
- 조회 **시작 페이지**와 **이탈 페이지**의 조합을 집계하는 쿼리

#### CODE 15.3. 세션별 입구 페이지와 출구 페이지의 조합을 집계하는 쿼리
- `PostgreSQL`, `Hive`, `Redshit`, `BigQuery`, `SparkSQL`
```sql
WITH
activity_log_with_landing_exit AS (
  ...
)
SELECT
  landing
  , exit
  , COUNT(DISTINCT session) AS count
FROM
  activity_log_with_landing_exit
GROUP BY
  landing, exit
;
```

### 원포인트
- 웹 사이트의 담당자는 최상위 페이지부터 사이트 설계 진행
- 하지만 최상위 페이지부터 **조회**를 시작하는 사용자는 **거의 없음**
- **상세 페이지**부터 조회를 시작하는 사용자도 많음
- 사용자가 어디로 **유입**되는지 **입구 페이지**를 파악하면
  - 사이트를 더 매력적으로 설계 가능

## 2. 이탈률과 직귀율 계산하기
- 개념
  - SQL : `ROW_NUMBER`, `COUNT 윈도 함수`, `AVG(CASE ~ END)`
  - 분석 : 이탈률, 직귀율
- **출구 페이지**를 사용하여 **이탈률**을 계산하고,
  - 문제가 되는 페이지를 찾아내는 방법
- **직귀율**
  - 특정 페이지만 조회하고, 곧바로 이탈한 비율을 나타내는 지표
  - 직귀율이 높은 페이지
    - 성과로 바로 **이어지지 않을 가능성**이 높음
    - 확인하고 대책 필요
  - 광고로 수익을 얻는 사이트의 경우
    - 여러 페이지를 조회해야 수익이 많지만,
    - **메인 페이지**에서 **직귀율**이 높다면,
      - 사용자가 어떤 정보를 찾을 동기가 약하기 때문
    - 사이트 배치등을 검토할 수 있음

### 이탈률 집계하기
- 이탈률 공식
  - `이탈률 = 출구 수 / 페이지 뷰`
- 단순히 **이탈률**이 높다고 해서, 나쁜 페이지는 아님
  - 사용자가 만족해서 이탈하는 페이지(`구매 완료`, `기사 상세 화면`)는 당연히 이탈률이 높아야 함
    - 문제가 되지 않음
- 다만, 사용자가 만족하지 못해, 중간 과정에 이탈한다면
  - 개선 검토 필요
- 이탈률 리포트(경로, 출구 수, 페이지 뷰, 이탈률)
  >**경로**|**출구 수 **|**페이지 수**|**이탈률**
  >-----|-----|-----|-----
  >/detail| 3,667 | 9,365 |39.10%
  >/search\_list| 2,574 | 8,516 |30.20%

#### CODE.15.4. 경로별 이탈률을 집계하는 쿼리
- `PostgreSQL`, `Hive`, `RedShift`, `BigQuery`, `SparkSQL`
```sql
WITH
activity_log_with_exit_flag AS (
  SELECT
    *
    -- 출구 페이지 판정
    , CASE
        WHEN ROW_NUMBER() OVER(PARTITION BY session ORDER BY stamp DESC) = 1 THEN 1
        ELSE 0
      END AS is_exit
    FROM
      activity_log
)
SELECT
  path
  , SUM(is_exit) AS exit_count
  , COUNT(1) AS page_view
  , AVG(100.0 * is_exit) AS exit_ratio
FROM
  activity_log_with_exit_flag
GROUP BY path
;
```

### 직귀율 집계하기
- 직귀 수(**한 페이지만을 조회한 방문 횟수**)를 구한 뒤, 다음과 같은 식을 사용해 계산
  - `직귀율 = 직귀 수 / 입구 수`
  - `직귀율 = 직귀 수 / 방문 횟수`
- 순수하게 **랜딩 페이지**에서 다른 페이지로 이동하는지 평가할 때는,
  - 전자의 지표가 더 바람직한 계산식
- 특정 페이지를 조회하고, 곧바로 이탈하는 원인은
  - 연관 기사 또는 상품으로 사용자를 **이동**시키는 모듈이 **기능하지 않는 점**
  - 페이지 자체의 **콘텐츠**에 사용자가 만족하지 못하는 경우
  - 이동이 **복잡**해서 다음 단계로 이동하지 못하는 경우
- 직귀율 리포트(경로, 직귀 수, 입구 수, 직귀율)
  >**경로**|**직귀 수**|**입구 수**|**직귀율**
  >-----|-----|-----|-----
  >/detail| 1,652 | 3,539 |46.60%
  >/search\_list| 585 | 2,330 |25.10%

#### CODE.15.5. 경로들의 직귀율을 집계하는 쿼리
- `PostgreSQL`, `Hive`, `RedShift`, `BigQuery`, `SparkSQL`
```sql
WITH
activity_log_with_landing_bounce_flag AS (
  SELECT
    *
    -- 입구 페이지 판정
    , CASE
        WHEN ROW_NUMBER() OVER(PARTITION BY session ORDER BY stamp ASC) = 1 THEN 1
        ELSE 0
      END AS is_landing
    -- 직귀 판정
    , CASE
        WHEN COUNT(1) OVER(PARTITION BY session) = 1 THEN 1
        ELSE 0
      END AS is_bounce
  FROM
    activity_log
)
SELECT
  path
  , SUM(is_bounce) AS bounce_count
  , SUM(is_landing) AS landing_count
  , AVG(100.0 * CASE WHEN is_landing = 1 THEN is_bounce END) AS bounce_ratio
FROM
  actibity_log_with_landing_bounce_flag
GROUP BY path
;
```

### 원포인트
- **컴퓨터 전용 사이트**와 **스마트폰 전용 사이트**가 개별적으로 존재하는 경우
  - 사용자가 원하는 **콘텐츠**나 **콘텐츠 배치**에 차이가 있을 수 있음
- 이런 경우, 컴퓨터 전용 사이트와 스마트폰 전용 사이트를
  - **따로따로 집계**할 것

## 3. 성과로 이어지는 페이지 파악하기
- 개념
  - SQL : `SIGN` 함수, `SUM` window 함수
  - 분석 : CVR
- 방문 횟수를 아무리 늘려도, 성과와 관계 없는 페이지로만 사용자가 유도된다면,
  - 성과가 발생하지 않음
- **성과와 직결되는 페이지**로 유도해야
  - 웹사이트 전체의 **CVR**을 향상시킬 수 있음
- 여러 가지 검색 기능을 제공할 때,
  - 성과에 이르는 비율이 **적은 검색 기능**이 있다면
  - 위치를 옮기거나, 삭제하는 등의 방안 검토
- 페이지를 비교 했을 때
  - **조회 수**는 높지 않지만
  - 성과에 이르는 비율이 높다면
    - 해당 페이지의 **중요도**가 높음
  - 페이지의 **구성 검토**시에 좋은 정보가 될 수 있음

### 목표
- 어떤 페이지로 방문하였을 때, 성과로 빠르게 연결될 수 있는지 파악할 수 있게 아래 내용 파악
  - 페이지 또는 경로에 대한 **방문 횟수**
  - 방문이 **성과로 연결**되는지 여부
- 완료 화면(`/complete`)에 도달하는 것을 성과(`conversion`)이라 정의하고,
  - 완료 화면에 도달할 때까지, 사용자가 방문한 **경로** 계산

### 다양한 비교 패턴
- 기능기반 비교
  - 업종 검색
  - 직종 검색
  - 조건 검색
- 캠페인 기반 비교 
  - 축하금 캠페인
  - 기프트카드 캠페인
- 페이지 기반 비교
  - 서비스 소개
  - 비공개 구인
- 콘텐츠 종류 기반 비교
  - 상세 페이지(그림 있음)
  - 상세 페이지(그림 없음)
- 경로들의 방문 횟수와 성과

>**경로**|**방문 횟수**|**성과수**|**CVR**
>:-----:|:-----:|:-----:|:-----:
>/detail|5894|52|0.88%
>/search\_list|5045|31|0.61%

### CODE.15.6. 컨버전 페이지보다 이전 접근에 플래그를 추가하는 쿼리
- **윈도 함수**를 사용하여
  - **세션**들을 **타임 스탬프 내림차순**으로 정렬
- `/complete` 페이지에 접근할 때까지의 접근 로그에 **플래그**를 추가
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  , activity_log_with_conversion_flag AS (
    SELECT
      session
      , stamp
      , path
      -- 성과를 발생시키는 컨버전 페이지의 이전 접근에 플래그 추가
      , SIGN(SUM(CASE WHEN path = '/complete' THEN 1 ELSE 0 END)
              OVER(PARTITION BY session ORDER BY stamp DESC
                ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW))
        AS has_conversion
    FROM
      activity_log
  )
  SELECT *
  FROM
    activity_log_with_conversion_flag
  ORDER BY
    session, stamp
  ;
  ```
- 예시 결과
>**session**|**stamp**|**path**|**has\_conversion**
>-----|-----|-----|-----
>05e9cc2b|2016-12-29 21:49:50|/search\_list|1
>05e9cc2b|2016-12-30 21:49:50|/detail|1
>05e9cc2b|2016-12-31 21:49:50|/input|1
>05e9cc2b|2017-01-01 21:49:50|/confirm|1
>05e9cc2b|2017-01-02 21:49:50|/complete|1
>05e9cc2b|2017-01-03 21:49:50|/|0
>05e9cc2b|2017-01-04 21:49:50|/detail|0

- `/complete`로의 접근을 포함한 세션 로그에서는
  - `/complete`에 도달할 때까지의 접근로그의 `has_session_conversion` 컬럼에 `1`이라는 플래그가 붙음
- `/complete` 이후의 접근에서는 페이지 가치의 계산에 **포함되지 않으므로**
  - 플래그가 `0`이 됨
- `SIGN` 함수
  ```js
  expr < 0 ; return -1
  expr = 0 ; return 0
  expr > 0 ; return 1
  ```

### CODE.15.7. 경로들의 방문 횟수와 구성 수를 집계하는 쿼리
- **방문 횟수**와 **구성 수**를 집계하여 `CVR`을 계산
- 비율 계산에는
  - **분모**가 `distcint`이기 때문에 
  - 단순한 conversion flag의 `avg`가 아니라 `SUM/COUNT`로 계산
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  activity_log_with_conversion_flag AS (
    -- CODE.15.6.
  )
  SELECT
    path
    -- 방문 횟수
    , COUNT(DISTINCT session) AS sessions
    -- 성과 수
    , SUM(has_conversion) AS conversions
    -- 성과 수 / 방문 횟수
    , 1.0 * SUM(has_conversion) / COUNT(DISTINCT session) AS cvr
  FROM
    activity_log_with_conversion_flag
  -- 경로별로 집약
  GROUP BY path
  ;
  ```

### 원포인트
- 상품 구매, 자료 청구, 회원 등록을 **성과**라고 하면,
  - 성과 직전에 있는 페이지는 **CVR**이 **상당히 높게** 측정 됨
- 같은 계층의 컨텐츠, 유사 컨텐츠를 비교해볼 것
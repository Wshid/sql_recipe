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

## 4. 페이지 가치 산출하기
- 개념
  - SQL : `ROW_NUMBER` 함수
  - 분석 : 페이지 평가
- 성과로 이어지는 페이지로 사용자를 유도하는 것이
  - 웹사이트 매출 향상에 도움이 될 수 있음
- **사이트 맵**을 변경해서
  - 해당 페이지를 **경유**하게 만들거나
  - 해당 페이지의 **콘텐츠**와 **광고**를 다른페이지에 활용하는 것이 좋음
- **페이지 가치**라는 지표를 사용하여, 자세하게 페이지 분석하기

### 페이지 가치 집계 준비
- 페이지의 가치를 계산하려면 **성과 수치화**필요
- 어떻게 성과를 수치화 할 수 있을지 파악

#### 성과를 수치화 하기
- 해당 사이트에서의 성과가 **구매**라면
  - **구매 금액**을 기반으로 성과 수치화가 가능
- 성과는
  - `자료 청구`, `견적 의뢰 신청`등으로 설정할 수도 있고,
  - 이런 페이지를 하는 페이지로 **이동하는 버튼**을 클릭하는 것으로 설정 가능
- 이러한 설정을 기반을 기반으로
  - `자료 청구`, `견적 의뢰`등을 할 때마다 `1`이라는 점수를 부여하거나
  - 자료를 청구했을 때의 `매출`등을 점수로 부여하는 방법이 있음
- 매출 금액 등의 `구체적인 정보`를 산출할 수 없는 경우
  - 임시로 **ICV**라는 가치를 `1,000`으로 설정하기
  - 페이지의 가치를 **상대적**으로 비교 가능

#### 페이지의 가치를 할당하는 방법
- **페이지 가치**로 어떤 판단을 내리고 싶은지에 따라
  - 페이지 가치 **할당 로직**이 달라짐
- 5가지
  - **마지막 페이지에 할당하기**
    - 직접적인 효과가 있다고 판단할 수 있는 **페이지**에 **성과**를 모두 할당
    - **매출**에 직접적으로 기여하는 페이지 판단 가능
  - **첫 페이지에 할당하기**
    - 성과로 이어지는 **계기**가 되었던 첫 페이지에 성과를 모두 할당
    - 이를 활용하면 **매출**에 간접적으로 기여하는 페이지 판단 가능
    - `광고`또는 `검색 엔진`등의 **외부 유입**에서, 어떤 페이지가 가치가 높은지 판단 가능
  - **균등하게 분산하기**
    - 성과에 이르기까지 거쳤던 **모든 페이지**에 성과를 균등하게 할당
    - 어떤 페이지를 **경유**했을 때, 사용자가 성과에 이르는지 판단 가능
    - `적은 페이지를 거쳐 성과로 연결된 경우`와 `반복적으로 방문하는 페이지`의 경우
      - 페이지의 가치가 **높게** 측정됨 
  - **성과 지점에서 가까운 페이지에 더 높게 할당하기**
    - **마지막 페이지**에 가까울수록 높은 가치 할당
  - **성과 지점에서 먼 페이지에 더 높게 할당하기**
    - **첫 페이지**에 가까울수록 높은 가치 할당

### 특정한 계급 수로 히스토그램 만들기
- 페이지의 가치를 구하려면 아래 두 내용을 결정해야 함
  - 무엇을 **성과 수치**로 사용할 것인가
  - 무엇을 **평가**할 것인가
- 예시
  - `신청 입력 양식을 제출하고 완료 화면이 출력된 경우`를 성과
  - 이때의 성과 수치를 `1,000`으로 하여, **경유한 페이지**에 균등 할당
  - 단, 페이지를 평가 할 때 `입력 ~ 확인 ~ 완료` 페이지를 포함하면, 집계가 제대로 이루어지지 않으므로
    - 이는 집계 대상에서 제외
    - 위 세 페이지는 `신청할 때 무조건 거치는 페이지`이기 때문에, 집계할 필요가 없음

#### CODE.15.8. 페이지 가치 할당을 계산하는 쿼리
- 페이지 가치 계산 대상 조건은
  - **CODE.15.6**의 `has_conversion = 1`인 `입력/확인/완료`이외의 접근 로그
  - `WHERE`절에 명시
  - `SELECT`구문 내부에서 한 conversion에 `1,000`이라는 가치 할당 이후,
  - 하나의 로그별 가치를 **윈도 함수**로 집계
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  activity_log_with_session_conversion_flag AS (
    -- CODE.15.6.
  )
  , activity_log_with_conversion_assign AS (
    SELECT
      session
      , stamp
      , path
      -- 성과에 이르기까지 접근 로그를 오름차순 정렬
      , ROW_NUMBER() OVER(PARTITION BY session ORDER BY stamp ASC) AS asc_order
      -- 성과에 이르기까지 접근 로그를 내림차순으로 순번
      , ROW_NUMBER() OVER(PARTITION BY session ORDER BY stamp DESC) AS desc_order
      -- 성과에 이르기까지 접근 수 세기
      , COUNT(1) OVER(PARTITION BY session) AS page_count

      -- 1. 성과에 이르기까지 접근 로그에 균등한 가치 부여
      , 1000.0 / COUNT(1) OVER(PARTITION BY session) AS fair_assign

      -- 2. 성과에 이르기까지 접근 로그의 첫 페이지 가치 부여
      , CASE
          WITH ROW_NUMBER() OVER(PARTITION BY session ORDER BY stamp ASC) = 1
            THEN 1000.0
          ELSE 0.0
        END AS first_assign
      
      -- 3. 성과에 이르기까지 접근 로그의 마지막 페이지 가치 부여
      , CASE
          WHEN ROW_NUMBER() OVER(PARTITION BY session ORDER BY stamp DESC) = 1
            THEN 1000.0
          ELSE 0.0
        END AS last_assign
      
      -- 4. 성과에 이르기까지 접근 로그의 성과 지점에서 가까운 페이지 높은 가치 부여
      , 1000.0
          * ROW_NUMBER() OVER(PARTITION BY session ORDER BY stamp ASC)
          -- 순번 합계로 나누기( N*(N+1)/2 )
          / ( COUNT(1) OVER(PARTITION BY session)
              * (COUNT(1) OVER(PARTITION BY session) + 1) / 2)
        AS decrease_assign

      -- 5. 성과에 이르기까지의 접근 로그의 성과 지점에서 먼 페이지에 높은 가치 부여
      , 1000.0
          * ROW_NUMBER() OVER(PARTITION BY session ORDER BY stamp DESC)
          -- 순번 합계로 나누기( N*(N+1)/2 )
          / ( COUNT(1) OVER(PARTITION BY session) 
              * (COUNT(1) OVER(PARTITION BY session) + 1) / 2)
        AS increase_assign
    FROM activity_log_with_conversion_flag
    WHERE
      -- conversion으로 이어지는 세션 로그만 추출
      has_conversion = 1
      -- 입력, 확인, 완료 페이지 제외하기
      AND path NOT IN ('/input', '/confirm', '/complete')
  )
  SELECT
    session
    , asc_order
    , path
    , fair_assign AS fair_a
    , first_assign AS first_a
    , last_assign AS last_a
    , decrease_assign AS dec_a
    , increase_assign AS inc_a
  FROM
    activity_log_with_conversion_assign
  ORDER BY
    session, asc_order;
  ```
- 출력 결과 확인시,
  - 한 `conversion`에 `1,000`의 가치가 부여됨
- `fair_assign`, 세션의 수로 나눈 **평균값**, 처음 접근에만 가치 할당
- `last_assign`, 마지막 접근에만 가치 할당

#### CODE.15.9. 경로별 페이지 가치 합계를 구하는 쿼리
- 페이지 가치의 합계를 **경로별**로 집계
- 현재 상태에서는 **페이지 뷰**가 많은 페이지에서, 페이지의 가치가 높게 판정
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  activity_log_with_session_conversion_flag AS (
    -- CODE.15.6.
  )
  , activity_log_With_conversion_assign AS (
    -- CODE.15.8.
  )
  , page_total_values AS (
    -- 페이지 가치 합계 계산
    SELECT
      path
      , SUM(fair_assign) AS fair_assign
      , SUM(first_assign) AS first_assign
      , SUM(last_assign) AS last_assign
    FROM
      activity_log_with_conversion_assign
    GROUP BY
      path
  )
  SELECT * FROM page_total_values;
  ```

### CODE.15.10. 경로들의 평균 페이지 가치를 구하는 쿼리
- 페이지 가치의 합계를 구했다면,
  - **페이지 뷰**가 적으면서도, **높은 페이지 가치**를 가진 페이지를 찾기
  - `페이지 가치 / (페이지 방문 횟수 OR 페이지 뷰)`
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  activity_log_with_session_conversion_flag AS (
    -- CODE.15.6.
  )
  , activity_log_with_conversion_assign AS (
    -- CODE.15.8
  ), page_total_values AS (
    -- CODE.15.9
  )
  , page_total_cnt AS (
    -- 페이지 뷰 계산
    SELECT
      path
      , COUNT(1) AS access_cnt -- 페이지 뷰
      -- 방문 횟수로 나누고 싶은 경우, 다음과 같은 코드 작성
      , COUNT(DISTINCT session) AS access_cnt
    FROM
      activity_log
    GROUP BY
      path
  )
  SELECT
    -- 한 번의 방문에 따른 페이지 가치 계산
    s.path
    , s.access_cnt
    , v.sum_fair / s.access_cnt AS avg_fair
    , v.sum_first / s.access_cnt AS avg_first
    , v.sum_last / s.access_cnt AS avg_last
    , v.sum_dec / s.access_cnt AS avg_dec
    , v.sum_inc / s.access_cnt AS avg_inc
  FROM
    page_total_cnt AS s
  JOIN
    page_total_values As v
    ON s.path = v.path
  ORDER BY
    s.access_cnt DESC;
  ```

### 정리
- 대부분의 접근 분석 도구는 **페이지의 평가 산출 로직이 한정**되는 경우가 많음
- 무엇을 **평가**하고 싶은지를 명확하게 생각하고, 자유롭게 계산할 수 있게 되면,
- 접근 분석 도구의 제한 없이, 새로운 가치 분석이 가능

## 5. 검색 조건들의 사용자 행동 가시화 하기
- 개념
  - SQL : `SIGN` 함수, `SUM(CASE~END)`, `AVG(CASE~END)`, `LAG` 함수
  - 분석 : CTR, CVR
- 상품 또는 정보 검색 사이트에서는 **다양한 검색 방법 제공**
  - EC의 경우 `카테고리`, `제조사`, `가격대`등의 필터 기능 제공
  - 구인/구직 사이트의 경우 `지역`, `직종`등의 필터 기능
- **검색 조건**을 더 자세하게 지정하여 사용하는 사용자의 동기가 명확함
  - **성과**로 이어지는 경우가 많음
- 검색 조건이 미흡하다면,
  - **검색 조건 지정**을 유도하여, 성과를 높일 수 있음
- CTR & CVR
  - CTR : **검색 조건**을 활용하여 **상세 페이지**로 이동한 비율
  - CVR : 상세 페이지 조회 후에 **성과**로 이어지는 비율
- CTR을 x축, CVR을 y축으로 두었을 때,
  - value는 방문 횟수,
  - CTR, CVR이 높고, 검색 조건(왼쪽 위)쪽으로 사용자를 이동시키면 **성과**가 늘어날 수 있음
- 차트를 그리기 위해 필요한 표
  - 검색 타입 : 검색 결과 페이지의 `URL 매개변수`를 분석하여 분류
  - CTR : 검색 화면에서 상세화면 이동 수 집계(단, 1회 이동, 3회 이동 모두 `1`로 치환)
  >**검색 타입**|**방문횟수**|**CTR**|**상세 화면 이동 세션 수**|**CVR**|**성과 세션 수**
  >-----|-----|-----|-----|-----|-----
  >지역|1,683|51.40%|866|1.20%|11
  >큰 에리어|1,199|50.90%|611|1.10%|7
  >큰 에리어 + 직종 지정|603|56.00%|338|0.20%|1

### CODE.15.11. 클릭 플래그와 컨버전 플래그를 계산하는 쿼리
- 클릭 플래그 : 상세 페이지로의 이동을 의미
- 컨버전 플래그 : 최종적으로 컨버전에 도달했는지를 판별
- 윈도 함수로 집계
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  activity_log_with_session_click_conversion_flag AS (
    SELECT
      session
      , stamp
      , path
      , search_type
      -- 상세 페이지 이전 접근에 플래그 추가
      , SIGN(SUM(CASE WHEN path = '/detail' THEN 1 ELSE 0 END)
                  OVER(PARTITION BY session ORDER BY stamp DESC
                    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW))
        AS has_session_click
      -- 성과를 발생시키는 페이지의 이전 접근에 플래그 추가
      , SIGN(SUM(CASE WHEN path='/complete' THEN 1 ELSE 0 END)
                  OVER(PARTITION BY session ORDER BY stamp DESC
                    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW))
        AS has_session_conversion
    FROM activity_log
  )
  SELECT
    session
    , stamp
    , path
    , search_type
    , has_session_click AS click
    , has_session_conversion AS cnv
  FROM
    activity_log_with_session_click_conversion_flag
  ORDER BY
    session, stamp
  ;
  ```

### CODE.15.2. 검색 타입별 CTR, CVR을 집계하는 쿼리
- **클릭 플래그**와 **컨버전 플래그** 계싼 이후
  - **검색 로그**로 압축하여, `접근 수`, `클릭 수`, `전환 수`, `CTR`, `CVR` 계산
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  activity_log_with_session_click_conversion_flag AS (
    -- CODE.15.11
  )
  SELECT
    search_type
    , COUNT(1) AS count
    , SUM(has_session_click) AS detail
    , AVG(has_session_click) AS ctr
    , SUM(CASE WHEN has_session_click = 1 THEN has_session_conversion END) AS conversion
    , AVG(CASE WHEN has_session_click = 1 THEN has_session_conversion END) AS cvr
  FROM
    activity_log_with_session_click_conversion_flag
  WHERE
    -- 검색 로그만 추출
    path = '/search_list'
  -- 검색 조건으로 집약
  GROUP BY
    search_type
  ORDER BY
    count DESC
  ;
  ```
- 1회의 방문에서 **여러개의 검색 타입**으로 검색한 경우 **각각 모두 카운트**
- 전체적인 느낌을 파악할 때는 괜찮으나,
  - 성과 직전의 검색 결과만을 원할 때는, `WITH`구문을 아래와 같이 수정해서 사용해야 함

### CODE.15.13. 클릭 플래그를 직전 페이지에 한정하는 쿼리
- `LAG` 함수를 사용해 상세 페이지로 접근하기 직전의 접근에 **플래그**를 붙ㅌ임
- `LAG` 함수의 `OVER`구문에서 `ORDER BY stamp DESC`와 타임스탬프를 **내림차순**으로 하는 것은
  - `has_session_conversion` 컬럼의 `OVER`구문과 정렬 조건으로 효율적인 정렬 처리를 하기 위함
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  activity_log_with_session_click_conversion_flag AS (
    SELECT
      session
      , stamp
      , path
      , search_type
      -- 상세 페이지의 직전 접근에 플래그 추가
      , CASE
          WHEN LAG(path) OVER(PARTITION BY session ORDER BY stamp DESC) = '/detail'
            THEN 1
          ELSE 0
        END AS has_session_click
      -- 성과가 발생하는 페이지의 이전 접근에 플래그 추가
      , SIGN(
        SUM(CASE WHEN path ='/complete' THEN 1 ELSE 0 END)
          OVER(PARTITION BY session ORDER BY stamp DESC
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
        ) AS has_session_conversion
    FROM
      activity_log
  )
  SELECT
    session
    , stamp
    , path
    , search_type
    , has_session_click AS click
    , has_session_conversion AS cnv
  FROM
    activity_log_with_session_click_conversion_flag
  ORDER BY
    session, stamp
  ;
  ```

### 원포인트
- 조건을 상세하게 지정하더라도
  - 걸리는 항목이 **적거나 없으면**
  - 사용자가 **이탈**할 확률이 높음
- 각각의 **검색 조건**과 **히트되는 항목 수**의 균형을 고려해서
  - 카테고리를 어떻게 묶을지 **검토** 및 **개선** 필요

## 6. 폴아웃 리포트를 사용해 사용자 회유를 가시화하기
- 최상위 페이지에서의 **검색 조건 입력**
- 검색 결과 목록에서 **상세 화면**으로의 이동
- **입력 양식 입력**부터 **확인/완료**까지 이어지는 **사용자 회유 흐름** 중
  - 어디에서 이탈이 많은지, 얼마나 이동이 이루어지는지를 확인하고, 개선할 수 있다면
    - 전체적인 **CVR**을 향상 시킬 수 있음 
- 이전 절까지는
  - 해당 페이지를 조회한 사용자가 **최종적으로 성과에 도달**할때까지 확인하는법 확인
- 이번 절에서는
  - **중간 지점의 도달률**도 함께 집계하는 방법
- **Fall Trough** : 어떤 지점에서, 다른 지점으로 옮겨가는 것
- **Fall Out** : 어떤 지점에서의 **이탈**
- **폴 아웃 리포트** : 여러 지점에서의 이동률을 집계한 리포트
  >**단계**|**방문 횟수**|**선두로부터의 이동률**|**직전까지의 이동률**
  >-----|-----|-----|-----
  >/| 30,841 |100.00%|100.00%
  >/list| 15,603 |50.60%|50.60%
  >/detail| 12,042 |39.00%|77.20%
  >/input| 565 |1.80%|4.70%
  >/complete| 188 |0.60%|33.30%
  - URL에 기재된 지점간의 이동은
    - 접속자가 다른 페이지를 경유 했거나, 직후에 이동했는지는 상관 없이
    - **다음 지점에 도달한 방문 횟수**를 기록
    - 표의 URL 순서대로 이동을 했다는 의미가 아님
  - 직전/직후의 이동에 관련한 부분은 **15.7** 확인

### CODE.15.14. 폴아웃 단계 순서를 접근 로그와 결합하는 쿼리
- 단계(step) 순서를 번호로 명시한 마스터 테이블(`mst_fallout_step`)을 작성 후 **로그 데이터**와 결합
- 추가로 세션들을 **스텝 순서(& URL)**로 집계하여,
  - 폴아웃 리포트에서 필요한 로그를 선별하는 컬럼 계산
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  mst_fallout_step AS (
    -- 폴아웃 단계와 경로의 마스터 테이블
              SELECT 1 AS step, '/' AS path
    UNION ALL SELECT 2 AS step, '/search_list' AS path
    UNION ALL SELECT 3 AS step, '/detail' AS path
    UNION ALL SELECT 4 AS step, '/input' AS path
    UNION ALL SELECT 5 AS step, '/complete' AS path
  )
  , activity_log_with_fallout_step AS (
    SELECT
      l.session
      , m.step
      , m.path
      -- 첫 접근과 마지막 접근 시간 구하기
      , MAX(l.stamp) AS max_stamp
      , MIN(l.stamp) AS min_stamp
    FROM
      mst_fallout_step AS m
      JOIN
        activity_log As l
        ON m.path = l.path
    GROUP BY
      -- 세션별로 단계 순서와 경로를 사용해 집약
      l.session, m.step, m.path
  )
  , activity_log_with_mod_fallout_step AS (
    SELECT
      session
      , step
      , path
      , max_stamp
      -- 직전 단계에서의 첫 접근 시간 구하기
      , LAG(min_stamp)
          OVER(PARTITION BY session ORDER BY step)*-
          -- sparkSQL의 경우 LAG 함수에 프레임 지정 필요
          OVER(PARTITION BY session ORDER BY step
            ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING)
        AS lag_min_stamp
      -- 세션에서의 단계 순서 최소값 구하기
      , MIN(step) OVER(PARTITION BY session) AS min_step
      -- 해당 단계 도달할때까지 걸린 단계 수 누계
      , COUNT(1)
        OVER(PARTITION BY session ORDER BY step
          ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
        AS cum_count
    FROM
      activity_log_with_fallout_step
  )
  SELECT
    *
  FROM
    activity_log_with_mod_fallout_step
  ORDER BY
    session, step
  ;
  ```

### CODE.15.5. 폴아웃 리포트에 필요한 로그를 압축하는 쿼리
- 쿼리 해석
  1) session에 `step=1` URL이 있는 경우
     - `step=1` URL(`/`)에 접근하지 않은 경우, **폴아웃 리포트** 대상에서 제외
  2) 현재 step이 해당 `step`에 도달할때까지 누계 `step`수와 같은지 확인
     - `step=2` URL(`/search_list`)를 스킵하고, `step=3` URL(`/detail`)에 직접 접근한 로그 등을 제외
  3) 바로 전의 `step`에 처음 접근한 시각이 현재 `step`의 최종 접근 시각보다 이전인지 확인
     - `step=3` 페이지에 접근한 이후, `step=2`로 돌아간 경우,
       - `step=3`에 접근했던 로그를 제외
     - 다만, `step=1`의 경우, 앞 페이지가 따로 존재하지 않기 때문에, 예외적으로 다루어야 함
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  mst_fallout_step AS (
    -- CODE.15.14
  )
  , activity_log_with_fallout_step AS (
    -- CODE.15.14
  )
  , activity_log_with_mod_fallout_step AS (
    -- CODE.15.14
  )
  , fallout_log AS (
    -- 폴아웃 리포트에 사용할 로그만 추출
    SELECT
      session
      , step
      , path
    FROM
      activity_log_with_mod_fallout_step
    WHERE
      -- 세션에서 단계 순서가 1인지 확인하기
      min_step = 1
      -- 현재 단계 순서가 해당 단계의 도달할 떄까지 누계 단계 수와 같은지 확인
      AND step = cum_count
      -- 직전 단계의 첫 접근 시간이 NULL 또는 현재 시간의 최종 접근 시간보다 이전인지 확인
      AND (lag_min_stamp IS NULL OR max_stamp >= lag_min_stamp)
  )
  SELECT
    *
  FROM
    fallout_log
  ORDER BY
    session, step
  ;
  ```

### CODE.15.6. 폴아웃 리포트를 출력하는 쿼리
- **스텝 순서**와 **url**로 집약하고
  - **접근수**와 **페이지 이동률**을 집계
- 스텝들의 접근 수와 페이지 이동률을 집계하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  mst_fallout_step AS (
    -- CODE.15.4
  )
  , activity_log_with_fallout_step AS (
    -- CODE.15.4
  )
  , fallout_log AS(
    -- CODE.15.5
  )
  SELECT
    step
    , path
    , COUNT(1) AS count
    -- 단계 순서 = 1 URL부터의 이동률
    , 100.0 * COUNT(1)
      / FIRST_VALUE(COUNT(1))
        OVER(ORDER BY step ASC ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)
      AS first_trans_rate
    -- 직전 단계까지의 이동률
    , 100.0 * COUNT(1)
      / LAG(COUNT(1)) OVER(ORDER BY step ASC)
      -- sparkSQL의 경우 LAG함수에 프레임 지정 필요
      / LAG(COUNT(1)) OVER(ORDER BY step ASC ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING)
      AS step_trans_rate
  FROM
    fallout_log
  GROUP BY
    step, path
  ORDER BY
    step
  ;
  ```
- `FIRST_VALUE` 함수 사용시에 `OVER(ORDER BY...)` 구문을 사용해야 함
  - https://clausvisby.com/ko/1690-how-to-use-first_value-038-last_value-function-in-sql-server.html

### 정리
- 폴아웃 리포트는
  - 웹사이트가 사용자를 어떻게 **유도**하는지를 대략적으로 파악하고 싶을 때 사용
- **성과**로 이어진 사용자와, 이어지지 않은 사용자를 **따로 구분**해보면
  - 더 유용한 정보 집계 가능

## 7. 사이트 내부에서 사용자 흐름 파악하기
- 개념
  - SQL : `LAG` 함수, `LEAD` 함수, `SUM` 윈도 함수
  - 분석 : 사용자 흐름
- 웹 사이트에서 사용자를 어떻게 **유도**하는지 파악하면
  - `사이트맵`, `메뉴`, `모듈 배치`등을 개선할 수 있음
- 사용자가 쓰는 기능이, `서비스 제공자가 생각하는 것`과 다르다고 생각된다면,
  - 서비스 사용자에 맞게 사이트 구성을 변경하거나,
  - 제공자의 의도에 맞게 서비스를 사용하도록 **유도**하는 등의 검토 필요
- **15.6**에서는
  - 특정 페이지를 조회한 사용자인지,
  - 특정 페이지로 이동한 사용자가 **어느정도 존재**하는지 확인하는 방법 확인
- 이번절에서는, 더 **상세하게**활용할 수 있는 **사용자 흐름 분석 방법** 소개

### 사용자 흐름 그래프
- 무엇을 분석할지 결정하고, 시작 지점으로 삼을 페이지 결정 필요
- 두가지 질문
  - **최상위 페이지에서 어떤 식으로 유도하는가**
    - 일단 `검색`을 사용하는가
    - 서비스를 `소개하는 페이지`를 보는가
  - **상세 화면 전후에서 어떤 행동을 하는가**
    - 검색이 아니라 **추천**을 사용하는 경우가 있는가
    - **검색 결과**에서 **상세 화면**으로 이동하는 **비율**이 얼마나 되는가
    - 상세 화면 출력 후 **갖고 싶은 물건**에 추가하거나 **장바구니**에 담는 행동을 하는가
    - 상세 화면 출력 후 **추천**으로 다른 상품의 상세 화면으로 이동하는가

### 다음 페이지 집계 하기 
- **시작 지점이 되는 페이지**를 설정하면
  - 시작 지점에서부터 어떤 형태로 **유도**되는지 정리한 표를 집계하는 SQL 확인 가능
  >**시작 지점**|**방문 횟수**|**다음 페이지1**|**방문 횟수1**|**비율1**|**다음 페이지2**|**방문 횟수2**|**비율2**
  >:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:
  >/detail| 16,930 |/detail| 5,683 |33.60%|/| 1,793 |31.60%
  > | | | | |/list| 2,426 |42.70%
  > | | | | |NULL| 1,464 |25.80%
  > | |/list| 3,048 |18.00%|/map| 1,532 |50.30%
  > | | | | |/| 493 |16.20%
  > | | | | |NULL| 1,023 |33.60%
  > | |…| … |…|…| … |…
  > | |NULL| 5,301 |31.30%|NULL| 5,301 |31.30%
- `/detail` 페이지로 진입 했을 때, 어느 페이지로 가는 표로 정리
  - 전환할 때의 비율은 **이전 페이지 방문 횟수**를 **분모**로 넣어 계산
    - 이 때, 다음 페이지가 `NULL`이라면,
      - 다음 페이지를 보지 않고 이탈한 경우를 의미

#### CODE.15.7. `/detail` 페이지 이후의 사용자 흐름을 집계하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
```sql
WITH
activity_log_with_lead_path AS (
  SELECT
    session
    , stamp
    , path AS path0
    -- 곧바로 접근한 경로 추출하기
    , LEAD(path, 1) OVER(PARTITION BY session ORDER BY stamp ASC) AS path1
    -- 이어서 접근한 경로 추출하기
    , LEAD(path, 2) OVER(PARTITION BY session ORDER BY stamp ASC) AS path2
  FROM
    activity_log
)
, raw_user_flow AS (
  SELECT
    path0
    -- 시작 지점 경로로의 접근 수
    , SUM(COUNT(1)) OVER() AS count0
    -- 곧바로 접근한 경로 (존재하지 않는 경우 문자열 NULL)
    , COALESCE(path1, 'NULL') AS path1
    -- 곧바로 접근한 경로로의 접근 수
    , SUM(COUNT(1)) OVER(PARTITION BY path0, path1) AS count1
    -- 이어서 접근한 경로(존재하지 않는 경우 문자열로 `NULL` 지정)
    , COALESCE(path2, 'NULL') AS path2
    -- 이어서 접근한 경로로의 접근 수
    , COUNT(1) AS count2
  FROM
    activity_log_with_lead_path
  WHERE
    -- 상세 페이지를 시작 지점으로 두기
    path0 = '/detail'
  GROUP BY
    path0, path1, path2
)
SELECT
  path0
  , count0
  , path1
  , count1
  , 100.0 * count1 / count0 AS rate1
  , path2
  , count2
  , 100.0 * count2 / count1 AS rate2
FROM
  raw_user_flow
ORDER BY
  count1 DESC, count2 DESC
;
```
- 위 쿼리의 결과에서 **중복된 데이터**가 많이 나와, 흐름파악이 어려움
- 리포트 도구 등의 **사용자 흐름** 기능을 이요하면 쉽게 해결 가능
  - 단, SQL로 처리해보기

#### CODE.15.18. 바로 위의 레코드와 같은 값을 가졌을 때 출력하지 않게 데이터 가공
- `LAG`함수를 사용해서, 바로 위에 있는 레코드 **경로 이름**을 추출한 뒤,
  - 값이 **다를 경우**에만 출력하게 변경
- `LAG` 함수 값이 `NULL`이 될 가능성이 있기 때문에
  - `COALESCE` 함수를 사용해서, 적당한 디폴트 값 설정 
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  activity_log_with_lead_path AS (
    -- CODE.15.17
  )
  , raw_user_flow AS (
    -- CODE.15.17
  )
  SELECT
    CASE
      WHEN
        COALESCE(
          -- 바로 위의 레코드가 가진 path0 추출(존재 하지 않는 경우 NOT FOUND)
          LAG(path0)
            OVER(ORDER BY count1 DESC, count2 DESC)
            , 'NOT FOUND'
        ) <> path0
      THEN path0
    END AS path0
    , CASE
        WHEN
          COALESCE(
            LAG(path0)
            OVER(ORDER BY count1 DESC, count2 DESC)
            , 'NOT FOUND'
          ) AS path0
        THEN count0
      END AS count0
    , CASE
        WHEN
          COALESCE(
            -- 바로 위의 레코드가 가진 여러 값을 추출할 수 있게, 문자열 결합 후 추출
            -- PostgreSQL, Redshift의 경우 || 연산자 사용
            -- Hive, BigQuery, SparkSQL의 경우 concat 함수 사용
            LAG(path0 || path1)
              OVER(ORDER BY count1 DESC, count2 DESC)
            , 'NOT FOUND'
          ) <> (path0 || path1)
        THEN path1
      END AS page1
    , CASE
        WHEN
          COALESCE(
            LAG(path0 || path1)
              OVER(ORDER BY count1 DESC, count2 DESC)
            , 'NOT FOUND'
          ) <> (path0 || path1)
        THEN count1
      END AS count1
    , CASE
        WHEN
          COALESCE(
            LAG(path0 || path1)
              OVER(ORDER BY count1 DESC, count2 DESC)
            , 'NOT FOUND') <> (path0 || path1)
        THEN 100.0 * count1 / count0
      END AS rate1
    , CASE
        WHEN
          COALESCE(
            LAG(path0 || path1 || path2)
              OVER(ORDER BY count1 DESC, count2 DESC)
            , 'NOT FOUND') <> (path0 || path1 || path2)
        THEN path2
      END AS page2
    , CASE
        WHEN
          COALESCE(
            LAG(path0 || path1 || path2)
              OVER(ORDER BY count1 DESC, count2 DESC)
            , 'NOT FOUND') <> (path0 || path1 || path2)
        THEN count2
      END AS count2
    , CASE
        WHEN
          COALESCE(
            LAG(path0 || path1 || path2)
              OVER(ORDER BY count1 DESC, count2 DESC)
            , 'NOT FOUND') <> (path0 || path1 || path2)
        THEN 100.0 * count2 / count1
      END AS rate2
    FROM
      raw_user_flow
    ORDER BY
      count1 DESC
      , count2 DESC
  ;
  ```

### 이전 페이지 집계하기
- `/detail` 페이지를 시작 기점으로, 이전 흐름 두 단계를 집계하기
- 값이 `NULL`인 것은, 이전 페이지를 조회한 로그가 없다는 뜻 => 곧바로 방문
  >**이전페이지2**|**방문횟수2**|**이동률2**|**이전페이지1**|**방문 횟수1**|**이동률1**|**시작 지점**|**방문 횟수**
  >:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:
  >/| 3,781 |61.10%|/list| 6,191 |36.60%|/detail| 16,930 
  >/detail| 1,983 |32.00%| | | | | 
  >NULL| 428 |6.90%| | | | | 
  >/list| 302 |15.20%|/| 1,981 |11.70%| | 
  >/detail| 692 |34.90%| | | | | 
  >NULL| 987 |49.80%| | | | | 
  >…| … | |…| … | | | 
  >NULL| 6,049 |35.70%|NULL| 6,049 |35.70%| | 

#### CODE.15.19 /detail 페이지 이전의 사용자 흐름을 집계하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  activity_log_with_lag_path AS (
    SELECT
      session
      , stamp
      , path AS path0
      -- 바로 전에 접근한 경로 추출하기(존재하지 않는 경우 문자열 'NULL'로 지정)
      , COALESCE(LAG(path, 1) OVER(PARTITION BY session ORDER BY stamp ASC), 'NULL') AS path1
      -- 그 전에 접근한 페이지 추출하기(존재하지 않는 경우 문자열 'NULL'로 지정)
      , COALESCE(LAG(path, 2) OVER(PARTITION BY session ORDER BY stamp ASC), 'NULL') AS path2
    FROM
      activity_log
  )
  , raw_user_flow AS (
    SELECT
      path0
      -- 시작 지점 경로로의 접근 수
      , SUM(COUNT(1)) OVER() AS count0
      , path1
      -- 바로 전의 경로로의 접근 수
      , SUM(COUNT(1)) OVER(PARTITION BY path0, path1) AS count1
      , path2
      -- 그 전에 접근한 경로로의 접근 수
      , COUNT(1) AS count2
    FROM
      activity_log_with_lag_path
    WHERE
      -- 상세 페이지를 시작 지점으로 두기
      path0 = '/detail'
    GROUP BY
      path0, path1, path2
  )
  SELECT
    path2
    , count2
    , 100.0 * count2 / count1 AS rate2
    , path1
    , count1
    , 100.0 * count1 / count0 AS rate1
    , path0
    , count0
  FROM
    raw_user_flow
  ORDER BY
    count1 DESC
    , count2 DESC
  ;
  ```

### 원포인트
- 이번 절에서 소개한 `SQL`과 함께 사용해서
  - `컴퓨터 사용자`, `스마트폰 사용자`, `광고 유입자`에 따라 어떤 흐름이 발생하였는지 살펴보기
    - 새로운 문제를 찾고 해결 가능

## 8. 페이지 완독률 집계하기
- 개념
  - SQL : `SUM(CASE ~ END)`
  - 분석 : 완독률
- **직귀율**, **이탈률**, **페이지 뷰**를 확인하더라도
  - 사용자가 페이지 **끝까지** 조회했는지는 확인이 어려움
- 따라서 페이지 내용에 **만족**하면서 이탈했는지, 
  - 만족하지 못하여 **중간 이탈**을 했는지 확인이 어려움
- 완독률
  - 페이지를 끝까지 읽었는지 비율로 나타낸 것
  - 이를 집계하여
    - 사용자에게 **가치를 제대로 전달**했는지 확인 가능
- 특정 컨텐츠의 **완독률**이 높거나 낮다면,
  - 해당 종류의 컨텐츠가 사용자가 **원하는 콘텐츠**인지 아닌지 확인 가능
- 또한 **전체적인 완독률**이 낮다면
  - 페이지의 **가독성**이 낮을 가능성이 높음
  - 이럴 경우, `페이지의 폰트 변경`등으로 가독성을 높여 **개선**할 수 있음
- 완독률을 **집계**하려면
  - **페이지 조회 로그**와 함께
  - 어디까지 조회했는지를 파악할 수 있도록 **로그 데이터**가 존재해야 함
    - `view`, `read-20%`, `read-40%`, `read-60%`
- 이런 로그를 만들기 위해서는
  - **자바스크립트**를 사용하여, 어디까지 읽었는지를 전송할 수 있는 시스템이 구축되어야 함
  - 해당 페이지의 **길이**를 판단한 뒤에, `30%`, `60%` 조회시 정보 전송 후 로그 기입
- 페이지에 따라
  - 어떻게 어디까지 조회했는지 보다
  - **어느 부분**까지 조회했는지를 기록하는게 더 좋을 수 있음
  - `action`을 활용하여 집계
    - `view`, `read-news-start`, `read-recommend`, `read-ranking`, ...

### CODE.15.20. 완독률을 집계하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    url
    , action
    , COUNT(1)
      / SUM(CASE WHEN action='view' THEN COUNT(1) ELSE 0 END)
          OVER(PARTITION BY url)
      AS action_per_view
  FROM read_log
  GROUP BY
    url, action
  ORDER BY
    url, count DESC
  ;
  ```

### 원포인트
- `EC 사이트`에서는
  - `상품 상세 정보`아래에 `추천 상품`을 출력하는 모듈
- `뉴스 사이트`에서는
  - 기사 마지막에 `관련 기사`를 출력하는 모듈이 설치되어 있는 경우가 많음
- 이러한 **모듈**의 경우
  - 일반적으로 상품 정보와 기사를 **끝까지 봐야**하므로,
  - 페이지의 완독률이 낮으면 해당 모듈을 설치해도 의미가 없음
- 따라서 이러한 모듈을 사용할 때는
  - 페이지의 **완독률**을 확인하고
  - 모듈의 효과가 **충분히 효과가 발생할지** 고민해볼 것
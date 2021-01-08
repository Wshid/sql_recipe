# 16. 입력 양식 최적화하기
- 엔트리폼(Entry Form)
  - 자료 청구 양식과 구매 양식 등을 의미
  - 입력 양식
- **입력 양식**의 항목이 너무 많으면
  - 사용자가 **스트레스**를 받아 **이탈률**이 높아짐
- 입력 양식 최적화(EFO, Entry Form Optimization)
  - 위 이탈을 막고, 성과를 높이고자 **입력 양식을 최적화**하는 것

### 입력 양식 최적화의 종류
#### 필수 입력과 선택 입력을 명확하게 구분해서 입력 수를 줄임
- 필수 입력 항목을 위로 올려 배치
#### 오류 발생 빈도를 줄임
- 입력 예를 보여준다
- 제대로 입력하지 않았다고 **실시간**으로 알려준다
#### 쉽게 입력할 수 있게 만든다
- 입력 항목을 줄인다
- 우편번호와 주소 등의 자동 완성 기능을 사용한다
#### 이탈할 만한 요소를 제거한다
- 불필요한 링크를 제거한다(네비게이션 메뉴 단순화 등)
- 실수로 이탈하지 않게
  - 페이지를 벗어날 때, 확인 대화 상자를 띄워 이탈할 것인지, 다시 한번 묻는다

### 유의점 
- 입력 양식 최적화 전에, `어떤 문제가 있는지` 명확하게 파악해야 함
- 입력 양식을 사용하는 동안
  - 사용자가 얼마나 이탈하는지
  - 이를 수정했을 경우, 얼마나 **이탈률 개선**이 이루어지는지
  - **더 개선의 여지가 있는지** 모두 수치화 되어야 함

### 목표
- `EFO 과정` 자체는 물론, `EFO`시 사용하는 대표적인 지표와 리포트
- 기본적으로는 이번 절의 내용으로 각종 집계가 가능하나,
  - 같은 **호칭의 지표**라도, **입력 양식**에 따라 다른 경우도 있음

## 1. 오류율 집계하기
- 개념
  - SQL : `SUM(CASE ~)`, `AVG(CASE ~)`
  - 분석 : 오류율
- **입력 양식** 중에는
  - 발생한 **오류**를, 같은 `URL`에 재출력하여 통지하는 경우가 존재
- 이 경우에는, `URL`만으로는 확인 화면으로의 **이동률 집계가 어려움**
- **확인 화면**으로 이동했을 때
  - `정상 화면`인지, `오류 화면`인지를 집계할 수 있도록 `로그`를 기록
- 로그 데이터에는
  - 오류가 발생했을 때, 페이지 열람 로그에 `error`라는 상태를  출력
  - `stamp, session, action, url, status`
    - `status`에는 에러 여부를 기록

### CODE.16.1 확인 화면에서 오류율을 집계하는 쿼리
- `/confirm` 페이지에서 **오류**가 발생하여, **재입력 화면**을 출력한 경우를 집계
- 분모로 URL이 `/regist/confirm`인 경우를 지정
  - 상태 오류를 `CASE`식을 사용해 **플래그**로 변환 후, 
  - `SUM` 함수와 `AVG`함수로 **오류율 집계 
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    COUNT(*) AS confirm_count
    , SUM(CASE WHEN status='error' THEN 1 ELSE 0 END) AS error_count
    , AVG(CASE WHEN status='error' THEN 1.0 ELSE 0.0 END) AS error_rate
    , SUM(CASE WHEN status='error' THEN 1.0 ELSE 0.0 END) / COUNT(DISTINCT session) AS error_per_user
  FROM form_log
  WHERE
    -- 확인 화면 페이지 판정
    path = '/regist/confirm'
  ;
  ```

### 원포인트
- **오류율**이 높다면, **오류 통지 방법**에 문제가 있어
  - 사용자가 문제를 이해하지 못해 반복적으로 문제를 발생시키는 경우 일 수 있음
- 이때는, **오류 통지 방법**을 변경할 것

## 2. 입력~확인~완료까지의 이동률 집계하기
- 개념
  - SQL : `LAG`, `MIN`, `COUNT` 윈도 함수, `FIRST_VALUE` 함수
  - 분석 : 확정률, 이탈률
- **입력 양식**을 최적화 할때는
  - 일단 `입력~확인~완료`까지의 **폴아웃 리포트**를 확인해야 함
- 앞서 **16.1**처럼
  - `URL`만으로는 **확인 화면**을 출력한 것인지, 오류를 출력한뒤 다시 입려고하면을 재출력한 것인지 **판별이 어려움**
- 따라서 별도의 **상태 필드**를 포함시켜
  - **오류 상태**를 함께 **로그**에 저장하는 것이 좋음

### TABLE.16.1. 입력 양식을 활용한 폴아웃 리포트
>**단계**|**방문횟수**|**입력 화면부터의 이동률**|**직전으로부터의 이동률**
>:-----:|:-----:|:-----:|:-----:
>Input| 7,292 |100.0%|100.0%
>Confirm| 1,234 |16.9%|(*1)16.9%
>Complete| 1,033 |(*2)14.2%|83.7%
- **확정률** : `입력 시작 ~ 확인 화면`까지의 이동 비율
- **CVR** : `완료 화면`까지 이동한 비율
- **이탈률** : `100% - CVR`

### CODE.16.2. 입력 양식의 폴아웃 리포트
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  mst_fallout_step AS (
    -- /regist 입력 양식의 폴아웃 단계와 경로 마스터
              SELECT 1 AS step, '/regist/input'     AS path
    UNION ALL SELECT 2 AS step, '/regist/confirm'   AS path
    UNION ALL SELECT 3 AS step, '/regist/complete'  AS path
  )
  , form_log_with_fallout_step AS (
    SELECT
      l.session
      , m.step
      , m.path
      -- 특정 단계 경로의 처음/마지막 접근 시간 구하기
      , MAX(l.stamp) AS max_stamp
      , MIN(l.stamp) AS min_stamp
    FROM
      mst_fallout_step AS m
      JOIN
        form_log AS l
        ON m.path = l.path
    -- 확인 화면의 상태가 오류인 것만 추출하기
    WHERE status = ''
    -- 세션별로 단계 순서와 경로 집약하기
    GROUP BY l.session, m.step, m.path
  )
  , form_log_with_mod_fallout_step AS (
    SELECT
      session
      , step
      , path
      , max_stamp
      -- 직전 단계 경로의 첫 접근 시간
      , LAG(min_stamp)
          OVER(PARTITION BY session ORDER BY step)
          -- sparkSQL의 경우 LAG 함수에 프레임 지정 필요
          OVER(PARTITION BY session ORDER BY step
            ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING)
        AS lag_min_stamp
      -- 세션 내부에서 단계 순서 최솟값
      , MIN(step) OVER(PARTITION BY session) AS min_Step
      -- 해당 단계에 도달할 때까지의 누계 단계 수
      , COUNT(1)
        OVER(PARTITION BY session ORDER BY step
          ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT_ROW)
        AS cum_count
    FROM form_log_with_fallout_step
  )
  , fallout_log AS (
    -- 폴아웃 리포트에 필요한 정보 추출하기
    SELECT
      session
      , step
      , path
    FROM
      form_log_with_mod_fallout_step
    WHERE
      -- 세션 내부에서 단계 순서가 1인 URL에 접근하는 경우
      min_step = 1
      -- 현재 단계 순서가 해당 단계에 도착할 때까지의 누계 단계 수와 같은 경우
      AND step = cum_count
      -- 직전 단계의 첫 접근 시간이 NULL 또는 현재 단계의 최종 접근 시간보다 앞인 경우
      AND (lag_min_stemp IS NULL OR max_stamp >= lag_min_stamp)
  )
  SELECT
    step
    , path
    , COUNT(1) AS count
    -- '단계 순서 = 1'인 URL로부터의 이동률
    , 100.0 * COUNT(1)
      / FIRST_VALUE(COUNT(1))
        OVER(ORDER BY step ASC ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)
      AS first_trans_rate
    -- 직전 단계로부터의 이동률
    , 100.0 * COUNT(1)
      / LAG(COUNT(1)) OVER(ORDER BY step ASC)
      -- sparkSQL의 경우 LAG 함수에 프레임 지정 필요
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

### 원포인트
- 위 리포트는 **입력 화면**에서
  - 사용자가 **입력의 의사** 유무는 판별할 수 없음
    - `실수`로 버튼을 눌러 화면 이동이 된 경우도 있기 때문
- **입력 의사**를 확인하고 싶다면,
  - **최초 입력 항목**을 클릭할 때 또는 **최초 입력**시에 `javascript`를 사용해서 **추가 로그**를 전송해야 함
- 위와 같이 할 경우,
  - 확실하게 `입력 화면 출력 ~ 입력 시작 ~ 확인화면 출력 ~ 완료화면 출력`까지의 **폴아웃 리포트**를 작성할 수 있음

## 3. 입력 양식 직귀율 집계하기
- 개념
  - SQL : `SUM(CASE ~)`, `SIGN`
  - 분석 : 입력 양식 직귀율
- 입력 양식 직귀율
  - `입력 화면`으로 이동한 후
  - `입력 시작`, `확인 화면`, `오류 화면`으로 이동한 로그가 없는 상태의 레코드를 센 것
- 입력 양식 직귀 수가 높다면
  - 사용자가 입력을 중간에 **포기**할만큼 `입력 항목`이 많다거나,
  - **출력 레이아웃**이 난잡한다등의 이유가 존재할 수 있음
- 추가로 **입력 화면**으로 이동하는 과정에서
  - 사용자의 **모티베이션 환기**가 충분하지 않으면,
  - 아무 이유 없이 양력 양식이 출력되는 인상을 주는 문제도 존재

### TABLE.16.2. 입력 양식 직귀율
>**일자**|**입력 양식 방문 횟수**|**입력 양식 직귀 수**|**입력 양식 직귀율**
>-----|-----|-----|-----
>2016.11.1| 1,382 | 692 |50.10%
>2016.11.2| 1,463 | 677 |46.30%
>2016.11.3| 1,311 | 568 |45.60%
>2016.11.4| 1,289 | 569 |44.10%
- **입력 화면 방문 횟수**를 분모에 두고,
  - **입력 양식 직귀 수**의 비율을 집계하면 계산 가능

### CODE.16.3. 입력 양식 직귀율을 집계하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  WITH
  form_with_progress_flag AS (
    SELECT
      -- PostgreSQL, Hive, Redshift, SparkSQL의 경우 substring으로 날짜 부분 추출
      substring(stamp, 1, 10) AS dt
      -- PostgreSQL, Hive, BigQuery, SparkSQL의 경우 substr 사용
      substr(stamp, 1, 10) AS dt
      
      , session
      -- 입력 화면으로부터의 방문 플래그 계산
      , SIGN(
          SUM(CASE WHEN path IN ('/regist/input') THEN 1 ELSE 0 END)
      ) AS has_input
      -- 입력 확인 화면 또는 완료 화면으로의 방문 플래그 계산하기
      , SIGN(
        SUM(CASE WHEN path IN('/regsit/confirm', '/regist/complete') THEN 1 ELSE 0 END)
      ) AS has_progress
    FROM form_log
    GROUP BY
      -- PostgreSQL, Redshift, BigQuery의 경우
      -- SELECT 구문에서 정의한 별칭을 GROUP BY에 지정할 수 있음
      dt, session
      -- PostgreSQL, Hive, Redshift, SparkSQL의 경우
      -- SELECT 구문에서 별칭을 지정하기 이전의 식을 GROUP BY에 지정할 수 있음
      substring(stamp, 1, 10), session
  )
  SELECT
    dt
    , COUNT(1) AS input_count
    , SUM(CASE WHEN has_progress = 0 THEN 1 ELSE 0 END) AS bounce_count
    , 100.0 * AVG(CASE WHEN has_progrss = 0 THEN 1 ELSE 0 END) AS bounce_rate
  FROM
    form_with_progress_flag
  WHERE
    -- 입력 화면에 방문했던 세션만 추출하기
    has_input = 1
  GROUP BY
    dt
  ;
  ```
- 세션 별로
  - 입력 화면(`/regist/input`) 방문 횟수,
  - 확인 화면(`/regist/confirm`) 방문 횟수,
  - 완료 화면(`/regist/complete`) 방문 횟수
  - 를 각각 `SUM(CASE ~)` 구문으로 세고,
    - `SIGN` 함수를 사용하여 **플래그** 변환
- 그리고 입력 화면을 **방문 하고 있는 세션**이라는 조건을 사용한 뒤,
  - 마찬가지로 `SUM(CASE ~)` 구문으로 **직귀 수**를 계산하며,
  - `AVG(CASE ~)` 구문으로 **직귀율**을 계산

### 원포인트
- **페이지 열람 로그**를 사용하여, 앞의 리포트를 작성해도,
  - `입력 시작 후 이탈`인지, `입력 없이 이탈`인지는 판별 불가
- **16.2**와 같이
  - `처음 입력한 항목을 클릭한 때` / `처음 입력한 때`에 `js`로 로그를 송신해서 활용해야
  - 더 정밀하게 **직귀율**계산 가능 

## 4. 오류가 발생하는 항목과 내용 집계하기
- 개념
  - SQL : SUM 윈도 함수
- 입력 양식에서의 **오류율 집계**는 **16.1**에서 소개했으나,
  - 그 방법만으로는, 발생한 오류가 **어떤 원인**으로 발생했는지는 파악이 어려움
  - 원인 파악이 되어야 **적절한 대응**이 가능
- 입력 양식에서 **오류**가 발생한 경우에는 적절한 **오류 로그**를 남길 것
  - stamp, session, form, field, error_type, value
- 입력 양식을 입력할 때, **무엇을 입력했는지** 리포트 하기 위해
  - 입력된 문자열을 그대로 **로그**로 출력
- 하지만 입력 양식에는 **개인 정보**가 포함될 수 있으므로,
  - 개인 정보 관련 항목은 **입력된 문자열**을 직접 저장하지 않게 주의해야 함
- 필요에 따라 저장해야 할 경우에는 **관련 보안 정책**이 필요함
- 예시 : 로그 출력 결과를 집계하고, 어떤 항목에 어떤 오류가 발생했는지를 정리한 리포트의 예를 들면 다음과 같음
  >form|name|error\_type|count|share
  >:-----:|:-----:|:-----:|:-----:|:-----:
  >cart|eng|require|30|60%
  >cart|eng|not\_eng|10|20%
  >cart|email|require|10|20%
  >contact|email|require|3|60%
  >contact|tel|length\_less|2|40%

### CODE.16.4. 각 입력 양식의 오류 발생 장소와 원인을 집계하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
  ```sql
  SELECT
    form
    , field
    , error_type
    , COUNT(1) AS count
    , 100.0 * COUNT(1) / SUM(COUNT(1)) OVER(PARTITION BY form) AS share
  FROM
    form_error_log
  GROUP BY
    form, field, error_type
  ORDER BY
    form, count DESC
  ```
- 입력 양식의 종류, 입력 항목, 오류 종류로 집약하고
  - 오류 수와 전체에서 차지하는 비율을 계산
- 전체에서 차지하는 비율을 구하기 위해 `SUM 윈도 함수`로 `전체 오류 수의 모수`를 구함

### 원포인트
- 오류 발생율이 높은 항목부터 **16강** 첫 소절에서 설명한 대책을 실시하면 좋음
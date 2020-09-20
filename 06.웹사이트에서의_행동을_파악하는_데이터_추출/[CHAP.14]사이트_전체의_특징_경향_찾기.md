# 14. 사이트 전체의 특징/경향 찾기
- 웹 사이트 전체의 특징과 경향을 찾기 위한 리포트와 SQL
- **접근 해석 도구**를 도입한 서비스에서는 이미 파악한 값일 수 있음
- **접근 분석 도구**로 집계한 결과 및 리포트도, 어느정도는 필터링이 가능하겠으나,
  - 제공되는 기능 이외의 리포트는 작성할 수 없음
  - 리포트의 설계가 **접근 분석 도구**의 기능으로 인해, 제한 될 수 있음
- 위 제한을 넘어, 자유롭게 리포트를 설계할 수 있다는 점이
  - 빅데이터 기반을 도입하는 장점
- **접근 분석 도구**에서 제공되는 결과에서
  - 추가로 복잡한 조건으로 **드릴 다운**해서 상세를 찾으려면,
  - 접근 분석 도구에서는 **최저한의 상태**로 집계할 항목을 추출해야 함
- 또한, 접근 분석 도구에서 활용할 수 없는
  - **사용자 데이터** 또는 **상품 데이터**등의 업무 데이터와 조합하면,
  - 작성할 수 없었던 리포트 작성 가능

### 샘플 데이터
- access_log : stamp / short_session / long_session / url / referer
- purchase_log : stamp / short_session / long_session / purchase_id / amount

## 1. 날짜별 방문자 수 / 방문 횟수 / 페이지 뷰 집계하기
- `COUNT`, `COUNT(DISTINCT ~ )`
- 방문자 수, 방문 횟수, 페이지 뷰
- 웹 사이트에서는
  - `방문자 수`, `방문 횟수`, `페이지 뷰 집계`가 기본
  - 각 지표가 무엇을 집계하는지 이해하고 활용할 것
- 각 용어의 의미
  - 방문자 수
    - 브라우저를 꺼도 사라지지 않는 **쿠키의 유니크 수**
    - `e.g. 한명의 사용자가 1일에 3회 방문에도, 1회로 집계`
  - 방문 횟수
    - 브라우저를 껏을 때 **사라지는** 쿠키의 유니크 수
    - `e.g. 한명의 사용자가 1일에 3회 방문하면, 3회로 집계`
  - 페이지 뷰
    - 페이지를 출력한 로그의 줄 수
- **1회 방문당 페이지 뷰**를 날짜별로 집계하기
  - 일자 / 방문자 수 / 방문 횟수 / 페이지 뷰 / 1번 방문당 페이지 뷰

### CODE.14.1. 날짜별 접근 데이터를 집계하는 쿼리
- `PostgreSQL`, `Hive`, `Redshift`, `BigQuery`, `SparkSQL`
```sql
SELECT
  -- PostgreSQL, Hive, Redshift, SparkSQL, subString으로 날짜 부분 추출
  substring(stamp, 1, 10) as dt
  -- PostgreSQL, Hive, BigQuery, SparkSQL, substr 사용
  , substr(stamp, 1, 10) AS dt
  -- 쿠키 계산하기
  , COUNT(DISTINCT long_session) AS access_users
  -- 방문 횟수 계산
  , COUNT(DISTINCT short_session) AS access_count
  -- 페이지 뷰 계산
  , COUNT(*) AS page_view

  -- 1인당 페이지 뷰 수
  -- PostgreSQL, Redshift, BigQuery, SparkSQL, NULLIF 함수 사용
  , 1.0 * COUNT(*) / NULLIF(COUNT(DISTINCT long_session), 0) AS pv_per_user
  -- Hive, NULLIF 함수 대신 CASE 사용
  , 1.0 * COUNT(*)
      / CASE
          WHEN COUNT(DISTINCT long_session) <> 0 THEN COUNT(DISTINCT long_session)
        END
    AS pv_per_user
FROM
  access_log
GROUP BY
  -- PostgreSQL, Redshift, BigQuery
  -- SELECT 구문에서 정의나 별칭을 GROUP BY에 지정할 수 있음
  dt
  -- PostgreSQL, Hive, Redshift, SparkSQL
  -- SELECT 구문에서 별칭을 지정하기 이전의 식을 GROUP BY에 지정할 수 있음
  substring(stamp, 1, 10)
ORDER BY
  dt
;
```
- 사이트에서 로그인 기능을 제공하는 경우
  - 추가로 **로그인 UU**와 **비로그인 UU**도 동시에 집계하면 좋음
    - **CHAP.11.1** 참고

### 원포인트
- **페이지 뷰** 정보만 필요하다 해도,
  - 그 이외의 지표를 집계할 때마다, 별도의 `SQL`을 작성해서, 실행하기엔 번거로움
- 따라서 한번에 모든 정보를 추출할 수 있는 SQL을 만들어 사용하기
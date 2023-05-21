## BigQuery 시계열 분석 - Fan Chart Analysis

지난 핸즈온에서 시계열 분석의 하나인 `Z-Chart` 분석을 다루었다. 이번에는 시계열 데이터인 날짜별 매출액을 `Fan Chart` 시각화하여 상품의 매출 증가율을 비교 분석하는 과정을 다뤄보자.

### Fan Chart Analysis

> `Fan Chart`란 기준 시점의 수치를 100%로 두고 그 이후의 시간에 따른 변동을 기준 시점에 대한 백분율로 표시한 꺾은선 그래프를 말한다.  시각화된 모습이 부채(fan)를 펼친 듯한 모습을 띈다하여 `Fan Chart`라 불린다.

![](https://images.velog.io/images/jaytiger/post/bd4f4304-61f2-4295-96a2-223b66e3843f/fan_chart.png)

위의 `Fan Chart`를 통해 카테고리 어떤 카테고리의 제품군이 매출 성장률이 높은지 시각적으로 확인 가능하다.  

### 분석과정

`Fan Chart` 분석으로 상품 카테고리별 매출액의 변화 추이를 확인하기 위한 과정은 다음과 같다.

1. 데이터 준비 - 매출 이력 데이터 생성

2. Fan Chart 분석에 필요한 지표 (metric) 정의
   2-1. 기준시점 매출
   2-2. **기준시점 대비 매출 비율**

3. 지표에 대한 기준 (dimension) 설정
   3-1. month, category
   3-2. ~~date range~~

4. 분석 쿼리 작성
   4-1. 전처리- 일별/월별 집계
   4-2. 월별 집계에 사용할 년/월 추출 (날짜 표현에서)
   4-3. 집계/분석 함수를 사용하여 정의된 지표 산출

### 데이터 준비

지난 번과 동일하게 참고 도서에서 제공하는 예시 데이터를 아래와 같이 빅쿼리에 적재 가능하도록 가공하였다. 실행 가능한 적재 쿼리는 블로그 하단에 덧붙여진 내용을 참고하자.

```sql
-- Pseudo Query
DECLARE purchase_detail_log_raw ARRAY<STRUCT<
    dt            STRING,
    order_id      INT64,
    user_id       STRING,
    item_id       STRING,
    price         INT64,
    category      STRING,
    sub_category  STRING
>>
DEFAULT [
  ('2017-01-18', 48291, 'usr33395', 'lad533', 37300,  'ladys_fashion', 'bag'),
  ('2017-01-18', 48291, 'usr33395', 'lad329', 97300,  'ladys_fashion', 'jacket'),
  ...
];
CREATE OR REPLACE TABLE learning_club.purchase_detail_log2 AS
SELECT l.* FROM UNNEST(purchase_detail_log_raw) l;
 
```

### 분석 쿼리

분석을 위한 첫 번째 단계로 `4-1`의 일별/월별 매출액을 집계해 보자.

상세 매출 정보를 담고 있는 테이블 `purchase_detail_log2`에서 일별 매출액을 집계하는 쿼리는 다음과 같다.

```sql
SELECT dt,
       category,
       SUM(price) AS amount,
  FROM learning_club.purchase_detail_log2
 GROUP BY 1, 2
```

제품 카테고리별 매출액 변화 추이를 보고자 하기 때문에 `GROUP BY`의 차원에 해당하는 컬럼에 날짜 차원인 `dt`와 제품 카테고리의 `category` 2개를 지정하여`SUM(price)` 집계 함수로 매출액을 산출한다.

다음은 일별 매출액에서 월별 매출액을 산출하는 쿼리이다.

```sql
-- 일별 매출액 산출
WITH daily_purchase AS (
  SELECT dt,
         category,
         SUBSTR(dt, 1, 4) AS year,
         SUBSTR(dt, 6, 2) AS month,
         SUBSTR(dt, 9, 2) AS day,
         SUM(price) AS amount,
    FROM learning_club.purchase_detail_log2
   GROUP BY 1, 2
)
SELECT year || '-' || month AS year_month,
       category,
       SUM(amount) AS amount,
  FROM daily_purchase
 GROUP BY 1, 2
```

`GROUP BY`에 날짜가 이번에는 하루가 아닌 월 단위가 되어야 하므로 날짜를 조작하여 년/월/일을 아래와 같이 `daily_purchase` 서브쿼리에서 미리 전처리 시켜놓았다.  메인쿼리를 간결하게 작성하기 위해서이다.

```
  SUBSTR(dt, 1, 4) AS year,
  SUBSTR(dt, 6, 2) AS month,
  SUBSTR(dt, 9, 2) AS day,
```

매출 이력이 여러 해에 걸쳐 수집된 경우 단순히 `month`로 집계를 하게 되면 2017년과 2018년의 월별 매출액이 같이 합산이 될 수 있으므로 월별 집계의 차원으로는 아래와 같이 년/월을 묶어서 사용해야 원하는 결과를 얻을 수 있다.

> year || '-' || month AS year_month,

여기서 `||`은 문자열 접합 연산자로 `CONCAT()` 함수와 동일한 역할을 한다.


```sql
-- 월별 매출액 산출 
WITH daily_purchase AS (
  SELECT dt,
         category,
         SUBSTR(dt, 1, 4) AS year,
         SUBSTR(dt, 6, 2) AS month,
         SUBSTR(dt, 9, 2) AS day,
         SUM(price) AS amount,
    FROM learning_club.purchase_detail_log2
   GROUP BY 1, 2
)
SELECT year || '-' || month AS year_month,
       category,
       SUM(amount) AS amount,
  FROM daily_purchase
 GROUP BY 1, 2;
```


### 첫 번째 Fan Chart 마트 쿼리

월별 매출액이 산출되었으니 기준 월을 정하고 기준 월 대비 각 월의 매출 비율을 산출해 보도록 하자.

> 기준 월(year_month)은 매출 이력을 수집하기 시작한 첫 번째 달로 정의

```sql
WITH daily_purchase AS (
  SELECT dt,
         category,
         SUBSTR(dt, 1, 4) AS year,
         SUBSTR(dt, 6, 2) AS month,
         SUBSTR(dt, 9, 2) AS day,
         SUM(price) AS amount,
    FROM learning_club.purchase_detail_log2
   GROUP BY 1, 2
),
monthly_purchase AS (
  SELECT year || '-' || month AS year_month,
         category,
         SUM(amount) AS amount,
    FROM daily_purchase
   GROUP BY 1, 2
)
SELECT 
    -- 어떤 로직이 들어가야 할까?
  FROM monthly_purchase
 ORDER BY 1;
```

첫번째 관문은 원하는 지표를 산출하기 위해서는 필요한 `기준 월` 의 값을 어떻게 가져올 수 있을까에 대한 고민이다.

**1. Self-Join 을 이용하여 기준 월의 매출액 연결**

```sql
--Self-Join 사용
SELECT t1.year_month,
       t1.category,
       t1.amount,
       t2.amount AS base_amount,
       ROUND(t1.amount / t2.amount * 100, 1) AS rate
  FROM monthly_purchase t1 LEFT JOIN (
    SELECT category, year_month, amount 
      FROM monthly_purchase 
     WHERE TRUE 
   QUALIFY ROW_NUMBER() OVER (PARTITION BY category ORDER BY year_month) = 1
  ) t2 
 USING (category)
 ORDER BY 1, 2;
``` 

셀프 조인은 동일 테이블의 다른 행에 존재하는 값으로 컬럼을 늘려가는 목적으로 주로 사용된다.

`t2`  서브쿼리에서 각 카테고리를 날짜 순으로 정렬시켜 가장 최근의 행만 남긴 후에 조인을 통해 메인쿼리에서 기준 월의 매출액을 참조할 수 있도록 하였다.

**2. 분석(윈도우) 함수를 이용한 방법**

```sql
SELECT year_month,
       category,
       amount,
       FIRST_VALUE(amount) OVER (
         PARTITION BY category ORDER BY year_month
       ) AS base_amount,
       amount / 
       FIRST_VALUE(amount) OVER (
         PARTITION BY category ORDER BY year_month
       ) * 100 AS rate,
  FROM monthly_purchase
 ORDER BY 1, 2;
```

동일 테이블의 다른 행에 위치한 값으로 컬럼을 확장할 때 셀프 조인외에 윈도우 함수의 일종인 네비게이션 함수를 이용하는 방법도 있다.

여기서 사용된 `FIRST_VALUE()`는 네비게이션 함수로 `category`로 구분된 파티션에서 `year_month`을 오름차순으로 정렬한 후 기준 월에 해당하는 첫번째 행에서 매출값을 가져온다.

#### 첫번째 마트 쿼리

셀프 조인보다는 가급적 윈도우 함수를 쓰는 것이 성능상 유리하다. 윈도우 함수를 적용하여 첫번째 마트 쿼리를 완성해 보자.

```sql
WITH daily_purchase AS (
  SELECT dt,
         category,
         SUBSTR(dt, 1, 4) AS year,
         SUBSTR(dt, 6, 2) AS month,
         SUBSTR(dt, 9, 2) AS day,
         SUM(price) AS amount,
    FROM learning_club.purchase_detail_log2
   GROUP BY 1, 2
),
monthly_purchase AS (
  SELECT year || '-' || month AS year_month,
         category,
         SUM(amount) AS amount,
    FROM daily_purchase
   GROUP BY 1, 2
)
SELECT year_month,
       category,
       amount,
       FIRST_VALUE(amount) OVER (
         PARTITION BY category ORDER BY year_month
       ) AS base_amount,
       amount / 
       FIRST_VALUE(amount) OVER (
         PARTITION BY category ORDER BY year_month
       ) * 100 AS rate,
  FROM monthly_purchase
 ORDER BY 1, 2;
```

실행 결과는 다음과 같다.

![](https://images.velog.io/images/jaytiger/post/575c635a-f9fc-49c4-91b3-9e49d6e89c48/fan_chart_mart_1.png)

두 번째 관문은 만들어진 마트 테이블로 Fan Chart를 어떻게 그릴 수 있을까이다.

이 글의 도입부에 있는 Fan Chart를 다시 살펴보자.

가로 축에는 `날짜` 차원(dimension)이 지정되어 있고 세로축에는 `매출 비율`이라는 지표(metric)가 보여지고 있다. 그런데 꺾은 선이 하나가 아니고 각 카테고리별로 하나의 꺾은 선이 그려지고 있다.

이 의미는 카테고리의 각각의 값들인 `book`, `cd`, `food` 등등이 하나의 지표항목이 되고 각각 지표값을 갖는다는 의미이다. (_지표 자체도 하나의 차원을 이룬다_)

시각화를 위해서는 우선 차원(dimension)과 지표(metric)에 대한 이해가 필요하다.  자세한 설명보다는 각 차원항목과 지표항목들은 테이블에서 각각 하나의 컬럼에 대응된다는 점만 우선 기억하자.

예를 들어, 아래와 같은 집계 쿼리가 있을 경우 `col1`과 `col2` 컬럼은 각각 차원항목이 되고, 집계함수의 결과인 `metric_1`과 `metric_2`는 각각 지표항목이 된다.  

차원 항목이나 지표항목 모두 하나의 컬럼을 차지하고 있다.

```sql
-- 집계 쿼리에서의 차원항목과 지표항목
SELECT col1 AS dimension_1,
       col2 AS dimesnion_2,
       SUM(col3) AS metric_1,
       AVG(col3) AS metric_2,
  FROM some_table
 GROUP BY 1, 2
```

다시 첫 번째 마트 테이블로 돌아와 보자.

현재 마트 테이블은 Fan Chart로 시각화하기에는 적절하지 않은 구조를 갖는다.

앞서 설명에서처럼 Fan Chart를 그리려면 카테고리의 각 항목들이 하나의 지표항목으로 분리되어 하나의 컬럼을 차지하고 있어야 한다. 

하지만 첫 번째로 만든 마트는 카테고리들이 컬럼으로 구분되어 있지 않고 `Stacked Data` 형태를 가지고 있어 Fan Chart 시각화에 적합한 테이블 구조로 변경할 필요가 있다.

엑셀을 다루면서 많이 듣던 `Pivot` 이 등장할 차례이다.


![](https://images.velog.io/images/jaytiger/post/f7555fba-2831-41ec-bbe3-df8e6a3e5024/fan_chart_analysis1.png)


### 두 번째 Fan Chart 마트 쿼리

![pivot](https://pandas.pydata.org/pandas-docs/version/0.25.3/_images/reshaping_pivot.png)

BigQuery는 비교적 최근에 Workaround Query 도움없이 `PIVOT` 을 직접적으로 수행할 수 있도록 기능이 제공되고 있다. `PIVOT` 연산자를 사용하여 시각화에 적합한 마트 구조로 Pivoting을 수행해 보자.

```sql
WITH daily_purchase AS (
  SELECT dt,
         category,
         SUBSTR(dt, 1, 4) AS year,
         SUBSTR(dt, 6, 2) AS month,
         SUBSTR(dt, 9, 2) AS day,
         SUM(price) AS amount,
    FROM learning_club.purchase_detail_log2
   GROUP BY 1, 2
),
monthly_purchase AS (
  SELECT year || '-' || month AS year_month,
         category,
         SUM(amount) AS amount,
    FROM daily_purchase
   GROUP BY 1, 2
),
fan_chart_mart AS (
SELECT year_month,
       category,
       amount,
       FIRST_VALUE(amount) OVER (
         PARTITION BY category ORDER BY year_month ROWS UNBOUNDED PRECEDING
       ) AS base_amount,
       ROUND(
         amount / FIRST_VALUE(amount) OVER (
           PARTITION BY category ORDER BY year_month ROWS UNBOUNDED PRECEDING
         ) * 100, 1
       ) AS rate,
  FROM monthly_purchase
 ORDER BY 1
)
SELECT * 
  FROM (SELECT year_month, category, rate FROM fan_chart_mart) 
 PIVOT (
   SUM(rate) FOR category IN (
     'book', 'cd', 'dvd', 'food', 'game', 'ladys_fashion', 'mens_fashion'
   )
 );
```

`PIVOT` 연산자의 사용은 직관적이지는 않다. 필요한 내용만 우선 살펴보자.

`SUM(rate) FOR category IN ( ... )` 은 `category` 컬럼에 저장된 값들 중에 `IN` 리스트 내에 명시된 값들을 각각 하나의 컬럼으로 만들면서 해당 컬럼의 값으로 `SUM(rate)`를 저장하라는 의미이다.

![](https://images.velog.io/images/jaytiger/post/7ab73be8-6ebc-4329-a03b-2ac1c1aa6964/fan_chart_mart_2.png)

Pivoting된 마트 테이블에서는 카테고리 각각의 항목들이 하나의 컬럼으로 분리되어 있어 Fan Chart 시각화를 할 수 있다.

`Data Explore`의 Combo Chart에서 `Metric` 에 카테고리 항목들을 모두 추가해주면 아래의 결과를 얻을 수 있다.

![](https://images.velog.io/images/jaytiger/post/cbbe9d74-aa80-4e72-b9c9-6d058dc87da1/fan_chart2.png)

---
#### [참고] 매출 상세 데이터
```sql
DECLARE purchase_detail_log_raw ARRAY<STRUCT<
    dt            STRING,
    order_id      INT64,
    user_id       STRING,
    item_id       STRING,
    price         INT64,
    category      STRING,
    sub_category  STRING
>>
DEFAULT [
  ('2017-01-18', 48291, 'usr33395', 'lad533', 37300,  'ladys_fashion', 'bag'),
  ('2017-01-18', 48291, 'usr33395', 'lad329', 97300,  'ladys_fashion', 'jacket'),
  ('2017-01-18', 48291, 'usr33395', 'lad102', 114600, 'ladys_fashion', 'jacket'),
  ('2017-01-18', 48291, 'usr33395', 'lad886', 33300,  'ladys_fashion', 'bag'),
  ('2017-01-18', 48292, 'usr52832', 'dvd871', 32800,  'dvd'          , 'documentary'),
  ('2017-01-18', 48292, 'usr52832', 'gam167', 26000,  'game'         , 'accessories'),
  ('2017-01-18', 48292, 'usr52832', 'lad289', 57300,  'ladys_fashion', 'bag'),
  ('2017-01-18', 48293, 'usr28891', 'out977', 28600,  'outdoor'      , 'camp'),
  ('2017-01-18', 48293, 'usr28891', 'boo256', 22500,  'book'         , 'business'),
  ('2017-01-18', 48293, 'usr28891', 'lad125', 61500,  'ladys_fashion', 'jacket'),
  ('2017-01-18', 48294, 'usr33604', 'mem233', 116300, 'mens_fashion' , 'jacket'),
  ('2017-01-18', 48294, 'usr33604', 'cd477' , 25800,  'cd'           , 'classic'),
  ('2017-01-18', 48294, 'usr33604', 'boo468', 31000,  'book'         , 'business'),
  ('2017-01-18', 48294, 'usr33604', 'foo402', 48700,  'food'         , 'meats'),
  ('2017-01-18', 48295, 'usr38013', 'foo134', 32000,  'food'         , 'fish'),
  ('2017-01-18', 48295, 'usr38013', 'lad147', 96100,  'ladys_fashion', 'jacket'),
  ('2017-02-18', 48291, 'usr33395', 'lad533', 37300,  'ladys_fashion', 'bag'),
  ('2017-02-18', 48291, 'usr33395', 'lad329', 97300,  'ladys_fashion', 'jacket'),
  ('2017-02-18', 48291, 'usr33395', 'lad102', 114600, 'ladys_fashion', 'jacket'),
  ('2017-02-18', 48291, 'usr33395', 'lad329', 97300,  'ladys_fashion', 'jacket'),
  ('2017-02-18', 48291, 'usr33395', 'lad886', 33300,  'ladys_fashion', 'bag'),
  ('2017-02-18', 48292, 'usr52832', 'dvd871', 22800,  'dvd'          , 'documentary'),
  ('2017-02-18', 48292, 'usr52832', 'gam167', 12000,  'game'         , 'accessories'),
  ('2017-02-18', 48292, 'usr52832', 'gam167', 16000,  'game'         , 'accessories'),
  ('2017-02-18', 48292, 'usr52832', 'dvd871', 12800,  'dvd'          , 'documentary'),
  ('2017-02-18', 48292, 'usr52832', 'lad289', 57300,  'ladys_fashion', 'bag'),
  ('2017-02-18', 48293, 'usr28891', 'out977', 28600,  'outdoor'      , 'camp'),
  ('2017-02-18', 48293, 'usr28891', 'boo256', 12500,  'book'         , 'business'),
  ('2017-02-18', 48293, 'usr28891', 'out977', 28600,  'outdoor'      , 'camp'),
  ('2017-02-18', 48293, 'usr28891', 'lad125', 61500,  'ladys_fashion', 'jacket'),
  ('2017-02-18', 48293, 'usr28891', 'boo256', 22500,  'book'         , 'business'),
  ('2017-02-18', 48294, 'usr33604', 'mem233', 216300, 'mens_fashion' , 'jacket'),
  ('2017-02-18', 48294, 'usr33604', 'cd477' , 15800,  'cd'           , 'classic'),
  ('2017-02-18', 48293, 'usr28891', 'boo256', 22500,  'book'         , 'business'),
  ('2017-02-18', 48294, 'usr33604', 'cd477' , 25800,  'cd'           , 'classic'),
  ('2017-02-18', 48294, 'usr33604', 'foo402', 48700,  'food'         , 'meats'),
  ('2017-02-18', 48293, 'usr28891', 'boo256', 22500,  'book'         , 'business'),
  ('2017-02-18', 48295, 'usr38013', 'foo134', 32000,  'food'         , 'fish'),
  ('2017-02-18', 48295, 'usr38013', 'foo134', 11000,  'food'         , 'fish'),
  ('2017-02-18', 48295, 'usr38013', 'lad147', 96100,  'ladys_fashion', 'jacket'),
  ('2017-03-18', 48291, 'usr33395', 'lad533', 37300,  'ladys_fashion', 'bag'),
  ('2017-03-18', 48291, 'usr33395', 'lad329', 97300,  'ladys_fashion', 'jacket'),
  ('2017-03-18', 48291, 'usr33395', 'lad886', 33300,  'ladys_fashion', 'bag'),
  ('2017-03-18', 48292, 'usr52832', 'dvd871', 32800,  'dvd'          , 'documentary'),
  ('2017-02-18', 48292, 'usr52832', 'gam167', 22000,  'game'         , 'accessories'),
  ('2017-02-18', 48292, 'usr52832', 'gam167', 16000,  'game'         , 'accessories'),
  ('2017-03-18', 48292, 'usr52832', 'dvd871', 12800,  'dvd'          , 'documentary'),
  ('2017-03-18', 48292, 'usr52832', 'lad289', 57300,  'ladys_fashion', 'bag'),
  ('2017-03-18', 48293, 'usr28891', 'out977', 28600,  'outdoor'      , 'camp'),
  ('2017-03-18', 48293, 'usr28891', 'boo256', 22500,  'book'         , 'business'),
  ('2017-03-18', 48293, 'usr28891', 'out977', 28600,  'outdoor'      , 'camp'),
  ('2017-03-18', 48293, 'usr28891', 'boo256', 22500,  'book'         , 'business'),
  ('2017-03-18', 48294, 'usr33604', 'mem233', 116300, 'mens_fashion' , 'jacket'),
  ('2017-03-18', 48294, 'usr33604', 'mem233', 86300,  'mens_fashion' , 'jacket'),
  ('2017-03-18', 48294, 'usr33604', 'cd477' , 22800,  'cd'           , 'classic'),
  ('2017-03-18', 48294, 'usr33604', 'cd477' , 22800,  'cd'           , 'classic'),
  ('2017-03-18', 48293, 'usr28891', 'boo256', 22500,  'book'         , 'business'),
  ('2017-03-18', 48295, 'usr38013', 'foo134', 32000,  'food'         , 'fish'),
  ('2017-03-18', 48295, 'usr38013', 'foo134', 32000,  'food'         , 'fish'),
  ('2017-03-18', 48295, 'usr38013', 'foo134', 32000,  'food'         , 'fish'),
  ('2017-03-18', 48295, 'usr38013', 'lad147', 96100,  'ladys_fashion', 'jacket'),
  ('2017-03-18', 48292, 'usr52832', 'gam167', 25000,  'game'         , 'accessories'),
  ('2017-04-18', 48291, 'usr33395', 'lad533', 35300,  'ladys_fashion', 'bag'),
  ('2017-04-18', 48291, 'usr33395', 'lad329', 91300,  'ladys_fashion', 'jacket'),
  ('2017-04-18', 48291, 'usr33395', 'lad102', 113600, 'ladys_fashion', 'jacket'),
  ('2017-04-18', 48291, 'usr33395', 'lad329', 92300,  'ladys_fashion', 'jacket'),
  ('2017-04-18', 48291, 'usr33395', 'lad886', 43300,  'ladys_fashion', 'bag'),
  ('2017-04-18', 48292, 'usr52832', 'dvd871', 35800,  'dvd'          , 'documentary'),
  ('2017-04-18', 48292, 'usr52832', 'gam167', 21000,  'game'         , 'accessories'),
  ('2017-04-18', 48292, 'usr52832', 'gam167', 11000,  'game'         , 'accessories'),
  ('2017-04-18', 48292, 'usr52832', 'dvd871', 32800,  'dvd'          , 'documentary'),
  ('2017-04-18', 48292, 'usr52832', 'lad289', 57300,  'ladys_fashion', 'bag'),
  ('2017-04-18', 48293, 'usr28891', 'out977', 28600,  'outdoor'      , 'camp'),
  ('2017-04-18', 48293, 'usr28891', 'boo256', 22500,  'book'         , 'business'),
  ('2017-04-18', 48293, 'usr28891', 'out977', 28600,  'outdoor'      , 'camp'),
  ('2017-04-18', 48293, 'usr28891', 'lad125', 61500,  'ladys_fashion', 'jacket'),
  ('2017-04-18', 48293, 'usr28891', 'boo256', 22500,  'book'         , 'business'),
  ('2017-04-18', 48294, 'usr33604', 'mem233', 106300, 'mens_fashion' , 'jacket'),
  ('2017-02-18', 48294, 'usr33604', 'mem233', 96300,  'mens_fashion' , 'jacket'),
  ('2017-04-18', 48294, 'usr33604', 'cd477' , 35800,  'cd'           , 'classic'),
  ('2017-04-18', 48293, 'usr28891', 'boo256', 32500,  'book'         , 'business'),
  ('2017-04-18', 48294, 'usr33604', 'cd477' , 35800,  'cd'           , 'classic'),
  ('2017-04-18', 48294, 'usr33604', 'foo402', 28700,  'food'         , 'meats'),
  ('2017-04-18', 48293, 'usr28891', 'boo256', 12500,  'book'         , 'business'),
  ('2017-04-18', 48295, 'usr38013', 'foo134', 12000,  'food'         , 'fish'),
  ('2017-04-18', 48295, 'usr38013', 'foo134', 22000,  'food'         , 'fish'),
  ('2017-04-18', 48295, 'usr38013', 'lad147', 86100,  'ladys_fashion', 'jacket')
];
CREATE OR REPLACE TABLE learning_club.purchase_detail_log2 AS
SELECT l.* FROM UNNEST(purchase_detail_log_raw) l;
```



## BigQuery - ABC 분석

ABC 분석은 재고관리에 사용되는 재고 분류 기법의 하나이다.[1][2]

제품의 중요도에 따라 등급을 매기고 그에 따른 판매 전략을 세울 때 활용된다.  중요도는 매출에 따라 구분되며 20%의 제품이 80%의 매출을 차지한다고 알려진 파레토 법칙도 ABC 분석에 근거하고 있다.

> ex. 누적매출 비중 70% - A 그룹, 70~90% B 그룹, 90~100% C 그룹

### ABC 분석 (Pareto Chart)

제품별 매출 데이터를 가지고 ABC 분석을 진행해 보자.

다음은 판매 제품의 카테고리까지 포함된 매출 상세 이력이다.

#### **매출 데이터 생성 - 구매 상세 이력**

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
  ('2017-01-18', 48295, 'usr38013', 'lad147', 96100,  'ladys_fashion', 'jacket')
];
CREATE OR REPLACE TABLE learning_club.purchase_detail_log AS
SELECT l.* FROM UNNEST(purchase_detail_log_raw) l;
```

#### **ABC 분석 마트**

ABC 분석에 필요한 지표들로 마트를 구성한 예는 다음과 같다.  마지막 등급 컬럼은 편의상 추가하였다.

■ 데이터 마트 형식 

|row|category|매출|구성비|구성비누계|등급|
|-------------|---------|---|---|-|---|
|1	|ladys_fashion|497400|57.76|57.76|A
|2	|mens_fashion|116300|13.51|71.27|B
|3	|food|80700|9.37|80.64|B
|4	|book|53500|6.21|86.85|B
|5	|dvd|32800|3.81|90.66|C
|6	|outdoor|28600|3.32|93.98|C
|7	|game|26000|3.02|97.0|C
|8	|cd|25800|3.0|100.0|C


각 컬럼의 의미와 산출 방법을 간략히 살펴보자.

0. `category` - 분석의 기준
1. `매출` - `purchase_detail_log` 에서 제품 category별 집계 함수 적용
2. `구성비` - `매출` / `총매출` * 100 (%)
   1. 분석(윈도우) 함수로 `총매출` 계산 - `SUM(..) OVER ()`
   2. 스칼라 서브쿼리를 통해서 `총매출` 계산 - `(SELECT SUM(...) FROM ...)` 
3. `구성비누계` - cumulative sum 분석함수 적용
   1. `PARTITION BY` ? 
   2. `ORDER BY` ?
   3. Window Frame ?


■ category별 매출

어렵지 않게 계산할 수 있다. 여기서는 한달 매출을 기준으로 제품 카테고리별 매출액이 산출된다.

```sql
SELECT category,
       SUM(price),
       -- ...
  FROM learning_club.purchase_detail_log
 WHERE dt BETWEEN '2017-01-01' AND '2017-01-31'
 GROUP BY 1
;
```

■ category별 구성비

```sql
CREATE OR REPLACE TABLE learning_club.abc_mart AS 
SELECT category,
       SUM(price) AS amount,
       ROUND(SUM(price) / (SUM(SUM(price)) OVER ()) * 100, 2) AS amount_rate,
       ROUND(SUM(SUM(price)) OVER (ORDER BY SUM(price) DESC ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) /
       SUM(SUM(price)) OVER () * 100, 2) AS cumulative_rate
  FROM learning_club.purchase_detail_log
 WHERE dt BETWEEN '2017-01-01' AND '2017-01-31'
 GROUP BY 1
 ORDER BY 2 DESC;
```

구성비를 구하기 위해서는 우선 전체 매출액이 필요하다. 전체 매출액은 분석 함수를 사용하여 다음과 같이 계산한다.

> SUM(SUM(price)) OVER ()

`OVER` 구문 안에 파티션이나 윈도우프레임을 지정하지 않았다. 이는 전체 테이블을 하나의 파티션으로 보겠다는 얘기이며 윈도우프레임도 파티션의 처음부터 끝까지를 하나로 간주하겠다는 의미이다.  이렇게 만들어진 윈도우프레임을 `SUM(SUM())`하면 전체 매출액이 구해진다.

안쪽의 `SUM()`은 `GROUP BY`에 의해서 수행되는 집계함수로 직전의 `category별 매출`에서의 `SUM(price)`와 동일하다. 반면, 바깥쪽의 `SUM()`은 집계함수가 아닌 분석함수로서 카테고리별 매출을 모두 합산하게 된다.

`cumulative_rate`는 조금 복잡하다. 분해해서 살펴보자.

- `ROUND(..., 2)`는 실수값의 소수점 2자리까지만 보여지도록 지정한다. 생략하면 다음이 남는다.

  > SUM(SUM(price)) OVER (ORDER BY SUM(price) DESC ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) 
  > /
  > SUM(SUM(price)) OVER () * 100

- 여기서 분모는 전체 매출액을 계산하고 있다.
  > SUM(SUM(price)) OVER ()

- 분자에 위치하고 있는 분석함수 구문에서는 두 가지를 살펴야 한다.
  - 카테고리별 매출에 해당하는 `SUM(price)`의 내림차순으로 파티션을 정렬(`ORDER BY`)시키고 있다.
    > ORDER BY SUM(price) DESC
  - 윈도우프레임의 시작을 가장 매출이 높은 첫번째 행에서부터 현재 행까지로 설정하고 있다. 결국 가장 높은 매출부터 현재까지의 매출을 누적합산한다.
    > ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW

요약하면 매출액을 기준으로 내림차순으로 카테고리를 정렬시킨 후에 해당 카테고리까지의 누적매출을 전체매출로 나눔으로써 구성비를 구할 수 있게 된다.

위의 쿼리를 수행시켜 마트를 만든 후에 Data Stuio로 시각화를 하게 되면 아래와 같은 레포트를 만들 수 있다.  

![](https://images.velog.io/images/jaytiger/post/c4163f2b-79d4-4ebb-bdb3-2ddec1ea1e39/abc_chart.png)
  
- [ABC Analysis - Data Studio Visualization](https://datastudio.google.com/reporting/a04b456e-21ce-4547-8938-4e110146a7f6) 

### 참고 자료
1. [ABC analysis - Wikipedia](https://en.wikipedia.org/wiki/ABC_analysis)
2. [https://brunch.co.kr/@beusable/191](https://brunch.co.kr/@beusable/191)


### [참고] 다차원분석을 위한 마트 - Cube
```sql
SELECT category, sub_category, SUM(price) AS amount,
  FROM learning_club.purchase_detail_log
 GROUP BY ROLLUP (1, 2)
 ORDER BY 1, 2 NULLS LAST; 
```

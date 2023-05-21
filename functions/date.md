# 날짜 및 시간 표현하기 (2)

## ■ 날짜(Date) 표현하기

날짜를 상수(리터럴, literal)로 표현하는 방법은 다음과 같습니다.  표준 날짜 형식(Cannonical Date Format) 앞에 `DATE` 키워드를 붙여줍니다.

```sql
-- canonical format : 'YYYY-M[M]-D[D]'
SELECT DATE '2023-05-16' date;

+------------+
|    date    |
+------------+
| 2023-05-16 |
+------------+
```

오늘 날짜를 표현하려면 `CURRENT_DATE()` 함수를 사용합니다. `CURRENT_TIME`과 마찬가지로 Timezone 정보를 추가로 넘길 수 있습니다.

```sql
SELECT CURRENT_DATE() current_date_utc,
       CURRENT_DATE('Asia/Seoul') current_date_seoul;

+------------------+--------------------+
| current_date_utc | current_date_seoul |
+------------------+--------------------+
| 2023-05-16       | 2023-05-16         |
+------------------+--------------------+
```

## ■ 날짜 형식으로 변환하기

날짜 표기는 국가 혹은 문화권에 따라 다양합니다. `년/월/일`의 순서를 사용하기도 하고, `월/일/년` 혹은 `일/월/년`이 되기도 합니다.  
또한 구분자도 `/`, `-`, `.` 이나 공백 등 여러가지를 사용합니다.  
(참고 - Wikipedia [Date format by country](https://en.wikipedia.org/wiki/Date_format_by_country))

- `12 May 2023`
- `12/05/23`
- `12/05/2023`
- `12-05-2023`

문자열로 표현된 날짜 형식을 `DATE` 형식으로 변환하기 위해서는  `PARSE_DATE()` 함수를 사용합니다.

여기서도 날짜의 각 구성요소를 구분하기 위한 `형식 문자열(Format String)` 의 이해가 중요합니다.

날짜와 관련된 주요 형식 요소들은 다음과 같습니다.

- `%Y` - 4자리 연도, `2023`
- `%y` - 2자리 연도, `23`
- `%m` - 월, `01`
- `%d` - 일, `25`
- `%b` - 월 이름 (영어), `Jan`

이들 형식 요소로 형식 문자열을 만들어 날짜 정보를 파싱하는 예시를 살펴보겠습니다.

```sql
SELECT PARSE_DATE('%d %b %Y', '12 May 2023'),
       PARSE_DATE('%d/%m/%y', '12/05/23'),
       PARSE_DATE('%d/%m/%Y', '12/05/2023'),
       PARSE_DATE('%d-%m-%Y', '12-05-2023');

+------------+------------+------------+------------+
|    f0_     |    f1_     |    f2_     |    f3_     |
+------------+------------+------------+------------+
| 2023-05-12 | 2023-05-12 | 2023-05-12 | 2023-05-12 |
+------------+------------+------------+------------+
```

## ■ 날짜 정보 활용하기

`DATE` 형식에는 이를 활용하기 위한 함수가 정의되어 있습니다.

- 추출 함수
  - `EXTRACT` - 구성 요소의 한 부분(part)를 추출
- 연산 함수
  - `DATE_ADD` - 날짜에 `INTERVAL`을 더함
  - `DATE_SUB` - 날짜에서 `INTERVAL`을 뺌
  - `DATE_DIFF` - 날짜의 차이를 계산
- 절단 함수
  - `DATE_TRUNC` - 시간의 해상도를 조절하여 정규화 시킴
  - `LAST_DAY` - `TRUNC`와 반대로 마지막 날짜로 정규화 시킴.


### ▷ 추출 함수

`EXTRCT()` 함수는 날짜/시간의 구성요소 한 부분(part)를 분리하는데 사용됩니다.

```sql
SELECT EXTRACT(YEAR FROM CURRENT_DATE) year,
       EXTRACT(MONTH FROM CURRENT_DATE) month,
       EXTRACT(QUARTER FROM CURRENT_DATE) quarter,
       EXTRACT(ISOWEEK FROM CURRENT_DATE) week_no;

+------+-------+---------+---------+
| year | month | quarter | week_no |
+------+-------+---------+---------+
| 2023 |     5 |       2 |      20 |
+------+-------+---------+---------+
```

### ▷ 날짜(Date) 연산

시간(Time) 연산에서 설명드렸던 것과 동일하게 날짜(Date) 형식에서도 아래의 연산이 가능합니다.

- `DATE` + `INTERVAL` --> `DATE_ADD`
- `DATE` - `INTERVAL` --> `DATE_SUB`
- `DATE` - `DATE` --> `DATE_DIFF`


예시를 살펴보겠습니다.

```sql
SELECT CURRENT_DATE date,
       DATE_ADD(CURRENT_DATE, INTERVAL 5 DAY) later,
       DATE_SUB(CURRENT_DATE, INTERVAL 2 DAY) before,
       DATE_DIFF(CURRENT_DATE, '2023-05-01', DAY) diff,
       CURRENT_DATE - '2023-05-01' `interval`;

+------------+------------+------------+------+--------------+
|    date    |   later    |   before   | diff |   interval   |
+------------+------------+------------+------+--------------+
| 2023-05-21 | 2023-05-26 | 2023-05-19 |   20 | 0-0 20 0:0:0 |
+------------+------------+------------+------+--------------+
```

>[참고] DATE_DIFF() 함수가 아닌 `-` 연산자를 사용하여 시간의 차이를 구하면 `INTERVAL` 형식의 결과를 돌려줍니다.

`DATE_ADD`와 `DATE_DIFF`는 `-` 빼기 연산을 사용해서도 동일한 연산이 가능합니다.

```sql
SELECT CURRENT_DATE + 5 AS later,
       CURRENT_DATE - 2 AS before,
       CURRENT_DATE + INTERVAL 2 WEEK AS two_weeks_later;

+------------+------------+---------------------+
|   later    |   before   |   two_weeks_later   |
+------------+------------+---------------------+
| 2023-05-26 | 2023-05-19 | 2023-06-04T00:00:00 |
+------------+------------+---------------------+
```

### ▷ 날짜(Date) 절단

`TIME`에서의 설명과 동일합니다.  `DATE` 에서도 `DATE_TRUNC()` 함수는 날짜의 해상도(resolution or granularity)를 낮추는 목적으로 사용됩니다.

일(day)단위의 관측값을 주(week) 단위놔 월(month) 단위로 날짜의 해상도를 낮추어 같은 집단으로 묶고자 할 때 유용합니다.


```sql
SELECT DATE_TRUNC(CURRENT_DATE, ISOWEEK) start_of_weekno,
       DATE_TRUNC(CURRENT_DATE, MONTH) start_of_month,
       DATE_TRUNC(CURRENT_DATE, QUARTER) start_of_quarter;

+-----------------+----------------+------------------+
| start_of_weekno | start_of_month | start_of_quarter |
+-----------------+----------------+------------------+
| 2023-05-15      | 2023-05-01     | 2023-04-01       |
+-----------------+----------------+------------------+
```

월(month)은 항상 1일부터 시작을 하지만 마지막 날은 월에 다르고 윤년에는 윤일이 더해저 2월의 마지막 날짜가 또 달라집니다.  
지금까지 소개된 함수만으로 월의 마지막 일을 알아내는 쿼리는 조금 복잡합니다.

```sql
SELECT DATE_TRUNC(DATE_ADD(CURRENT_DATE, INTERVAL 1 MONTH), MONTH) - 1;
```

`LAST_DAY()`를 사용하면 간단하게 월의 마지막 날짜를 계산할 수 있습니다. 또한, 한 주의 마지막 날짜와 분기의 마지막 날짜도 계산이 가능합니다.

```sql
SELECT LAST_DAY(CURRENT_DATE, MONTH) last_day_of_month,
       LAST_DAY(CURRENT_DATE, QUARTER) last_day_of_quarter;

+-------------------+---------------------+
| last_day_of_month | last_day_of_quarter |
+-------------------+---------------------+
| 2023-05-31        | 2023-06-30          |
+-------------------+---------------------+
```

#### ▷ 그 밖의 함수

`Epoch` 시간은 `1970-01-01 00:00:00` 협정세계시(UTC) 이후 경과한 시간을 나타냅니다.

`UNIX_DATE` 함수는 날짜에 대해서 '1970-01-01` Epoch 일로부터 경과한 일수만큼의 정수값을 돌려줍니다.

```
SELECT CURRENT_DATE AS current_date,
       UNIX_DATE(CURRENT_DATE) AS days_from_epoch;

+--------------+-----------------+
| current_date | days_from_epoch |
+--------------+-----------------+
| 2023-05-21   |           19498 |
+--------------+-----------------+
```

이를 통해서 우리는 각 날짜를 연속되는 자연수 중 하나에 대응시킬 수 있습니다. MoM(Month over Month), YoY(Year of Year) 지표나 Moving Average 등을 계산할 때 사용되는 윈도우 함수(또는 분석 함수)와 관련하여 이 개념은 중요하게 다루어집니다.

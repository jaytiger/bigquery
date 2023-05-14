# 날짜 및 시간 표현하기

## TIME, DATE and DATETIME

#### ■ 시간(Time) 표현하기

시간을 상수(리터럴, literal)로 표현하는 방법은 다음과 같습니다.  표준 시간 형식(Cannonical Time Format) 앞에 `TIME` 키워드를 붙여줍니다.

```sql
-- canonical format : '[H]H:[M]M:[S]S[.DDDDDD]'
SELECT TIME '09:00:00' time, TIME '10:00:00.300' time_usec;

+----------+-----------------+
|   time   |    time_usec    |
+----------+-----------------+
| 09:00:00 | 10:00:00.300000 |
+----------+-----------------+
```

현재 시간(Time)을 표현하려면 `CURRENT_TIME()` 함수를 사용합니다. 함수 호출시 Timezone 정보를 추가로 넘길 수 있습니다. 서울의 현재 시간은 협정 세계시(UTC)와 `+09:00` 만큼 차이가 납니다.

```sql
SELECT CURRENT_TIME() current_time_utc,
       CURRENT_TIME('Asia/Seoul') current_time_in_seoul;

+------------------+-----------------------+
| current_time_utc | current_time_in_seoul |
+------------------+-----------------------+
| 09:22:31.617137  | 18:22:31.617137       |
+------------------+-----------------------+
```

#### ■ 시간(Time)으로 변환하기

전처리하는 과정에서 다양한 형식으로 표현된 시간(Time) 데이터를 만나게 됩니다.

- `09:10:50` - 정식 시간 표현 (Canonical Time Format)
- `09:00` - 시/분 까지만 표현
- `2:30:50 PM` - 12-hour 시계 표현

이를 `TIME` 형식으로 변환하기 위해서는 `PARSE_` 함수를 사용해야 한다. `TIME` 형식을 위해서는 `PARSE_TIME()` 가 있습니다.

사실 함수의 이름 보다는 문자열 표현에서 시간을 구성하는 요소를 추출하기 위한 `형식 문자열(Format String)` 의 이해가 더 중요합니다.

형식 문자열은 `%`과 알파벳으로 이루어진 `형식 요소(Format Element)`들로 구성되고 하나의 형식 요소는 시간 표현에서 대부분 하나의 구성 요소에 대응됩니다..  예를 들어, `%H` 는 시간 표현에서의 `시(Hour)`에 대응됩니다.

주요 형식 요소들은 다음과 같습니다.

- `%H` - 24 hour clock 에서의 시(hour, 00-23)
- `%I` - 12 hour clock 에서의 시(hour, 01-12)
- `%M` - 분(minute, 00-59)
- `%S` - 초(second, 00-59)
- `%p` - **AM**, **PM**

형식 요소로 형식 문자열을 만들어 문자열로 표현된 시간 정보를 파싱하는 예시를 살펴봅시다.

```sql
SELECT PARSE_TIME('%H:%M', '09:00'), -- 시/분 까지만 표현
       PARSE_TIME('%I:%M:%S %p', '2:30:50 PM'), -- 12-hour clock 표현

+----------+----------+
|   f0_    |   f1_    |
+----------+----------+
| 09:00:00 | 14:30:50 |
+----------+----------+
```

아래는 `AM`, `PM` 을 포함하는 12-hour clock 표현을 파싱하는 예제인데 `14:30:50 AM` 라는 존재하지 않는 시간을 24-hour clock의 시(hour)를 의미하는 `%H` 형식 요소를 사용하여 잘 못 파싱한 경우입니다.

```sql
-- 잘못된 사용 예시
SELECT PARSE_TIME('%H:%M:%S %p', '14:30:50 AM') -- 잘못된 시간 표현

+----------+
|   f0_    |
+----------+
| 14:30:50 |
+----------+
```

12-hour clock을 파싱하는 경우는 `%H` 대신 반드시 `%I` 를 사용해야 하며 `%I`를 사용하여 위의 쿼리를 수행할 경우 아래처럼 오류가 발생합니다.

>Failed to parse input string "14:30:50 AM"

>[참고] BigQuery 함수 앞에 `SAFE.` 접두어를 붙여주면 에러 발생시 수행 결과로 `NULL`을 리턴하며 수행중인 Query Job 은 실패하지 않고 계속 진행이 됩니다.

```sql
SELECT SAFE.PARSE_TIME('%I:%M:%S %p', '14:30:50 AM');

+--------+
|  f0_   |
+--------+
| null   |
+--------+
```

#### ■ 시간(Time) 정보 활용하기

`TIME` 형식으로 변환된 데이터는 `TIME` 형식에 정의된 여러가지 함수를 사용할 수 있습니다.  `TIME` 형식 뿐만 아니라 `DATE`, `DATETIME`과 `TIMESTAMP` 형식에도 공통적으로 다음의 함수들이 정의되어 있습니다.

- 추출 함수
  - `EXTRACT` - 구성 요소의 한 부분(part)를 추출
- 연산 함수
  - `TIME_ADD` - 시간에 `INTERVAL`을 더함
  - `TIME_SUB` - 시간에서 `INTERVAL`을 뺌
  - `TIME_DIFF` - 시간의 차이를 계산
- 절단 함수
  - `TIME_TRUNC` - 시간의 해상도를 조절하여 정규화 시킴


##### ▷ 추출 함수

`EXTRCT()` 추출 함수는 시간의 구성요소 중 한 부분(part)를 분리하는데 사용됩니다.  시간은 `HOUR`, `MINUTE`, `SECOND` 등으로 구성되는데 이를 개별적으로 추출해서 연산에 활용할 수 있습니다.

```sql
SELECT EXTRACT(HOUR FROM TIME '15:30:00') hour;

+------+
| hour |
+------+
|   15 |
+------+
```

##### ▷ 시간(Time) 연산

날짜 시간 형식의 덧셈에서 다음의 문장을 생각해 봅시다.

> 1. `오늘`에 `내일`을 더한다.
> 2. `오늘`에 `하루`를 더한다.
> 3. `오늘`에서 `하루`를 뺀다.
> 4. `어제`에서 `오늘`을 뺀다.

첫 번째 문장은 의도를 알 수 없는 문장입니다. 반면 두 번째 문장은 `내일`, 세 번째 문장은 `어제` 라는 이라는 결과를 생각할 수 있는 문장입니다. 마지막 문장은 두 날짜의 차이를 묻고 있습니다.

빅쿼리의 날짜 시간 표현으로 설명하면, `오늘`과 `내일`은 빅쿼리의 날짜시간 형식이고, `하루`는 이전 장에서 나열했던 5 가지 날짜시간 데이터 형식 중의 하나인 `INTERVAL` 형식입니다.

위 문장에서 보듯이 빅쿼리에서도 `TIME` + `TIME` 은 정의되지 않은 연산이고, `TIME` + `INTERVAL`, `TIME` - `INTERVALE` 및 `TIME` - `TIME` 연산은 의미있는 결과를 돌려줍니다.


`TIME_ADD`와 `TIME_SUB`는 `TIME`과 `INTERVAL` 2개를 인자로 넘겨받아 연산을 수행한다.  `TIME_DIFF`는 2개의 `TIME` 값을 인자로 받아 그 차이를 계산해 줍니다.

예시를 통해서 살펴봅시다.

```sql
SELECT TIME '15:30:00' original_time,
       TIME_ADD('15:30:00', INTERVAL 10 MINUTE) later,
       TIME_SUB('15:30:00', INTERVAL 3 HOUR) before,
       TIME_DIFF('15:30:00', '15:00:00', MINUTE) diff;

+---------------+----------+----------+------+
| original_time |  later   |  before  | diff |
+---------------+----------+----------+------+
| 15:30:00      | 15:40:00 | 12:30:00 |   30 |
+---------------+----------+----------+------+
```

>TIME_DIFF() 함수가 아닌 `-` 연산자를 사용하여 시간의 차이를 구하면 `INTERVAL` 형식의 결과를 돌려줍니다. `INTERVAL` 타입 설명시에 보충토록 하겠습니다.

----

##### ▷ 시간(Time) 절단

`DATE_TRUNC()` 함수는 시간의 해상도(resolution or granularity)를 낮추는 목적으로 사용됩니다.

해상도를 낮춘다는 의미는 `TIME 14:30:35` 처럼 초단위까지 세밀하게 측정된 시간을 분단위 혹은 그 보다 더 뭉뚱그린 시간단위로만 사용하겠다는 것입니다.  분단위로 해상도를 낮추면 `TIME 14:30:00` 으로 `14:30:00` ~ `14:30:59` 사이의 모든 값들이 `14:30:00` 하나로 대표되는 것입니다.  초(second)단위 해상도 였던 것이 분(minute) 단위 해상도가 된 것입니다.

날짜 시간의 절단 함수는 이처럼 특정 기간내 수집된 관측값들을 하나의 대표 시간의 값으로 묶을 때 유용하게 사용됩니다.

```sql
SELECT TIME_TRUNC('15:30:00', HOUR) truncated;

+-----------+
| truncated |
+-----------+
| 15:00:00  |
+-----------+
```
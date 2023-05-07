# Hello, BigQuery

## Hello, World

새로운 프로그래밍 언어를 배울 때 `Hello, World` 를 화면에 출력하는 프로그램을 먼저 만들어 보곤 합니다.&#x20;

BigQuery에서는 SQL(Structured Query Language) 언어를 사용하여 다음과같이 작성할 수 있습니다.

```
SELECT 'Hello, World';
```

| f0\_         |
| ------------ |
| Hello, World |

### 네모의 꿈

SQL 언어는 표 형태의 데이터를 다루는데 최적화 되어 있습니다.  모든 자료를 테이블 형태로 구조화시키고 테이블 혹은 테이블간 연산을 통해 결과 테이블을 만들어 내는 언어로 생각해도 좋습니다.\
\
앞서의 쿼리는 화면상에 하나의 문자열을 보여주고 있습니다.  다르게 표현해 보면,  `Hello, world` 라는 문자열을 담은 네모가 하나 만들어지고 이 네모가 여러분께 보여진다고 말할 수 있습니다.

<figure><img src="https://ojsfile.ohmynews.com/STD_IMG_FILE/2010/0803/IE001222229_STD.jpg" alt=""><figcaption><p>네모의 꿈 - WHITE (1996年)</p></figcaption></figure>

​네모를 추가해 보겠습니다. 콤마를 사용하여 각 네모를 구분해 주면 네모들을 가로로 붙여 나갈 수 있습니다. 이제 3개의 네모를 가로로 붙여 만든 또 다른 네모가 만들어 졌습니다. 네모들을 담고 있는 바깥의 네모를 테이블이라고 부릅니다.

```sql
SELECT 'Alex', '10', 'Boy';
```

| f0\_ | f1\_ | f2\_ |
| ---- | ---- | ---- |
| Alex | 10   | Boy  |

네모에는 이름표가 붙어 있습니다. 앞에서는 멋진 이름을 붙여주지 않았기 때문에 보기에 이상한 이름을 사용하고 있습니다. 각각의 네모에 이름을 지어 주도록 하겠습니다.

```sql
SELECT 'Alex' AS name, '10' AS age, 'Boy' AS gender;
```

| name | age | gender |
| ---- | --- | ------ |
| Alex | 10  | Boy    |

이름표가 붙어 각 네모가 어떤 것을 의미하는지 파악이 쉬워졌습니다. 이제 네모 안의 값을 바꿔보겠습니다. 아까와는 다른 값들을 가지는 네모 테이블이 생겼습니다.

```sql
SELECT 'Benjamin' AS name, '15' AS age, 'Boy' AS gender;
```

| name     | age | gender |
| -------- | --- | ------ |
| Benjamin | 15  | Boy    |

이 번에는 따로 따로 만들어 낸 두 개의 네모 테이블을 세로로 붙여 보도록 하겠습니다.

```sql
SELECT 'Alex' AS name, '10' AS age, 'Boy' AS gender
 UNION ALL
SELECT 'Benjamin' AS name, '15' AS age, 'Boy' AS gender;
```

| name     | age | gender |
| -------- | --- | ------ |
| Alex     | 10  | Boy    |
| Benjamin | 15  | Boy    |

`UNION ALL` 이라는 구문을 사용하지만 당장은 모르셔도 괜찮습니다. 이렇듯 네모들의 세계에서는 네모가 이해할 수 있는 언어를 사용합니다. 앞서 언급한 SQL 이라는 언어입니다. 이 언어는 네모들 세상에서 필요한 재료들을 선별하고 조작하여 우리가 원하는 결과를 만들어내도록 도와 줍니다. 이제 네모의 세상으로 들어가 그들의 꿈을 좇아가 보겠습니다.

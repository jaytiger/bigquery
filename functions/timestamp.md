# 날짜 및 시간 다루기

## 날짜와 시간

날짜 시간 관련 함수를 다룰 때 아래 두 가지 개념의 이해가 도움이 됩니다.

- 절대시간과 상대시간
- Timezone 과 Offset

**절대시간**은 흐르는 시간의 축에서 한 점을 표현하고 이는 세계 어느 곳에서든 동일한 값을 가집니다. 살고 있는 지역과 무관하게
사용되는 시간입니다.  반대로, **상대시간**은 생활시간이라고도 하며 우리가 일상 생활에서 사용하는 시간을 의미합니다. 똑같은 아침 8시라고 하더라도 한국에서의 아침 8시와 런던에서의 아침 8시는 서로 다른 절대시간을 가집니다. 시차가 9시간이 나기 때문입니다.

<figure><img src="../../.gitbook/assets/images/functions/날짜시간함수-절대시간상대시간.png" alt=""><figcaption>절대시간과 상대시간</figcaption></figure>

이렇듯 시간에 대한 표현은 필요에 따라 다르게 사용될 수 있는데 빅쿼리에서는 다음의 5 가지 데이터 형식을 통해서 날짜와 시간을 나타낼 수 있습니다.

- `TIME`
- `DATE`
- `DATETIME`
- `TIMESTAMP`
- `INTERVAL`

`TIME`, `DATE` 그리고 이를 합친 `DATETIME` 은 상대시간을 표현할 때 사용하는 데이터 형식이며, 빅쿼리에서  `TIMESTAMP`는 절대시간을 의미합니다.

마지막으로 `INTERVAL`은 말 그대로 시간과 시간사이의 간격을 뜻합니다.

다음으로는 Timezone과 Offset에 대해서 살펴보겠습니다.

<figure><img src="../../.gitbook/assets/images/functions/날짜시간함수-TimezoneOffset.png" alt=""><figcaption>Timezone vs Offset</figcaption></figure>


간단하게 설명하면 `Timezone`은 역사적으로 시간대의 변경이 동일하게 적용되었던 지역을 의미하며, `Offset`은 협정세계시(UTC)와 시간차이를 나타냅니다.

`Timezone`과 `Offset`은 혼용되어 쓰는 경우가 많으나 엄밀하게 둘은 다른 개념입니다.  예를 들어, 한국과 일본은 서로 다른 Timezone을 가지지만 현재 기준으로는 `+09:00`의 동일한 Offset를 가집니다.

또한, 동일한 `Timezone` 내에서도 일광절약시간제 등의 이유로 시기별로 다른 `Offset` 을 가지기도 합니다.


## 날짜 시간 함수 다루기

시간 정보를 다루는 과정은 다음과 같이 요약할 수 있습니다.


<figure><img src="../../.gitbook/assets/images/functions/날짜시간함수-변환저장활용.png" alt=""><figcaption>Timezone vs Offset</figcaption></figure>

- 획득 및 변환
- 저장
- 활용

시간 정보는 운영계 RDBMS나 서버 로그 등에서 ETL 과정을 거쳐 BigQuery에 적재가 됩니다.

이 과정에서 이전에 문자열이나 다른 형식으로 표현된 시간 정보를 빅쿼리의 시간 날짜 형식으로 변환이 필요합니다. 변환 과정을 거친 후 빅쿼리에 테이블에 저장된 시간 정보는 이후 기간별 KPI 분석이나 시계열 분석 등을 위해 활용이 됩니다.

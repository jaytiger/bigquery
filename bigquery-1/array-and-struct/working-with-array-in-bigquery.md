# Working with Array in BigQuery

BigQuery 에서 제공하는 확장 데이터 형식인 배열과 구조체에 대해서 알아보도록 하겠습니다.  배열과 구조체는 BigQuery에서 가장 중요한 자료 구조로 BigQuery를 제대로 활용하기 위해서는 반드시 숙달해야 하는 내용입니다.

<figure><img src="../../.gitbook/assets/(공유완료-수정금지) EP02 - Array and Struct.png" alt=""><figcaption></figcaption></figure>

배열에서는 두 가지를 기억합니다. &#x20;

* 첫째, 배열 안의 모든 요소(element)들은 동일한 데이터 형식이어야 합니다.
* 둘째, 요소들은 순서를 가지고 있습니다.  \


배열을 표현하는 방법은 각괄호 `[` 와 `]` 사이에 요소들을 순서대로 나열하고 쉼표를 사용하여 구분해 줍니다.

<figure><img src="../../.gitbook/assets/(공유완료-수정금지) EP02 - Array and Struct (1).png" alt=""><figcaption></figcaption></figure>

배열과 관련된 공식문서를 보다 보면 **Flatten** 이라는 단어를 접하게 됩니다.  번역하면 **평탄화**(_평문화_로 해석하기도 함) 의 의미로 가로로 길게 늘어선 배열 요소를 세로로 납작하게 만들어 줄세운다는 심상(心象) 을 가지면 도움이 됩니다.\
\
배열의 요소들을 각괄호 안에 나열된 형태를 평탄화 되지 않은 출력 (un-flattened output) 이라고 하고, 평탄화 과정을 거친 이후의 출력을 평탄화된 출력 (flattened output) 이라고 합니다.   평탄화된 출력은 배열의 각 요소들이 각각의 행에서 보여집니다.  평탄화 과정은 뒤에서 설명드리도록 하겠습니다.

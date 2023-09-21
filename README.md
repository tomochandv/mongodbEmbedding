# mongodb atlas search 를 이용한 자연어 및 검색

[몽고디비 아틀라스에는 elastic search를 이용한 검색엔진을 구성 할 수 있습니다.](https://www.mongodb.com/atlas/search)

그리고 현재 Beta 버전 이긴 하지만 단어 또는 자연어를 통한 유사도 인접 검색이 가능 합니다. 

알고리즘은 [Hierarchical Navigable Small Worlds](https://arxiv.org/abs/1603.09320) 를 사용한다 합니다.

## 대상

저는 백엔드 개발자이며 정확히는 백엔드 응용프로그램 개발자입니다. 그러므로 ML,AI 같은 분야의 알고리즘은 1도 모릅니다.

이걸 공부할려면 다시 수학의 정석부터 봐야 하겠죠.

그러나 우리에겐 시간이 없습니다. 빨리 결과를 만들어야 하죠.

그래서 저와같은 분들을 위해 이 문서를 작성합니다.

물론 mongodb에 대한 기본 지식은 있으셔야 합니다. 

## 시도하게 된 이유

현재 제가 입사전 운영되고 있는 elasticsearch는 온프로미스이며 실제 검색은 db의 like 검색과 동일하게 쓰여 지고 있습니다.
데이터가 많다보니 실제 디비에 like 쿼리를 날리면 너무 느려서 도입한 것이지요.
하지만 es를 유지보수 할  수 있는 인력이 없습니다. 이를 만들고 세팅한 개발자가 퇴사한 상태입니다.
그리고 고객들은 계속 요청합니다. 영어로 food를 검색하면 한글로 음식 이라는 단어가 들어간 데이터도 나오게 해달라고요.

이때 때마침 [2023 mongodb local seoul](https://events.mongodb.com/mongodb-local-seoul) 에 참여 하게 됩니다.

여기서 컨설팅 부스에서 약 20분정도 상담을 하고, 세션을 들어서 이런 기능이 생겼구나 해봐야겠다 싶어 하게 되었습니다.

[참고세션영상](https://vimeo.com/866611055/c519673620?share=copy)

## 개념 및 내용

개념은 아래와 같이 할려합니다.

[벡터검색](https://codong.tistory.com/35) 을 이용할려 합니다.

이 개념은 단어를 백터값으로 계산하여 인접한 값을 구한다 라는 개념이라 합니다. 저도 개념정도만 파악합니다. 이 알고리즘 공식은 외계어네요.

예를 들면 내용만 있는 데이터가 있다고 가정합시다. mongodb에 아래와 같이 데이터를 넣겠습니다.

```
[
  {
     contents: "나의 이름은 토모찬이야. 나는 개발자로 일하고 있어."
  },
  {
     contents: "나의 이름은 토모찬이야. 나는 요리사로 일하고 있어."
  },
  {
     contents: "나의 이름은 토모찬이야. 나는 건축가로 일하고 있어."
  }
]
```
이때 우리는 프로퍼티를 하나 더넣을 것입니다. 백터 array를 넣을 것입니다.

아래와 같이 데이터가 실제 저장이 되어 질 것입니다.

```
[
  {
     contents: "나의 이름은 토모찬이야. 나는 개발자로 일하고 있어.",
     vectorEmbedding: [0.00121221, 0.3424343,.....]
  },
  {
     contents: "나의 이름은 토모찬이야. 나는 요리사로 일하고 있어.",
     vectorEmbedding: [0.00121221, 0.3424343,.....]
  },
  {
     contents: "나의 이름은 토모찬이야. 나는 건축가로 일하고 있어.",
     vectorEmbedding: [0.00121221, 0.3424343,.....]
  }
]
```

제가 한참을 해매던 부분인데 이 백터값 추출은 어떻게 해야 할까요?

우리에겐 그저 빛인 [chatgtp](https://openai.com/blog/chatgpt)가 있습니다.

[chatgtp api](https://platform.openai.com/docs/api-reference/embeddings)를 이용 할 것입니다.

테스트를 할려고 보니 유료 입니다..ㅠㅠ

하지만 걱정 없습니다. 가격이 엄청 나게 저렴합니다. [가격표](https://openai.com/pricing)

실제 테스트를 해보니 약 3만건 정도 호출했는데 $0.07 정도 나왔습니다.

실제 값을 받아오는 코드는 아래와 같습니다.

```
async function getEmbedding(query: string) {
  const url = 'https://api.openai.com/v1/embeddings'
  const openai_key = '여러분의 키를 넣으세요'

  let response = await axios.post(
    url,
    {
      input: query,
      model: 'text-embedding-ada-002',
    },
    {
      headers: {
        Authorization: `Bearer ${openai_key}`,
        'Content-Type': 'application/json',
      },
    }
  )

  if (response.status === 200) {
    return response.data.data[0].embedding
  } else {
    throw new Error(`Failed to get embedding. Status code: ${response.status}`)
  }
}

```

설명 할 필요도 없는 코드입니다. 굳이 하라면 openai에서 토큰을 생성해서 해당 토큰을 넣어 주면 됩니다.

[토큰생성](https://platform.openai.com/account/api-keys)은 여기서 하시면 됩니다.

위의 코드를 실행하면 숫자 배열을 되돌려 줍니다. 이를 데이터에 넣고 검색시에도 검색어를 벡터값을 추출하여 해당 값으로 검색하면 됩니다.

## mongodb atlas search

쉽게 이야기하면 인덱싱을 하는 작업입니다. 이때 우리가 하고자 하는 방향으로 설정 할 수 있습니다.
제가 글로 쓰는것보다 [mongodb document](https://www.mongodb.com/developer/products/atlas/semantic-search-mongodb-atlas-vector-search/)를 보시면 됩니다.
이 문서에서는 데이터 저장시 트리거를 걸어서 데이터 저장 또는 수정시 mongodb atlas 내부에서 openapi 를 호출하여 vector를 구하고 저장 한다 입니다.
저는 일단 트리거는 사용하지 않기로 했습니다. 

```
{
  "mappings": {
    "dynamic": true,
    "fields": {
      "내백터값이저장되는 필드 명": {
        "dimensions": 1536,
        "similarity": "cosine",
        "type": "knnVector"
      }
    }
  }
}
```

해당 문서의 핵심이 요놈입니다.

간단하죠.

그리고 또하나는 일반적인 쿼리로 조회하는게 아닙니다. [파이프 라인](https://www.mongodb.com/docs/manual/core/aggregation-pipeline/)을 이용하여 조회합니다.

예를 들어 볼게요.

```
    const documents = await collection
      .aggregate([
        {
          $search: {
            index: 'default',
            knnBeta: {
              vector: embedding,
              path: 'plotEmbedding',
              k: 5,
              filter: {
                text: {
                  path: 'type',
                  query: 'C',
                },
              },
            },
          },
        },
        {
          $project: {
            _id: 0,
            idx: 1,
            type: 1,
            name: 1,
            brandName: 1,
            score: { $meta: 'searchScore' },
          },
        },
      ])
      .toArray()
```

해당 부분을 해석하면 plotEmbedding 필드에 있는 값을 vector 값으로 조회 하는데, type이라는 필드는 C 라는 겂을 가지고있는 애들만 해줘 입니다.
그리고 되돌려주는 값은 _id, idx, type, brandName, score로 되돌려줘 입니다.
score는 유사도의 점수입니다. _id는 저정사 생가는 obectId 필드이고요. 나머지는 제가 저장한 데이터 필드겠죠.


## 결론

생각보다 간단합니다. 근데 제가 이걸 하는 과정은 3일이나 걸렸습니다. 저 vector로 만드는 부분이 애매 해서 어떠한 알고리즘 도뮬을 써야 하나 엄청 검색하고 테스트를 해본결과
결론은 openai였습니다.

이제 해야 할 일이 또 있습니다. openai를 튜닝해야 하는데 이게 저에게는 큰 숙제네요. 또 한걸음 한걸음 해봐야죠. 

개발 환경과 트렌드는 너무 빠르게 바뀐네요. 개발한지 20년이 이제 눈앞에 보이는데도 항상 공부를 해야 합니다.

그렇지만 참 잼있어요. 이 글이 누군가에게 도움이 되었으면 합니다.

질문은 이슈를 이용해주시면 답변 드리겠습니다. 제가 알고 있다면 말이죠.


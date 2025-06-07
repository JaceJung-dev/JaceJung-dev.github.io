---
layout: post
title:  "Transformer 1 (Embedding, Positional Encoding)"
date:   2025-06-03 16:38:50 +0900
categories:
    - studyarchive
    - mldlai
    
---
# [LLM] Transformer 1 (Embedding, Positional Encoding)

## Background
1. 정보를 나타내는 텐서는 Weight matrix를 통과해도 원래 가지고 있는 정보는 보존.
2. Weighted Sum (Weight들은 양의 소수이며, 모든 weight의 합이 1인 경우)
    - 각각 다른 정보를 가진 텐서에 weighted sum을 하면, weight가 큰 쪽의 정보를 더 가지고 있음.
        - ex) 0.9와 0.1이 weight라고 할 때, 벡터v 는 v2보다 v1의 정보를 더 많이 가지고 있음.

        $$\vec{v} = 0.9 \cdot \vec{v}_1 + 0.1 \cdot \vec{v}_2$$

    - 텐서 3개, 4개, … 많아도 상관없음.
3. inner product / Dot product (내적)
    - 두 벡터의 유사도나 방향성을 측정할 때 사용되는 연산.
    - 같은 위치(차원)끼리 곱해서 더하는 것.
    - 서로 비슷한 벡터끼리 내적을 하면 값이 크고, 서로 유사성이 적으면 내적의 값이 작음.
    - 유사성은 벡터의 방향이 같다는 것.

    $$\vec{a} \cdot \vec{b} = \|\vec{a}\| \, \|\vec{b}\| \cos{\theta}$$

    - 두 벡터가 방향이 같을수록 두 벡터가 이루는 각 θ의 값이 작아지고, θ=0 일 때 cosθ = 1로 최대값을 가진다는 것으로 대강 이해해볼 수 있음.

## 핵심구조
- Encoder-Decoder 구조
- Multi-Head Self-Attention
- Positional Encoding

## Encoder-Decoder
![encoder-decoder](/img//transformer/encoder-decoder.png)
- Encoder: 입력 문장의 의미와 문맥을 이해하여 정리된 정보로 변환.
    - “나는 학생 이다” → 내부적으로 단어의 중요도, 단어간의 관계, 전체 의미를 벡터로 변환.
- Decoder: Encoder가 만든 정보를 바탕으로 적절한 문장을 생성.
    - ‘I am a student” 생성.
![simplified_model](/img/transformer/simplifed_model.png)
- N개의 layer로 구성된 Encoder-Decoder 구조.
- 각 모듈의 구조는 동일하고, 파라미터만 다름.
- 이전 모듈의 output이 다음 모듈의 input으로 들어감.

## Attention

- Attention (Encoder-Decoder Attention)
    - Decoder가 Encoder의 출력(hidden state) 중 어떤 부분에 주목할지 결정하는 매커니즘
    - Decoder의 각 토큰이 Encoder의 모든 토큰과 가중치를 계산
- Self-Attention
    - Encoder 내부 또는 Decoder 내부에서, 각 단어가 자기 자신을 포함한 문장 내 모든 단어와의 관계 (가중치)를 계산하여 문맥을 파악하는 매커니즘
    - 문장 내부에서 토큰 간의 관계를 파악하는 것에 초점

## Transformer의 세부구조
![tf_architecture](/img/transformer/tf_architecture.png)
<br><The Transformer - model architecture.>

### 1. Embedding
- Token 단위로 분리된 자연어를 고차원 벡터로 변환.
- 문장을 조각(Token)으로 나눈 뒤, 각각을 컴퓨터가 이해할 수 있는 숫자 벡터로 변환.
- 자세한 것은 word embedding 을 참고하기.
![embedding](/img/transformer/embedding.png)
- 단어의 토큰ID를 기반으로 Embedding layer를 거쳐, 각 토큰에 대응하는 고차원의 벡터를 생성 (d=512)
```python
# PyTorch에서 사용하는 word embedding
embedding = nn.Embedding(num_embeddings=50000, embedding_dim=512)
output = embedding(token_ids)
# num_embeddings = 모델이 처리할 수 있는 토큰의 개수
# embedding_dim = 하나의 토큰을 나타내는 벡터의 차원 수
```

### 2. Positional Encoding

- 문장에서 단어들의 순서 정보를 반영하기 위한 기법.
- Transformer는 단어들을 순서대로 처리하지 않고, 전체 문장을 병렬적으로 처리하기 때문에, 순서에 대한 정보가 없음.
    - 컴퓨터는 단어(토큰) 백터를 문장 순서에 맞게 순서대로 읽는 것이 아니라, ‘나는’, ‘학생’, ‘이다’ 벡터 3개를 병렬적으로 읽기 때문에 순서를 전혀 알 수 없음.
- 이를 보완하기 위해, 각 단어 벡터에 위치 정보를 인코딩한 값을 함께 더해 모델이 단어의 순서를 인식할 수 있도록 함.
![positional_encoding](/img/transformer/positional_encoding.png)

Transformer에서 Positional Encoding

- Sinusoid Positional Encoding (sin, cos 주기함수를 활용)
    - -1 ~ 1 사이를 반복하는 주기함수
    - 순차성을 부여하되 의미정보가 변질되지 않음. (글자수가 늘어나도 안정적)
    - 서로 다른 두 위치의 벡터 간 차이가 일정한 패턴을 가지기 때문에, 단어 간 상대적 위치를 추론가능
    - 위치 임베딩을 따로 학습하지 않고, 고정된 sin, cos함수를 사용하면서 모델 파라미터를 줄일 수 있음

![linear_sinusoid_PE](/img/transformer/linear_sinusoid_pe.png)

- Equation
![equation_PE](/img/transformer/equation_PE.png)

- Application
![matrix_PE](/img/transformer/matrix_PE.png)
- ex_1) ‘나는’이라는 단어 토큰의 positional embedding vector 만들기 (vector의 차원(d_model)은 편의상 4)
    1. ‘나는’이라는 단어 토큰은 문장 시퀀스의 첫번째 순서이므로 pos = 0 ( ∵ 0번이 첫번째 순서인 것은 익숙합니다.)
    2. ‘나는’이라는 단어 토큰의 positional encoding vector도 embedding vector(의미 벡터)와 똑같은 4개의 차원이므로 i = 0, 1, 2, 3 이 됩니다.
    3. (pos, i) = (0, 0) 이고, 짝수 공식을 사용(0번째 순서라 짝수로 취급)하고, k=0, d_model = 4을 대입하면,

    $$ \text{PE}_{(0,0)} = \sin\left(\frac{0}{10000^{\frac{0}{4}}}\right) = 0 $$

    4\. (pos, i) = (0, 1) 이고, 홀수 공식을 사용하고, k=0, d_model = 4 대입하면

    $$ \text{PE}_{(0,1)} = \cos\left(\frac{0}{10000^{\frac{0}{4}}}\right) = 1 $$

- ex_2) ‘학생’이란 토큰의 2차원 요소의 값을 구해보기
    (pos, i) = (1, 2) 이고, 짝수 공식을 사용하고, k = 1, d_model = 4를 대입하면,
    
    $$ \text{PE}_{(1,2)} = \sin\left(\frac{1}{10000^{\frac{2}{4}}}\right) = \sin\left(\frac{1}{100}\right) \approx 0.00999983 $$

    이런 식으로 계산하여 Positional Encoding Vector들을 생성 (실제 과정은 Matrix에서 이루어짐.)

![positional_encoding_2](/img/transformer/positional_encoding_2.png)
- 이렇게 구한 positional encoding vector 각 차원(위치)에 맞게 더해주는 방식으로 위치정보 추가가 이루어짐.


cf) 한 걸음 더
![visualization_PE](/img/transformer/visualization_PE.png)<br>
[The 128-dimensional positional encoding for a sentence with the maximum length of 50. Each row represents the embedding vector](https://kazemnejad.com/blog/transformer_architecture_positional_encoding/)
- pos = 0 인 경우를 보면, 우리의 경우랑 똑같이 0, 1이 반복되는 것을 볼 수 있고
- pos랑 차원(depth)에 따라 positional encoding vector를 이렇게 시각화할 수 있음.
    - 파랑색에 가까울수록 PE = 1에 가깝고, 빨간색에 가까우면 -1에 가까움.

Positional Encoding의 특징
![dot_product_PE](/img/transformer/dot_product_PE.png)<br>
[Dot product of position embeddings for all time-steps](https://kazemnejad.com/blog/transformer_architecture_positional_encoding/)
- 서로 가까운 위치에 있는 단어들끼리는 유사도가 큼 (내적의 값이 큼)
- 서로 먼 위치에 있는 단어들끼리는 유사도가 작음 (내적의 값이 작다)



## References
[The illustrated Transformer](https://jalammar.github.io/illustrated-transformer/)

[Transformer Architecture: The Positional Encoding](https://kazemnejad.com/blog/transformer_architecture_positional_encoding/)

[[핵심 머신러닝]Transformer](https://www.youtube.com/watch?v=a_-YgMO0u0E)

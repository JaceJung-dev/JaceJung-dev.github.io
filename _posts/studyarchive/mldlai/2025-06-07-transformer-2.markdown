---
layout: post
title:  "Transformer 2 (Encoder)"
date:   2025-06-07 18:58:00 +0900
categories:
    - studyarchive
    - mldlai
    
---
# [LLM] Transformer 2 (Encoder)

## Transformer 1에서...
- Transformer가 Encoder-Decoder 구조로 되어있는 것을 살펴봤음.
- 입력된 자연어의 시퀀스(문장)에서 토큰(단어)를 텐서(숫자)로 변환하는 Embedding 과정이 있다는 것을 봤음.
- 입력된 자연어의 시퀀스(문장)에서 토큰(단서)의 순서 정보를 추가하는 Positional Encoding을 자세히 살펴봤음.


![tf_architecture](/img/transformer2/tf_architecture2.png)

## Encoder: Multi-Head Attention (Self-Attention)
- 입력 문장의 의미와 문맥을 이해한 후, 이를 정리된 정보로 변환하는 과정.
- 자연어로 구성된 입력 시퀀스(문장)는 먼저 Embedding과 Positional Encoding을 거쳐 텐서 형태로 변환되며, 이는 Encoder의 input으로 들어감.
- 이 입력은 Multi-Head Attention(Self-Attention) 매커니즘과 Feed Forward 과정을 거친 후, 최종적으로 행렬(Matrix) 형태의 출력으로 변환됨.

### 1. Core Attention Structures: Multi-Head & Scaled Dot-Product (Encoder의 핵심구조)
![Multi-Head Attention](/img/transformer2/Scaled%20dot-porduct%20attention.png)

1. Multi-Head Attention
- Transformer의 Encoder와 Decoder는 Multi-Head Attention 레이어를 핵심 구성요소로 포함하고 있음.
- Multi-Head Attention은 여러 개의 Scaled Dot-Product Attention을 병렬로 수행함으로써, 정보를 다양한 관점에서 처리할 수 있게 함.
- 즉, 각 Attention Head는 서로 다른 가중치 행렬을 사용하여 입력 간의 관계를 조금씩 다른 방식으로 학습함. 이를 통해 모델은 문맥의 다양한 의미를 포착할 수 있음.

2. Scaled Dot-Product Attention
- Multi-Head Attention 내부에서 실제로 사용되는 핵심 Attention 매커니즘.
- Attention의 목적은 입력 간의 관계(연관성)를 계산하는 것. 특히, 어떤 단어가 문맥 내에서 다른 단어와 얼마나 관련 있는지를 정량적으로 계산함.
- Query(Q), Key(K), Value(V) 라는 세 가지 벡터를 사용해 단어들 간의 관계를 계산.
<br>

### 2. Generating the Query, Key, and Value matrices (Query, Key, Value 행렬 계산하기)
![QKV](/img/transformer2/QKV.png)

- 각 토큰(단어)들의 embedding vector에 Query, Key, Value Weight를 곱해서 각 토큰마다 Q, K, V 벡터를 얻음.
    - Query (Q): 관심 있는 단어가 다른 단어들과 얼마나 관련 있는지 판단하기 위해 사용하는 벡터.
    - Key (K): 문장 내 각 단어가 자기 자신을 설명하는 벡터로, Query와 내적(dot product)을 통해 유사도를 계산함.
    - Value (V): 해당 단어가 가지고 있는 실제 정보가 담긴 벡터로 Attention 결과로 사용됨.
<br>

### 3. Self-Attention Mechanism
1. Equation

$$
\mathrm{Attention}(Q, K, V) = \mathrm{softmax}\left(\frac{QK^\top}{\sqrt{d_k}}\right)V
$$


여기서, $d_k$는 Query와 Key 벡터의 차원 수

<br>
2\. Step-by-Step Attention Calculation
ex) '나는 학생 이다' 문장에서 '나는'이 다른 단어들과 가지는 관계를 계산.
![Encoder_self_attention](/img/transformer2/Encoder_self_attention.png)

- (1) 모든 토큰들의 Embedding vector에 Query, Key, Value weight matrix를 곱해 `q`, `k`, `v`를 만듦.
    - "나는": $x_1$ → $q_1$, $k_1$, $v_1$
    - "학생": $x_2$ → $q_2$, $k_2$, $v_2$
    - "이다": $x_3$ → $q_3$, $k_3$, $v_3$
    
- (2) 관심 있는 단어의 Query($q_1$)와 모든 Key($k_1$, $k_2$, $k_3$)를 내적해서 유사도를 계산함.

    $$
\begin{aligned}
q_1 \cdot k_1^T &= 9 \\
q_1 \cdot k_2^T &= 6 \\
q_1 \cdot k_3^T &= 3
\end{aligned}
$$

- (3) 각 Attention Score에 ${\sqrt{d_k}}$ 로 나누어주면서 스케일 조정을 함.

$$
\begin{aligned}
\frac{9}{\sqrt{2}} &\approx 6.36 \\
\frac{6}{\sqrt{2}} &\approx 4.24 \\
\frac{3}{\sqrt{2}} &\approx 2.12
\end{aligned}
$$

- (4) Scaled Attention Score에 Softmax를 적용해서 확률(Attention Weight)로 변환.
    - softmax(6.36,4.24,2.12)≈(0.8816,0.1057,0.0127)
    - $q_1$은 $k_1$과 가장 관련이 있는 것으로 판단할 수 있음.

- (5) Value vector에 Softmax값 곱하기.

$$
\begin{aligned}
\mathbf{v}'_1 &= 0.8816 \times \mathbf{v}_1 \\
\mathbf{v}'_2 &= 0.1057 \times \mathbf{v}_2 \\
\mathbf{v}'_3 &= 0.0127 \times \mathbf{v}_3
\end{aligned}
$$

- (6) 모든 가중된 Value 벡터를 더해서 $z_1$ 생성.
    - $z_1$ = $v'_1$ + $v'_2$ + $v'_3$
    - $z_1$ 은 "나는"이라는 단어가 문맥에서 어떤 의미를 갖는지를 나타냄. (의미, 순서, 관계 포함)


![Encoder_self_attention_2](/img/transformer2/Encoder_self_attention_2.png)

ex) '나는 학생 이다' 문장에서 '학생'이 다른 단어들과 가지는 관계를 계산
![Encoder_self_attention](/img/transformer2/Encoder_self_attention.png)

- (1) 관심 있는 단어의 Query($q_2$)와 모든 Key($k_1$, $k_2$, $k_3$)를 내적해서 유사도를 계산함.

    $$
\begin{aligned}
q_2 \cdot k_1^T &= 6 \\
q_2 \cdot k_2^T &= 9 \\
q_2 \cdot k_3^T &= 6
\end{aligned}
$$

- (3) 각 Attention Score에 ${\sqrt{d_k}}$ 로 나누어주면서 스케일 조정을 함.

$$
\begin{aligned}
\frac{6}{\sqrt{2}} &\approx 4.24 \\
\frac{9}{\sqrt{2}} &\approx 6.36 \\
\frac{6}{\sqrt{2}} &\approx 4.24
\end{aligned}
$$

- (4) Scaled Attention Score에 Softmax를 적용해서 확률(Attention Weight)로 변환.
    - softmax(4.24,6.36,4.23)≈(0.0967,0.8066,0.0967)
    - $q_2$은 $k_2$과 가장 관련이 있는 것으로 판단할 수 있음.

- (5) Value vector에 Softmax값 곱하기.

$$
\begin{aligned}
\mathbf{v}'_1 &= 0.0967 \times \mathbf{v}_1 \\
\mathbf{v}'_2 &= 0.8066 \times \mathbf{v}_2 \\
\mathbf{v}'_3 &= 0.0967 \times \mathbf{v}_3
\end{aligned}
$$

- (6) 모든 가중된 Value 벡터를 더해서 $z_1$ 생성.
    - $z_2$ = $v'_1$ + $v'_2$ + $v'_3$
    - $z_2$ 은 "학생"이라는 단어가 문맥에서 어떤 의미를 갖는지를 나타냄. (의미, 순서, 관계 포함)

<br>

3\. In Reality: Matrix Computation (실제 계산과정: 행렬계산)

![In_reality_self_attention](/img/transformer2/In_reality_self_attention.png)

- 위에서는 이해를 위해 Vector로 쪼개서 계산을 했지만, 실제로는 Matrix 형태로 한번에 계산을 진행.

<br>

4\. Meaning Of Multi-Head (Multi-Head란?)

![Multi_head](/img/transformer2/Multi-head.png)

- Multi-Head란 여러 개의 Attention Head를 병렬로 사용하는 매커니즘.
- 각 Head는 서로 다른 가중치 행렬(Weight Matrix)을 사용해 입력을 다른 방식으로 변환하고, 다른 관점에서 관계를 학습.
    - 각 Head는 고유의 $W_Q^i$, $W_K^i$, $W_V^i$ 를 사용해 Query, Key, Value를 생성.
    - Scaled Dot-Product Attention을 각 Head에서 독립적으로 수행한 후, 출력들을 `Concat`(연결)하고, 다시 하나의 Linear Layer를 통과시켜 최종 결과를 생성.
- 단일 Head는 제한된 관점만 학습하므로, 복잡한 문맥 정보나 다양한 의미 관계를 포착하는 데 한계가 있음.
- Multi-Head를 사용하면, Head 1은 구문 구조에 집중할 수 있고, Head 2는 문법적 역할, Head 3는 단어 간 의미적 유사성에 집중하는 식.
<br>


5\. Finalization: `Concat` 후 Linear 연산으로 통합

![finalization](/img/transformer2/finalization.png)

- 앞서 설명한 대로, Multi-Head Attention은 여러 개의 Attention Head를 병렬로 구성하여 입력 간의 관계를 다양한 관점에서 학습함.
- 개별 Head들의 출력을 합쳐, 모델의 다음 계층으로 전달할 output을 만들어야 함.
- 다양한 관점에서 학습한 정보를 하나의 의미 있는 표현으로 통합해주는 것이 이 단계의 핵심.
    - (1) 각 Head의 결과를 `Concat`
        - 생성된 출력 $z_{\text{head1}}$, $z_{\text{head2}}$, $z_{\text{head3}}$ 를 가로 방향으로 연결(Concatenate) 하여 하나의 긴 벡터(행렬)로 만듦
        - Equation

            $$ Z_{\text{concat}} = \text{Concat}(z_0, z_1, \dots, z_{h-1}) \in \mathbb{R}^{h \cdot d_k} $$

        - 여기서 $h$는 Head의 개수, $d_k$는 각 Head의 출력 차원입니다.
        
    - (2) 선형 변환 (Linear Transformation)
        - 연결된 벡터 $Z_{\text{concat}}$는 모델에 의해 학습된 출력 가중치 행렬 $W^O$와 곱해져 최종 출력 벡터 $Z$를 생성
        - Equation

        $$ Z = Z_{\text{concat}} \cdot W^O $$

        - 이 연산은 단순히 Head들을 결합하는 것이 아니라, 각 Head에서 얻은 정보를 어떤 비율로 조합할지 학습하는 과정.
        - $W^O$는 학습 가능한 파라미터이며, 모델이 학습 도중 최적의 조합 방법을 스스로 찾아냄.
- 최종 벡터 $Z$는 다음 레이어인 Feed-Forward Neural Network (FFN)으로 전달


## Encoder: Residual Connection & Normalization
1. Residual Connection의 배경

![https://medium.com/@achronus/exploring-residual-connections-in-transformers-2cd18b9e35eb](/img/transformer2/original_residual_connection.png)

- Residual Connection은 “Deep Residual Learning for Image Recognition” (He et al., 2015) 에서 처음 제안됨. 
- CNN의 깊이를 증가시키면 성능이 향상될 것으로 기대했지만, 오히려 정확도가 감소하고 성능 저하가 나타났음.
- 단순한 과적합(Overfitting)이 문제 아니며 조기 종료(Early Stopping) 같은 기법으로 해결되지 않음.
- 정확도 포화(Accuracy Saturation)와 출력값의 극단화로 인해 학습이 어려워짐.
    - 뉴런 출력이 활성화 함수의 극단값(0 or 1)에 수렴하여 정보 표현력이 떨어짐.
- 해결책으로, 이전층의 출력을 그대로 유지하는 스킵 연결(Skip Connection)을 도입함.
    - 개별 레이어를 순차적으로 통과시키는 대신, 여러 레이어를 하나의 블록으로 묶고, 그 주위를 우회하여 입력 정보를 전달함.
    - 이를 통해 깊은 네트워크에서도 안정적인 학습과 성능 개선이 가능해짐.

- 이러한 구조가 반복되면, 수학적으로 다음과 같이 누적되는 형태로 표현할 수 있음.

$$
x_{l+1} = x_l + F(x_l) \\
x_{l+2} = x_{l+1} + F(x_{l+1}) = x_l + F(x_l) + F(x_{l+1}) \\
$$

$$
\therefore \quad x_L = x_l + \sum_{i=l}^{L-1} F(x_i)
$$

2\. Transformer에서 Residual Connection

![https://medium.com/@achronus/exploring-residual-connections-in-transformers-2cd18b9e35eb](/img/transformer2/transformer_residual_connection.png)

- Transformer는 Self-Attention, Feed-Forward Network, LayerNorm 등을 반복적으로 쌓아 구성된 매우 깊은 딥러닝 아키텍처.
- 각 주요 구성 블록(self-attention, FFN 등) 뒤에 Residual Connection + LayerNorm 구조 사용.

3\. Normalization
- Transformer에서는 Residual Connection 뒤에 Layer Normalization을 적용함.
- 모델이 깊어질수록 출력값의 분포가 불안정해질 수 있기 때문에, 이를 정규화하여 출력 값의 크기를 조절하고 학습을 안정화함
- Equation

$$ \text{LayerNorm}(x_i) = \frac{x_i - \mu}{\sigma + \epsilon} \cdot \gamma + \beta $$

μ: 평균, σ: 표준편차, γ,β: 학습 가능한 파라미터, ϵ: 분모 0 방지를 위한 작은 수

- Layer Normalization은 하나의 입력 벡터 내부 값들을 평균과 분산 기준으로 정규화함.
- 배치 크기에 영향을 받지 않기 때문에, Transformer와 같은 시퀀스 모델에 적합함.


## Encoder: Feed Forward
![FFN](/img/transformer2/FFN.png)

- Self-Attention과 Residual Connection & Normalization을 거친 출력은 완전 연결층(Fully Connected Layer) 구조인  Feed Forward Neural Network(FFN)으로 전달됨.
- FFN은 각 토큰의 벡터를 독립적으로 처리하며, 일반적으로 다음과 같은 두 개의 Linear Layer와 비선형 활성화 함수(ReLu 등)로 구성됨.
- 모든 위치(Token)에 대해 동일한 FFN이 적용되므로 Position-wise Feed Forward 라고도 함.
- Attention이 “무엇에 집중할지”를 알려줬다면, FFN은 그 정보를 바탕으로 “어떻게 해석할지”를 학습
- FFN은 각 위치의 토큰 표현에 비선형 변환을 가해 표현력을 확장하고, 개별 토큰의 의미를 정교하게 가공하는 역할을 함.
- Equation

$$ \text{FFN}(x) = \max(0, xW_1 + b_1)W_2 + b_2 $$

$W_1$, $W_2$: 학습 가능한 가중치 행렬, $b_1$, $b_2$: 편향(Bias) 벡터
max(0, -): ReLu 활성화 함수(비선형성 부여)

- (1) 첫 번째 Linear Layer
    -  입력층 → 은닉층
    - 연산: $xW_1 + b_1$

- (2) 두 번째 Linear Layer
    - 은닉층 → 출력층
- 연산: $\text{ReLU}(xW_1 + b_1)W_2 + b_2$

## References
[The illustrated Transformer](https://jalammar.github.io/illustrated-transformer/)

[Transformer Architecture: The Positional Encoding](https://kazemnejad.com/blog/transformer_architecture_positional_encoding/)

[[핵심 머신러닝]Transformer](https://www.youtube.com/watch?v=a_-YgMO0u0E)

[Exploring Residual Connections In Transformers](https://medium.com/@achronus/exploring-residual-connections-in-transformers-2cd18b9e35eb)
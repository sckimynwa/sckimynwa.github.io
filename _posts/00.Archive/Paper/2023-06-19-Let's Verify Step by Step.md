---
share: true
toc: true
tags:
  - ai
  - inference
  - openai
layout: post
title: Let's Verify Step by Step
date: 2023-06-19
github_title: 2023-06-19-Let's Verify Step by Step
background: /assets/img/background.jpg
---


In recent years, large language models have greatly improved in their ability to perform complex multi-step reasoning. However, even state-of-the-art models still regularly produce logical mistakes.

We conduct our own investigation, finding that process supervision significantly outperforms outcome supervision for training models to solve problems from the challenging MATH dataset

outcome supervision and process supervision. Outcome-supervised reward models (ORMs) are trained using only the final result of the model’s chain-of-thought, while process-supervised reward models (PRMs) receive feedback for each step in the chain-of-thought

There are compelling reasons to favor process supervision. It provides more precise feedback, since it specifies the exact location of any errors that occur. It also has several advantages relevant to AI alignment: it is easier for humans to interpret, and it more directly rewards models for following a human-endorsed chain-of- thought.

We have shown that process supervision can be used to train much more reliable reward models than outcome supervision in the domain of mathematical reasoning. 

We have also shown that active learning can be used to lower the cost of human data collection by surfacing only the most valuable model completions for human feedback.

We release PRM800K, the full dataset of human feedback used to train our state-of-the-art reward model, with the hope that removing this significant barrier to entry will catalyze related research on the alignment of large language models. 

We believe that process supervision is currently under- explored, and we are excited for future work to more deeply investigate the extent to which these methods generalize.



## Thoughts

ChatGPT가 등장하고 나서 수많은 언론들에서는 마치 AI가 수많은 일자리를 대체하고, 지금껏 보지 못했던 대 혁신을 가져올 것처럼 이야기했지만, GPT-4가 등장한지 시간이 꽤 지난 현 시점에서 세상을 바꿀 정도의 임팩트나 기대감을 주는 킬러 앱이 등장한 느낌은 아니다.(하지만 앞으로는 충분히 등장할 여지가 많다고 생각한다) 개인적으로는 크게 2가지 이유가 있다고 생각한다.

### Lack of Hardware
전반적으로 GPU, ASIC과 같은 하드웨어가 시장에 매우매우 부족한 상황이다. GPT와 같은 Large Language Model을 훈련하기 위해서는 단순 GPU가 아니라 Matrix Multiplication에 최적화된 TPU, NPU와 같은 특수 설계된 하드웨어 칩과 전용 소프트웨어가 필요한데, NVIDIA 이외에는 별다른 대항마가 없는 상황이고, 생산량에 한계가 있어 공급에 비해 수요가 매우매우 높은 상황이다. (구글이나 테슬라는 자체 생산하는 것 같긴 하지만 기업을 위해 대규모로 이를 공급할 정도로 열려있는 느낌은 아닌 것 같다.)

On-Demand AI Service를 하기 위해서는 실시간으로 대용량의 Inference를 처리할만한 하드웨어 공급이 필요하며, 이 문제가 해결되지 않으면 Scalable한 애플리케이션이 등장하기 힘들다. (실제로 OpenAI도 Chat GPT 서비스로 인해 막대한 운영비용을 지출하고 있다.)

하지만 이에 대해 크게 걱정하지는 않는 시각인데, 최근 AMD 측에서 HuggingFace와 손을 잡으면서 AI 훈련 및 추론을 협업을 통해 풀어나가려는 시도를 하고 있다는 점 하나와, 이미 수많은 기업들이 GPU에 대한 수요를 확인하고 앞으로의 시장 크기에 대해 짐작하고 있는 상황에서 어느 정도의 시간이 지나면 자유 시장 경제에 의해 해결되지 않을까 하는 막연한 기대이다.

### Lack of Reasoning
하드웨어의 공급 부족과는 별개로 정말 해결하기 어렵고 중요한 문제는 현재 GPT4와 같은 매우 성능이 뛰어난 Baseline 모델에서 보여주는 추론 능력의 한계와 Hallucination, 즉 낮지 않은 논리적 오류의 가능성이라 생각한다. 잘 훈련된 모델이 어떤 논리적인 Step(Chain of Thoughts)을 거쳐 어떤 결론을 낸다고 했을 때, 해당 결론의 논리적 타당성은(여기서의 논리적 타당성이란 false positive와 같이 답만 맞는 경우를 모두 제외한 결과와 추론 과정이 모두 타당하다는 것을 의미한다.) 필연적으로 Chain of Thoughts 의 가장 약한고리 보다도 약하다.

이를 해결하기 위해 전체 결과로부터 모든 step을 back propagation 하는 과정이 아닌 각 Step마다 해당 step의 타당성을 평가하고 Reward Model을 학습시키는 과정이 필요하다고 생각했었는데, 이를 최근(23.05.31) Open AI에서 PRM(Process-supervised Reward Model)이라는 이름으로 풀어내려 시도했고, 유의미한 결과를 얻은 것 같아서 조금 더 희망을 걸게 되었다.

### Interesting Parts
놀라웠던 점 중 하나는, MATH Dataset으로 학습한 모델이 Physics, Chemistry와 같은 "추론"이 필요한 다른 영역에서도 높은 성능을 보였다는 점이다. 논문에서도 언급되었듯, PRM은 블랙 박스라고만 여겨졌던 AI Model의 추론 과정에 Human-visible 한 영역들을 제공하고, 답이 틀렸을 때, 어떤 Step에서부터 잘못되었는지를 역산할 수 있는 가능성을 제공한다. 

>There are compelling reasons to favor process supervision. It provides more precise feedback, since it specifies the exact location of any errors that occur. It also has several advantages relevant to AI alignment: it is easier for humans to interpret, and it more directly rewards models for following a human-endorsed chain-of- thought.

수능 수학 문제를 풀 때, 정답을 맞히기 위해 문제를 풀었던 과정을 복기해보면
1. 문제를 읽고, 가능한 풀이 방식을 여러 개 떠올려 본다
2. 그중 가장 "안전한" 방식을 선택하고 첫 수식을 써내려간다
3. 해당 수식이 맞는지를 가볍게 검증하고 다음 수식을 써내려간다
와 같은 방식을 무의식적으로 수행했던 것 같다. 여기서 경험적으로 "안전한" 방식이라는 것은 그동안 수많은 연습문제와 모의고사를 풀면서 정답률이 가장 높고 빠르게 풀수 있는 방식을 의미하며, 이는 훈련 과정을 통해 학습된 내 뇌 안에 있는 "보상 모델"이 결정했다고 비유할 수 있을 것이다.


## Reference
- [Let's Verify Step by Step](https://arxiv.org/abs/2305.20050)
- [A dive into the AMD driver workflow](https://geohot.github.io/blog/jekyll/update/2023/06/07/a-dive-into-amds-drivers.html)
- [Hugging Face & AMD](https://huggingface.co/blog/huggingface-and-amd)

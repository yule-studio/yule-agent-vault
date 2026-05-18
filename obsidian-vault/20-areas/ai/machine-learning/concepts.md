---
title: "ML 핵심 개념 — supervised / unsupervised / loss / gradient / 평가"
kind: knowledge
project: ai
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-19T01:45:00+09:00
tags: [ai, machine-learning, concepts]
home_hub: machine-learning
related:
  - "[[machine-learning]]"
  - "[[loss-and-gradient-descent]]"
  - "[[training-and-evaluation]]"
  - "[[../foundations/foundations]]"
  - "[[../deep-learning/concepts]]"
---

# ML 핵심 개념 — supervised / unsupervised / loss / gradient / 평가

**[[machine-learning|↑ machine-learning]]**

---

## 1. 목적

본 문서는 머신러닝 (ML) 의 기본 용어 / 학습 종류 / 데이터 흐름 / 평가 / 일반화의 정의를 정리한다.

본 문서가 정의하는 것:
- ML 의 5 학습 종류 (supervised / unsupervised / self-supervised / semi-supervised / reinforcement)
- 가설 / 모델 / 파라미터 / hyperparameter 의 구분
- loss / gradient / optimizer 의 관계
- train / val / test 데이터 분리의 의미
- bias-variance / overfitting / 일반화
- 평가 metric 의 종류와 선택 기준

본 문서가 정의하지 않는 것:
- 손실 함수 / gradient descent 의 수학 — [[loss-and-gradient-descent]]
- train / val / test 분리의 절차 / cross-validation — [[training-and-evaluation]]
- 신경망 / backprop / autograd — [[../deep-learning/concepts]], [[../deep-learning/backpropagation]]
- Python / NumPy / Pandas 사용법 — [[../foundations/data-analysis]]

---

## 2. 범위

| 구분 | 포함 |
| --- | --- |
| 대상 | 전통 ML (선형 회귀 / logistic / tree / SVM / k-NN / k-means / gradient boosting) + 일반 학습 이론 |
| 제외 | 신경망 / Transformer / LLM — `../deep-learning/`, `../llm/` |
| 도구 | scikit-learn / NumPy / Pandas |

---

## 3. 용어

| 용어 | 정의 |
| --- | --- |
| **dataset** | 입력 `X` 와 (선택적) 정답 `y` 의 집합. 보통 N 개 sample. |
| **sample (example, row)** | 한 입력 단위. `X[i]`. |
| **feature (column, attribute)** | sample 의 한 차원. `X[i, j]`. |
| **label (target, y)** | 정답. 회귀에선 실수, 분류에선 클래스. |
| **model** | 입력 → 출력의 함수 `f(X; θ)`. θ 는 학습으로 결정되는 파라미터. |
| **parameter (θ)** | 모델이 학습하는 값 (선형 회귀의 가중치 W / 편향 b). |
| **hyperparameter** | 사람이 정하는 값 (learning rate / 트리 깊이 / regularization 강도). |
| **loss function** | 모델 출력과 정답의 차이를 한 숫자로. 학습은 이 값을 최소화. |
| **gradient** | loss 의 파라미터에 대한 편미분 vector. 어느 방향으로 움직이면 loss 가 줄어드는지. |
| **optimizer** | gradient 를 받아 파라미터를 갱신하는 알고리즘 (SGD / Adam / ...). |
| **epoch** | 전체 학습 데이터를 한 번 통과한 횟수. |
| **batch** | 한 번 gradient 를 계산하는 sample 단위. |
| **inference (prediction)** | 학습된 모델로 새 입력에 대한 출력 계산. |
| **generalization** | 학습 데이터 외 새 데이터에서의 성능. |
| **overfitting** | 학습 데이터에는 잘 맞지만 새 데이터엔 못 맞춤. |
| **underfitting** | 학습 데이터에도 못 맞춤. 모델 / feature 가 부족. |

---

## 4. ML 의 5 학습 종류

### 4.1 supervised learning (지도 학습)

input `X` 와 정답 `y` 가 같이 주어진다. 모델은 `f(X) ≈ y` 가 되도록 학습.

| 하위 분류 | 정의 | 예 |
| --- | --- | --- |
| **regression** | y 가 연속값 | 집값 / 주가 / 온도 예측 |
| **classification** | y 가 이산 클래스 | spam / 이미지 라벨 / 신용 등급 |

대표 모델: 선형 / logistic regression, decision tree, random forest, gradient boosting (XGBoost / LightGBM), SVM, k-NN, 신경망.

### 4.2 unsupervised learning (비지도 학습)

input `X` 만 주어지고 정답 없음. 데이터의 구조 / 분포를 추론.

| 하위 분류 | 정의 | 예 |
| --- | --- | --- |
| **clustering** | sample 들을 그룹화 | 고객 segment / 문서 분류 |
| **dimensionality reduction** | 고차원 → 저차원 보존 | 시각화 (PCA / t-SNE / UMAP) |
| **density estimation** | 데이터 분포 추정 | 이상치 탐지 |
| **association rule** | 동시 발생 패턴 | "기저귀 사면 맥주" |

대표 모델: k-means, hierarchical clustering, DBSCAN, PCA, t-SNE, UMAP, Gaussian Mixture, Autoencoder.

### 4.3 self-supervised learning

label 을 데이터 자체에서 만들어 supervised 처럼 학습. 현재 LLM / 비전 의 표준.

| 예 | 자기 label |
| --- | --- |
| 다음 단어 예측 (LLM) | 다음 token |
| masked language modeling (BERT) | 가린 token |
| contrastive learning (SimCLR) | 이미지 augment 페어 |

→ supervised 의 일종이지만 label 이 자동 생성되어 매우 많은 데이터 사용 가능.

### 4.4 semi-supervised learning

소량 labeled + 대량 unlabeled. unlabeled 데이터의 구조도 활용.

### 4.5 reinforcement learning (RL)

agent 가 환경에서 행동하고 reward 를 받음. 누적 reward 최대화 정책 학습.

| 구성 | 정의 |
| --- | --- |
| state (s) | 환경의 현재 상황 |
| action (a) | agent 의 선택 |
| reward (r) | 결과의 점수 |
| policy (π) | s → a 의 함수 |

예: 게임 (DQN / AlphaGo), 로봇 제어, RLHF (LLM fine-tuning).

---

## 5. ML 의 데이터 흐름

```
[raw data]
    ↓ 전처리 (cleaning / encoding / scaling)
[X, y]
    ↓ split
[train (60-80%) / val (10-20%) / test (10-20%)]
    ↓ 학습 (train + val)
[model with θ]
    ↓ 평가 (test)
[metric value]
    ↓ deploy
[inference on new X]
```

각 단계 상세는 [[training-and-evaluation]].

---

## 6. 모델 / 가설 공간

### 6.1 가설 공간 (hypothesis space)

모델 종류와 파라미터의 가능한 조합. 예: "선형 모델 `y = w·x + b`" 은 (w, b) 의 모든 조합.

| 가설 공간 크기 | 효과 |
| --- | --- |
| 너무 작음 | 데이터 패턴 표현 못함 → underfitting |
| 너무 큼 | 데이터의 noise 까지 학습 → overfitting |
| 적절 | 일반화 잘됨 |

### 6.2 inductive bias

모델이 가진 "이런 데이터일 거다" 라는 가정.

| 모델 | inductive bias |
| --- | --- |
| 선형 회귀 | 출력은 입력의 선형 조합 |
| decision tree | 데이터는 축에 직교한 분할로 구분 가능 |
| CNN | 공간적 locality + translation invariance |
| RNN | 순차 의존성 |
| Transformer | 모든 위치가 모든 위치와 직접 연결 가능 |

### 6.3 No Free Lunch 정리

모든 task 에서 최고인 모델은 없다. task / 데이터 / 제약에 맞는 모델 선택이 필요.

---

## 7. loss / gradient / optimizer

### 7.1 loss function

모델 출력 `ŷ` 와 정답 `y` 의 차이를 1 숫자로.

| task | 대표 loss |
| --- | --- |
| 회귀 | MSE (mean squared error), MAE (mean absolute error), Huber |
| 이진 분류 | binary cross-entropy (BCE), hinge |
| 다중 분류 | categorical cross-entropy, focal loss |
| 순위 학습 | pairwise / listwise (NDCG-based) |
| LLM | next-token cross-entropy |

상세: [[loss-and-gradient-descent]].

### 7.2 gradient descent

loss 의 파라미터에 대한 gradient (∂L/∂θ) 의 반대 방향으로 한 step 이동.

```
θ_{t+1} = θ_t - η · ∇L(θ_t)
```

| 기호 | 의미 |
| --- | --- |
| η (learning rate) | step 크기 (hyperparameter) |
| ∇L | gradient |

### 7.3 optimizer

gradient 를 받아 어떻게 파라미터를 갱신할지 결정.

| optimizer | 특징 |
| --- | --- |
| SGD | 가장 단순. 잘 동작하면 안정. |
| SGD + momentum | 누적 gradient 의 관성 |
| AdaGrad / RMSProp | 차원별 learning rate 적응 |
| Adam | momentum + 적응 — 현재 사실상 default |
| AdamW | Adam + 정확한 weight decay (LLM 표준) |
| LAMB / LARS | 매우 큰 batch 용 |

---

## 8. overfitting / underfitting / bias-variance

### 8.1 overfitting

모델이 학습 데이터의 noise 까지 외운다.

| 신호 | 측정 |
| --- | --- |
| train loss 매우 낮음, val loss 높음 | 학습 곡선 시각화 |
| 새 데이터에서 성능 급락 | test set 점수 |

### 8.2 underfitting

모델이 학습 데이터의 패턴도 못 잡는다.

| 신호 | 측정 |
| --- | --- |
| train loss 도 높음 | 학습 곡선 |
| 더 복잡한 모델 / feature 로 개선 | A/B 비교 |

### 8.3 bias-variance trade-off

| 항목 | 정의 |
| --- | --- |
| bias | 모델의 평균 예측이 정답과 얼마나 다른가 — 모델 가정의 강도 |
| variance | 모델이 데이터 sample 마다 얼마나 다르게 예측하는가 — 데이터 의존성 |

| 모델 | bias | variance |
| --- | --- | --- |
| 단순 (선형) | 높음 | 낮음 |
| 복잡 (deep tree / NN) | 낮음 | 높음 |

→ 적당한 bias-variance balance 가 일반화 최적. regularization / 더 많은 데이터 / 앙상블이 분산을 줄이는 도구.

### 8.4 regularization

overfitting 방지 기법.

| 기법 | 효과 |
| --- | --- |
| L2 (Ridge) | 가중치 크기 페널티 — 부드러운 모델 |
| L1 (Lasso) | 가중치 sparsity — 일부 feature 사용 안 함 |
| dropout (NN) | 학습 중 일부 뉴런 무작위 끄기 |
| early stopping | val loss 가 더 안 줄어들면 학습 중단 |
| data augmentation | 학습 데이터 다양성 증가 |
| label smoothing | 너무 자신감 있는 예측 억제 |
| ensemble | 여러 모델 결합으로 variance 감소 |

---

## 9. 평가 metric

### 9.1 회귀

| metric | 정의 | 특성 |
| --- | --- | --- |
| MSE | (ŷ-y)² 평균 | 큰 오차에 민감 |
| RMSE | √MSE | 원 단위 |
| MAE | |ŷ-y| 평균 | outlier 에 덜 민감 |
| R² | 분산 설명력 | 0-1 (1=완벽) |
| MAPE | |ŷ-y|/|y| 평균 | 상대 오차 |

### 9.2 이진 분류

| metric | 정의 | 사용 |
| --- | --- | --- |
| accuracy | (TP+TN) / N | 클래스 균형일 때 |
| precision | TP / (TP+FP) | 오탐 비용 큼 (스팸 / 진단) |
| recall | TP / (TP+FN) | 누락 비용 큼 (질병 / 사기) |
| F1 | 2·P·R / (P+R) | precision/recall 균형 |
| AUC-ROC | TPR vs FPR 곡선 면적 | threshold 무관 평가 |
| AUC-PR | precision vs recall 곡선 | 클래스 불균형 시 |
| log loss | -log P(true class) | calibration |

### 9.3 다중 분류

| metric | 정의 |
| --- | --- |
| top-k accuracy | top-k 안에 정답이 있는가 |
| macro F1 | 클래스별 F1 의 평균 (불균형에 강) |
| micro F1 | 전체 TP/FP/FN 으로 계산 |
| weighted F1 | 클래스 빈도 가중 평균 |
| confusion matrix | 클래스 × 클래스 |

### 9.4 클러스터링

| metric | 정의 |
| --- | --- |
| silhouette | 군집 내 vs 군집 간 거리 |
| Davies-Bouldin | 군집 분리도 |
| ARI / NMI | 정답 라벨이 있을 때 |

### 9.5 metric 선택 기준

| 상황 | 권장 metric |
| --- | --- |
| 클래스 균형 + 두 클래스의 비용 같음 | accuracy |
| 클래스 불균형 (1:100) | F1 / AUC-PR / recall@k |
| 오탐 비용 > 누락 비용 | precision |
| 누락 비용 > 오탐 비용 | recall |
| threshold 무관 비교 | AUC-ROC |
| 회귀 + outlier 무시 | MAE |
| 회귀 + 큰 오차에 민감 | RMSE / MSE |

---

## 10. data leakage — 평가의 함정

학습 시 평가 데이터의 정보가 새어 들어가면 평가가 부풀려진다.

| leakage 유형 | 예 |
| --- | --- |
| temporal | 미래 데이터로 과거 예측 학습 |
| feature | 정답을 거의 그대로 담은 feature |
| preprocessing | test 데이터의 통계로 train scaling |
| duplicate | train / test 에 같은 sample |
| group | 같은 사용자의 데이터가 train + test 양쪽 |

→ 평가는 항상 학습에 사용하지 않은 데이터로. 상세: [[training-and-evaluation]].

---

## 11. ML 모델 선택 결정

| 조건 | 권장 |
| --- | --- |
| 작은 데이터 (< 10k) + 표 형식 | 선형 / logistic / random forest |
| 표 형식 + 정확도 우선 | gradient boosting (XGBoost / LightGBM / CatBoost) |
| 표 형식 + 해석 가능성 | decision tree / logistic + SHAP |
| 이미지 | CNN (ResNet / EfficientNet) 또는 transfer learning |
| 자연어 | Transformer (BERT / GPT) — 전통 ML 보다 우월 |
| 시계열 | 전통 (ARIMA / Prophet) 또는 RNN / Transformer |
| 군집화 | k-means → DBSCAN → HDBSCAN |
| 차원 축소 시각화 | UMAP / t-SNE |
| 추천 | matrix factorization / 신경망 |
| 즉시 가능 + interpretable | 선형 / logistic |

---

## 12. 흔한 실패 모드

| 실패 | 원인 | 정정 |
| --- | --- | --- |
| train / val gap 큼 | overfitting | regularization / 데이터 추가 / 단순화 |
| train loss 도 안 줄어듦 | underfitting / lr 잘못 | 모델 복잡도 ↑ / lr 조정 |
| val accuracy 가 너무 좋음 (의심) | data leakage | feature / preprocessing / sample 분리 검증 |
| accuracy 는 95% 인데 실제 쓸모 없음 | 클래스 불균형 — 다수 클래스만 예측 | F1 / AUC-PR / recall 으로 평가 |
| 학습이 진동 / NaN | lr 너무 큼 / 데이터 정규화 안 됨 | lr 감소 / scaling / gradient clipping |
| 학습이 멈춤 (loss 그대로) | lr 너무 작음 / dead ReLU | lr 증가 / activation 변경 |
| test 결과가 실험마다 다름 | random seed 미고정 | seed 고정 + 여러 seed 평균 |
| production 에서 성능 급락 | distribution shift | 모니터링 / retraining pipeline |

---

## 13. 참고

- [[machine-learning|↑ machine-learning hub]]
- [[loss-and-gradient-descent]]
- [[training-and-evaluation]]
- [[../foundations/foundations]]
- [[../foundations/data-analysis]]
- [[../foundations/data-visualization]]
- [[../deep-learning/concepts]]
- [[../ai-senior-curriculum]]
- Andrew Ng — Coursera ML Specialization
- "Hands-On Machine Learning with Scikit-Learn, Keras, and TensorFlow" (Géron, 3rd ed)
- "Pattern Recognition and Machine Learning" (Bishop)
- scikit-learn docs — https://scikit-learn.org

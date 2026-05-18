---
title: "training-and-evaluation — split / cross-validation / metric / leakage 방지"
kind: knowledge
project: ai
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-19T02:00:00+09:00
tags: [ai, machine-learning, training, evaluation, cross-validation, leakage]
home_hub: machine-learning
related:
  - "[[machine-learning]]"
  - "[[concepts]]"
  - "[[loss-and-gradient-descent]]"
  - "[[../foundations/data-analysis]]"
---

# training-and-evaluation — split / cross-validation / metric / leakage 방지

**[[machine-learning|↑ machine-learning]]**

---

## 1. 목적

본 문서는 ML 모델의 학습 / 평가의 표준 절차 — 데이터 분리 / cross-validation / metric 계산 / 학습 곡선 분석 / leakage 방지 — 를 정의한다.

본 문서가 정의하는 것:
- train / val / test 분리의 의미와 비율
- hold-out / k-fold / stratified / time-series 의 cross-validation 변형
- 학습 곡선 (loss / accuracy / validation curve / learning curve) 의 의미
- hyperparameter tuning 절차 (grid / random / Bayesian)
- data leakage 의 6 형태와 방지
- 평가 결과 보고의 표준

본 문서가 정의하지 않는 것:
- 손실 함수 / gradient descent — [[loss-and-gradient-descent]]
- metric 의 정의 — [[concepts]] §9
- 신경망 학습 loop — [[../deep-learning/training-loop-pytorch]]

---

## 2. 범위

| 구분 | 포함 |
| --- | --- |
| 대상 | 모든 supervised ML / DL 모델 |
| 도구 | scikit-learn (`train_test_split`, `cross_val_score`, `GridSearchCV`) / PyTorch |
| 제외 | RL 의 평가 (별도 절차) |

---

## 3. 용어

| 용어 | 정의 |
| --- | --- |
| **train set** | 모델이 학습하는 데이터. |
| **validation set (dev)** | hyperparameter 선택 / early stopping 용. |
| **test set (hold-out)** | 최종 평가 1 회만 사용. 절대 학습 / 튜닝에 사용 X. |
| **cross-validation (CV)** | 같은 데이터를 여러 fold 로 나눠 여러 번 학습/평가. |
| **fold** | CV 의 1 분할. |
| **hyperparameter** | 사람이 정하는 값 (lr / depth / 정규화 강도). |
| **stratified** | 클래스 비율을 모든 fold 에서 동일하게 유지. |
| **leakage** | test 정보가 학습에 새어 들어감 → 평가 부풀려짐. |
| **early stopping** | val loss 가 더 안 줄면 학습 중단. |

---

## 4. train / val / test 분리

### 4.1 표준 비율

| 데이터 크기 | 권장 |
| --- | --- |
| < 10k | 60 / 20 / 20 또는 70 / 15 / 15 |
| 10k - 1M | 80 / 10 / 10 |
| > 1M (특히 DL) | 98 / 1 / 1 — val/test 1% 도 충분히 큼 |

### 4.2 의미

| set | 책임 |
| --- | --- |
| **train** | 모델 파라미터 (θ) 학습 |
| **val (dev)** | hyperparameter 선택 + early stopping + 모델 선택 |
| **test** | 최종 일반화 성능 1 회 측정 |

→ test 를 모델 / hyperparameter 선택에 사용하면 사실상 train 의 일부가 됨 (leakage). 보고된 metric 부풀려짐.

### 4.3 코드 (scikit-learn)

```python
from sklearn.model_selection import train_test_split

# 1차 분할: train+val vs test
X_trainval, X_test, y_trainval, y_test = train_test_split(
    X, y,
    test_size=0.2,
    random_state=42,
    stratify=y       # 분류일 때 클래스 비율 보존
)

# 2차 분할: train vs val
X_train, X_val, y_train, y_val = train_test_split(
    X_trainval, y_trainval,
    test_size=0.25,  # 0.25 * 0.8 = 0.2 → 전체의 20%
    random_state=42,
    stratify=y_trainval
)
```

### 4.4 random_state / seed

| 효과 | 결과 |
| --- | --- |
| seed 고정 | 같은 split / 같은 학습 결과 재현 |
| seed 미고정 | 실행마다 다른 결과 — 디버깅 / 비교 불가 |

→ random_state / torch.manual_seed / numpy.random.seed 모두 고정.

---

## 5. Cross-Validation 종류

### 5.1 hold-out (단일 split)

가장 단순. train / val 1 회.

| 장점 | 빠름 (1 회 학습) |
| --- | --- |
| 단점 | 평가 분산 큼 (val 하나의 sample) |
| 사용 | 데이터 크면 (val 자체가 충분히 큼) |

### 5.2 k-fold cross-validation

데이터를 k 개 fold 로 나눠 1 fold 는 val, 나머지 k-1 fold 는 train. k 번 반복.

```python
from sklearn.model_selection import KFold, cross_val_score

cv = KFold(n_splits=5, shuffle=True, random_state=42)
scores = cross_val_score(model, X_trainval, y_trainval,
                          cv=cv, scoring='accuracy')
print(f"CV mean = {scores.mean():.4f} ± {scores.std():.4f}")
```

| 장점 | 평가 안정 / 데이터 효율 |
| --- | --- |
| 단점 | k 배 학습 시간 |
| 권장 k | 5 또는 10 |

### 5.3 stratified k-fold

각 fold 에서 클래스 비율을 동일하게 유지. **불균형 분류** 의 표준.

```python
from sklearn.model_selection import StratifiedKFold

cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=42)
```

### 5.4 group k-fold

같은 group (사용자 id / 환자 id) 의 sample 이 train + val 양쪽에 들어가지 않도록.

```python
from sklearn.model_selection import GroupKFold

cv = GroupKFold(n_splits=5)
for train_idx, val_idx in cv.split(X, y, groups=user_ids):
    ...
```

→ 의료 / 추천 / 사용자별 데이터에서 필수. 같은 사용자가 양쪽에 있으면 leakage.

### 5.5 time-series split

시계열 데이터는 **시간 순서를 보존**. 미래 데이터로 과거 예측 학습 = leakage.

```python
from sklearn.model_selection import TimeSeriesSplit

tscv = TimeSeriesSplit(n_splits=5)
for train_idx, val_idx in tscv.split(X):    # train_idx 가 항상 val_idx 보다 이전
    ...
```

| 구조 | 예 |
| --- | --- |
| fold 1 | train: [0, 100], val: [100, 200] |
| fold 2 | train: [0, 200], val: [200, 300] |
| fold 3 | train: [0, 300], val: [300, 400] |

### 5.6 nested CV

hyperparameter tuning 까지 평가에 포함하려면 outer CV (평가) + inner CV (튜닝) 의 2중 구조.

```
outer fold 1:
  train = 80%, test = 20%
  inner CV (on outer train):
    best hyperparameter 선택
  → outer train 전체에 best hyperparameter 로 재학습
  → outer test 로 평가
... 반복
```

| 장점 | hyperparameter 도 평가에 반영 |
| --- | --- |
| 단점 | k² 배 학습 시간 |
| 사용 | 학회 / 논문 / 매우 엄격한 평가 |

---

## 6. 학습 곡선 (learning curves)

### 6.1 종류

| 곡선 | x 축 | y 축 | 의미 |
| --- | --- | --- | --- |
| **loss curve** | epoch / step | train / val loss | 학습 진행 / overfit 진단 |
| **accuracy curve** | epoch | train / val accuracy | 분류 성능 추적 |
| **validation curve** | hyperparameter | val score | hyperparameter 영향 |
| **learning curve (Sklearn)** | train 데이터 크기 | train / val score | 데이터 충분한지 진단 |

### 6.2 진단

```
        loss
         │      
         │       ╲       
   train │        ╲___________  ← train 잘 줄어듦
         │
   val   │        ___           ← val 줄다가 다시 증가
         │       /   ⌐
         └─────────────────── epoch
                  ↑
                early stopping 시점
```

| 패턴 | 진단 |
| --- | --- |
| train ↓ val ↓ 평행 | 정상 |
| train ↓ val ↑ (gap 큼) | overfitting → regularization / 데이터 ↑ |
| train, val 모두 높음 | underfitting → 모델 복잡도 ↑ / feature ↑ |
| train, val 모두 평탄 | 학습 멈춤 → lr 조정 / optimizer 변경 |
| 진동 | lr 너무 큼 → lr 감소 / gradient clipping |
| spike (한 epoch 만 솟음) | bad batch / numerical issue |

### 6.3 learning curve — 데이터 충분한지

`learning_curve` 는 train size 를 변화시키며 그린다.

```python
from sklearn.model_selection import learning_curve

sizes, train_scores, val_scores = learning_curve(
    model, X, y, cv=5,
    train_sizes=np.linspace(0.1, 1.0, 10),
    scoring='accuracy'
)
```

| 패턴 | 진단 |
| --- | --- |
| train ↓ / val ↑ → 수렴 | 데이터 추가가 도움 (high variance) |
| train ≈ val 이지만 낮음 | 모델 복잡도 부족 (high bias) — 데이터 추가는 거의 도움 안 됨 |

---

## 7. hyperparameter tuning

### 7.1 절차

| 도구 | 동작 |
| --- | --- |
| **GridSearchCV** | 모든 조합 시도 — 작은 공간 |
| **RandomizedSearchCV** | 무작위 N 회 — 큰 공간에 효율 |
| **Bayesian (optuna / hyperopt)** | 이전 결과 기반 다음 시도 — 가장 효율 |
| **Successive halving (HalvingGridSearchCV)** | 빠른 후보 → 좋은 것만 더 학습 |

### 7.2 GridSearchCV 예

```python
from sklearn.model_selection import GridSearchCV

param_grid = {
    'C': [0.1, 1, 10, 100],
    'kernel': ['rbf', 'linear'],
    'gamma': ['scale', 'auto', 0.1, 0.01],
}

grid = GridSearchCV(
    SVC(),
    param_grid,
    cv=5,
    scoring='f1_macro',
    n_jobs=-1,
)
grid.fit(X_trainval, y_trainval)

print(grid.best_params_)
print(grid.best_score_)
y_test_pred = grid.best_estimator_.predict(X_test)
```

→ best_estimator_ 로 test 평가. test 점수를 보고 다시 grid 를 변경하면 leakage.

### 7.3 Optuna 예 (Bayesian)

```python
import optuna

def objective(trial):
    C = trial.suggest_loguniform('C', 0.01, 100)
    gamma = trial.suggest_loguniform('gamma', 1e-4, 1)
    model = SVC(C=C, gamma=gamma, kernel='rbf')
    scores = cross_val_score(model, X_trainval, y_trainval, cv=5, scoring='f1_macro')
    return scores.mean()

study = optuna.create_study(direction='maximize')
study.optimize(objective, n_trials=100)
print(study.best_params)
```

---

## 8. data leakage — 6 형태 + 방지

### 8.1 temporal leakage

미래 데이터로 과거 예측 학습.

| 예 | 방지 |
| --- | --- |
| 2024년 데이터로 2020년 주가 예측 | time-series split (§5.5) |
| training 시점에 미래의 평균 / 통계 사용 | 시간 cut-off 명확화 |

### 8.2 feature leakage

정답을 거의 그대로 담은 feature.

| 예 | 방지 |
| --- | --- |
| 결제 완료 → "결제 상태" feature | feature 의 생성 시점 검증 |
| 진단명 예측에 "진단 코드" 포함 | feature audit |

### 8.3 preprocessing leakage

test 데이터의 통계로 train scaling.

```python
# BAD — leakage
scaler = StandardScaler()
X = scaler.fit_transform(X)        # 전체로 fit
X_train, X_test = train_test_split(X)

# GOOD
X_train, X_test = train_test_split(X)
scaler = StandardScaler()
X_train = scaler.fit_transform(X_train)   # train 만 fit
X_test = scaler.transform(X_test)         # test 는 transform 만
```

→ `Pipeline` 으로 자동화하면 안전.

```python
from sklearn.pipeline import Pipeline

pipe = Pipeline([
    ('scaler', StandardScaler()),
    ('model', LogisticRegression()),
])

# CV 내부에서 fold 마다 자동으로 fit/transform 분리
cross_val_score(pipe, X, y, cv=5)
```

### 8.4 duplicate leakage

train / test 에 같은 sample.

| 방지 | 방법 |
| --- | --- |
| sample id 로 dedup | `df.drop_duplicates(subset='id')` |
| hash 기반 dedup | row hash 비교 |
| near-duplicate (이미지 / 텍스트) | perceptual hash / embedding 유사도 |

### 8.5 group leakage

같은 group (user / patient / 영상) 이 train + test 양쪽.

→ §5.4 group k-fold 사용.

### 8.6 target leakage

feature 가 학습 시점에 알 수 없는 정보를 포함.

| 예 | 방지 |
| --- | --- |
| 의료 차트의 "퇴원 진단" 으로 입원 시 예측 | feature 시점 매핑 |
| 추천에 "사용자가 이미 본 상품" 의 다음 행동 | 시점 절단 |

---

## 9. 평가 결과 보고 표준

### 9.1 보고에 포함

| 항목 | 의미 |
| --- | --- |
| metric 이름 + 정의 | F1 / AUC / RMSE 등 + 어느 정의 (macro / weighted / micro) |
| 평가 데이터 크기 | N |
| 클래스 분포 (분류) | 불균형 여부 |
| CV 종류 + fold 수 | 5-fold StratifiedKFold |
| 평균 ± 표준편차 | `0.85 ± 0.02` |
| baseline 비교 | random / majority class / 이전 모델 |
| confusion matrix (분류) | 클래스별 오분류 패턴 |
| residual plot (회귀) | 오차 분포 |

### 9.2 단일 숫자 vs 분포

```python
scores = cross_val_score(model, X, y, cv=10, scoring='f1_macro')
# 1개 숫자만 보고 X
# scores.mean() = 0.85

# 분포까지
print(f"mean = {scores.mean():.4f}")
print(f"std = {scores.std():.4f}")
print(f"min = {scores.min():.4f}, max = {scores.max():.4f}")
```

→ 평균이 같아도 분산이 크면 불안정. seed 별 / fold 별 분포까지 보고.

### 9.3 통계적 유의성

두 모델 비교 시 차이가 우연 vs 유의한가.

```python
from scipy.stats import ttest_rel

# 같은 fold 의 두 모델 점수 paired test
t, p = ttest_rel(scores_model_a, scores_model_b)
# p < 0.05 면 유의한 차이
```

→ 작은 차이 (예: 0.851 vs 0.853) 는 보통 유의하지 않음. 강력한 결론 X.

---

## 10. 절차 — 표준 ML 학습 / 평가 파이프라인

```
1. 데이터 탐색 (EDA)
   ├─ 결측 / 이상치 / 분포 확인
   └─ [[../foundations/data-visualization]] 의 plot 사용

2. test set 분리 (한 번만, 끝까지 봉인)

3. CV 설정 (k-fold / stratified / group / time-series 중 선택)

4. Pipeline 구성
   ├─ preprocessing (scaling / encoding / imputation)
   ├─ feature engineering
   └─ model

5. hyperparameter tuning (CV 안에서)
   └─ best_params_ 선택

6. 최종 모델 학습 (train + val 전체)

7. test set 1 회 평가
   ├─ metric + 표준편차
   ├─ confusion matrix / residual plot
   └─ baseline 비교

8. 평가 결과 보고 (§9.1 형식)

9. (production) 모니터링 / 재학습 pipeline
```

---

## 11. 흔한 실패 모드

| 실패 | 원인 | 정정 |
| --- | --- | --- |
| test 점수가 매우 좋음 (의심) | leakage | §8 의 6 형태 audit |
| CV 점수가 fold 마다 매우 다름 | 작은 데이터 / 불균형 | stratified k-fold / 데이터 추가 / regularization |
| hyperparameter 변경마다 결과 다름 | 작은 데이터 / 분산 | nested CV / 더 큰 CV / 평균만 사용 |
| seed 변경 시 결과 완전 다름 | 모델 / 데이터 불안정 | seed 여러 개 평균 + 표준편차 보고 |
| production 에서 성능 급락 | distribution shift / leakage | 모니터링 / 재학습 / leakage 재검증 |
| GridSearch 가 너무 오래 걸림 | grid 너무 큼 | RandomSearch / Optuna / HalvingGridSearch |
| pipeline 없이 수동 scaling | preprocessing leakage | sklearn Pipeline 강제 |
| CV 점수와 test 점수가 크게 다름 | CV / test 분포 다름 | stratify / temporal split 재검증 |
| 단일 metric 으로 결정 | trade-off 무시 | confusion matrix + 비용 분석 |

---

## 12. 참고

- [[machine-learning|↑ machine-learning hub]]
- [[concepts]] §9 평가 metric
- [[loss-and-gradient-descent]]
- [[../foundations/data-analysis]]
- [[../foundations/data-visualization]]
- scikit-learn — `model_selection` / `pipeline` / `metrics`
- Optuna — https://optuna.org
- "Hands-On Machine Learning" (Géron) §2-3
- Andrew Ng — "Machine Learning Yearning" (free PDF) — 평가 / 디버깅

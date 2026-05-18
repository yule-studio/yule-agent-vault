---
title: "data-visualization — matplotlib / seaborn / plotly 차트 가이드"
kind: knowledge
project: ai
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-19T05:00:00+09:00
tags: [ai, foundations, visualization, matplotlib, seaborn, plotly, charts]
home_hub: foundations
related:
  - "[[foundations]]"
  - "[[data-analysis]]"
  - "[[../machine-learning/concepts]]"
  - "[[../machine-learning/training-and-evaluation]]"
---

# data-visualization — matplotlib / seaborn / plotly 차트 가이드

**[[foundations|↑ foundations]]**

---

## 1. 목적

본 문서는 ML / 분석에서 사용하는 차트 종류 / 라이브러리 / 코드를 정리한다.

본 문서가 정의하는 것:
- "이 데이터엔 이 차트" 의 선택 기준
- matplotlib 의 객체 모델 (Figure / Axes / Artist)
- seaborn 의 고수준 API (matplotlib 위)
- plotly 의 interactive 차트
- ML 학습 / 평가 시 표준 시각화 (학습 곡선 / confusion matrix / ROC / SHAP)
- 차트의 안티패턴

본 문서가 정의하지 않는 것:
- pandas / numpy 사용법 — [[data-analysis]]
- 디자인 / 색 이론 / 접근성 일반론 — 외부
- 대시보드 (Tableau / PowerBI / Grafana) — 외부

---

## 2. 범위

| 구분 | 포함 |
| --- | --- |
| 라이브러리 | matplotlib 3.8+ / seaborn 0.13+ / plotly 5.x |
| 환경 | Jupyter notebook / Python script |
| 제외 | D3.js / 웹 시각화 — 별도 |
| 제외 | GIS / 지도 (folium / geopandas) — 별도 |

---

## 3. 용어

| 용어 | 정의 |
| --- | --- |
| **Figure** | matplotlib 의 최상위 캔버스. |
| **Axes** | Figure 안의 plot 영역 (좌표축 1쌍). 흔히 "subplot". |
| **Artist** | Figure / Axes 안의 모든 시각 요소 (Line, Text, Patch). |
| **subplot** | 1 Figure 안의 여러 Axes 배치. |
| **colormap** | 수치값을 색으로 매핑. |
| **aesthetic mapping** | seaborn / ggplot 의 데이터 → 시각 속성 매핑 (x / y / hue / size). |

---

## 4. 차트 종류 선택 기준

데이터의 type 과 보여주려는 관계에 따라 차트 선택.

### 4.1 1 변수 (single variable)

| 데이터 type | 보여주려는 것 | 차트 |
| --- | --- | --- |
| 수치형 (연속) | 분포 모양 | histogram / KDE |
| 수치형 (연속) | 5 수치 요약 | boxplot / violinplot |
| 카테고리 | 빈도 | bar / count plot |
| 카테고리 | 비율 | pie (회피, 대신 bar) |
| 시간 | 추이 | line |

### 4.2 2 변수 (bivariate)

| x type | y type | 차트 |
| --- | --- | --- |
| 수치형 | 수치형 | scatter / hexbin / 2D density |
| 카테고리 | 수치형 | boxplot / violin / strip / swarm |
| 카테고리 | 카테고리 | heatmap / mosaic |
| 시간 | 수치형 | line / area |
| 수치형 | 카테고리 | histogram 분리 (hue) |

### 4.3 3+ 변수 (multivariate)

| 목적 | 차트 |
| --- | --- |
| 모든 쌍 관계 | scatter matrix / pairplot |
| 상관 | heatmap |
| 그룹 비교 | grouped bar / facet grid (small multiples) |
| 시간 + 카테고리 | stacked line / area |
| 고차원 군집 / 차원 축소 | scatter (PCA/t-SNE/UMAP) + color |

### 4.4 ML 전용

| 목적 | 차트 |
| --- | --- |
| 학습 진행 | line (epoch vs loss) |
| 분류 평가 | confusion matrix heatmap |
| 이진 분류 threshold | ROC curve / PR curve |
| 회귀 평가 | residual plot / Q-Q plot |
| feature 중요도 | bar (sorted) / SHAP summary |
| 임베딩 | 2D scatter (UMAP / t-SNE) + color |

---

## 5. matplotlib — 객체 모델

### 5.1 두 가지 API

| API | 사용 | 권장 |
| --- | --- | --- |
| **pyplot (procedural)** | `plt.plot()`, `plt.title()` | 빠른 탐색 / 노트북 |
| **OO (객체)** | `fig, ax = plt.subplots()` → `ax.plot(...)` | production / 다중 subplot |

### 5.2 객체 모델

```
Figure                         ← 캔버스 1개
├── Axes (subplot 1)            ← plot 영역
│   ├── Line2D, Patch, Text, ...   ← Artist
│   ├── XAxis, YAxis
│   └── Legend, Title
├── Axes (subplot 2)
└── ...
```

### 5.3 표준 OO 사용

```python
import matplotlib.pyplot as plt

fig, ax = plt.subplots(figsize=(8, 5))
ax.plot(x, y, color="steelblue", linewidth=2, label="series A")
ax.scatter(x2, y2, color="orange", s=20, alpha=0.5, label="series B")
ax.set_xlabel("Date")
ax.set_ylabel("Value")
ax.set_title("Daily Value")
ax.legend(loc="upper left")
ax.grid(True, alpha=0.3)
fig.tight_layout()
fig.savefig("output.png", dpi=150, bbox_inches="tight")
plt.show()
```

### 5.4 다중 subplot

```python
fig, axes = plt.subplots(2, 3, figsize=(15, 8), sharex=True, sharey=False)

axes[0, 0].plot(x1, y1); axes[0, 0].set_title("A")
axes[0, 1].plot(x2, y2); axes[0, 1].set_title("B")
axes[0, 2].plot(x3, y3); axes[0, 2].set_title("C")
axes[1, 0].plot(x4, y4); axes[1, 0].set_title("D")
axes[1, 1].plot(x5, y5); axes[1, 1].set_title("E")
axes[1, 2].plot(x6, y6); axes[1, 2].set_title("F")

fig.suptitle("Multi-panel")
fig.tight_layout()
```

### 5.5 스타일 / 색

```python
# 글로벌 스타일
plt.style.use("seaborn-v0_8-whitegrid")   # 다른 옵션: ggplot, fivethirtyeight, ...

# 색 (제한된 팔레트 사용)
colors = ["#4C72B0", "#DD8452", "#55A868", "#C44E52", "#8172B3"]

# colormap
import matplotlib.cm as cm
norm = plt.Normalize(0, 100)
ax.scatter(x, y, c=values, cmap="viridis", norm=norm)
fig.colorbar(ax.collections[0], ax=ax, label="value")
```

---

## 6. matplotlib — 차트 종류별 코드

### 6.1 line plot

```python
fig, ax = plt.subplots()
ax.plot(x, y, marker="o", markersize=4, linestyle="-")
ax.fill_between(x, y - err, y + err, alpha=0.2)    # 신뢰구간
```

### 6.2 bar plot

```python
fig, ax = plt.subplots()
ax.bar(categories, values, color="steelblue", edgecolor="black")
ax.set_xticklabels(categories, rotation=45, ha="right")

# horizontal
ax.barh(categories, values)

# stacked
ax.bar(x, y1, label="A")
ax.bar(x, y2, bottom=y1, label="B")

# grouped
import numpy as np
width = 0.35
ax.bar(np.arange(len(x)) - width/2, y1, width, label="A")
ax.bar(np.arange(len(x)) + width/2, y2, width, label="B")
ax.set_xticks(np.arange(len(x)), x)
```

### 6.3 histogram

```python
fig, ax = plt.subplots()
ax.hist(data, bins=30, color="steelblue", edgecolor="black", alpha=0.7)
ax.set_xlabel("value")
ax.set_ylabel("frequency")

# 두 분포 비교
ax.hist(data1, bins=30, alpha=0.5, label="A")
ax.hist(data2, bins=30, alpha=0.5, label="B")
ax.legend()
```

### 6.4 scatter

```python
fig, ax = plt.subplots()
ax.scatter(x, y, s=20, c=values, cmap="viridis", alpha=0.6)
fig.colorbar(ax.collections[0], ax=ax, label="value")

# regression 선 추가
import numpy as np
m, b = np.polyfit(x, y, 1)
ax.plot(x, m*x + b, color="red", linestyle="--")
```

### 6.5 boxplot

```python
fig, ax = plt.subplots()
ax.boxplot([group_a, group_b, group_c], labels=["A", "B", "C"])
```

### 6.6 heatmap (matrix)

```python
import numpy as np

fig, ax = plt.subplots()
im = ax.imshow(matrix, cmap="coolwarm", aspect="auto")
ax.set_xticks(range(len(col_labels)))
ax.set_yticks(range(len(row_labels)))
ax.set_xticklabels(col_labels, rotation=45, ha="right")
ax.set_yticklabels(row_labels)
fig.colorbar(im, ax=ax)

# 값 annotation
for i in range(matrix.shape[0]):
    for j in range(matrix.shape[1]):
        ax.text(j, i, f"{matrix[i, j]:.2f}", ha="center", va="center")
```

---

## 7. seaborn — 고수준 통계 차트

matplotlib 위에서 DataFrame + 통계 차트를 1 줄로.

### 7.1 setup

```python
import seaborn as sns
sns.set_theme(style="whitegrid", palette="muted")
```

### 7.2 1 변수

```python
sns.histplot(df, x="score", bins=30, kde=True)
sns.kdeplot(df, x="score", hue="category", fill=True)
sns.boxplot(df, x="category", y="score")
sns.violinplot(df, x="category", y="score", hue="passed", split=True)
sns.stripplot(df, x="category", y="score", jitter=True, alpha=0.5)
sns.swarmplot(df, x="category", y="score")     # 비포개짐
sns.countplot(df, x="category")
```

### 7.3 2 변수 + hue

```python
sns.scatterplot(df, x="age", y="income", hue="gender", size="experience")
sns.lineplot(df, x="date", y="value", hue="metric", ci="sd")     # 신뢰구간 자동
sns.regplot(df, x="x", y="y")                                     # regression
sns.lmplot(df, x="x", y="y", hue="group", col="region")           # facet
sns.jointplot(df, x="x", y="y", kind="hex")                       # marginal + scatter
```

### 7.4 다변수

```python
# pair plot — 모든 numeric 쌍의 scatter + 대각선 분포
sns.pairplot(df, hue="category", diag_kind="kde")

# correlation heatmap
corr = df.corr(numeric_only=True)
sns.heatmap(corr, annot=True, fmt=".2f", cmap="coolwarm", center=0, square=True)

# facet grid (small multiples)
g = sns.FacetGrid(df, col="region", row="year", hue="category")
g.map(sns.scatterplot, "x", "y")
g.add_legend()
```

### 7.5 catplot (통합 카테고리)

```python
sns.catplot(df, x="category", y="score",
            hue="passed", col="region",
            kind="box",      # box / violin / strip / swarm / bar / point / count
            height=4, aspect=1.2)
```

---

## 8. plotly — interactive 차트

웹 / 노트북에서 hover / zoom / pan / 비교가 필요할 때.

### 8.1 plotly express (간단)

```python
import plotly.express as px

fig = px.line(df, x="date", y="value", color="metric", title="Time series")
fig = px.scatter(df, x="x", y="y", color="category", size="weight",
                  hover_data=["name", "country"])
fig = px.bar(df, x="category", y="value", color="region", barmode="group")
fig = px.box(df, x="category", y="score", points="all")
fig = px.histogram(df, x="score", color="category", marginal="box", nbins=30)
fig = px.heatmap(corr, color_continuous_scale="RdBu_r")

fig.show()
fig.write_html("chart.html")
fig.write_image("chart.png", scale=2)
```

### 8.2 graph_objects (세밀 제어)

```python
import plotly.graph_objects as go

fig = go.Figure()
fig.add_trace(go.Scatter(x=df.date, y=df.actual,  mode="lines+markers", name="actual"))
fig.add_trace(go.Scatter(x=df.date, y=df.predict, mode="lines", name="predicted",
                          line=dict(dash="dash")))
fig.update_layout(
    title="Forecast vs Actual",
    xaxis_title="Date",
    yaxis_title="Value",
    legend=dict(x=0.02, y=0.98),
    hovermode="x unified",
)
fig.show()
```

---

## 9. ML 시각화 — 표준 차트

### 9.1 학습 곡선

```python
fig, ax = plt.subplots(figsize=(8, 5))
ax.plot(epochs, train_loss, label="train", color="C0")
ax.plot(epochs, val_loss,   label="val",   color="C1")
ax.set_xlabel("Epoch")
ax.set_ylabel("Loss")
ax.set_title("Learning Curve")
ax.legend()
ax.grid(alpha=0.3)
```

### 9.2 confusion matrix

```python
from sklearn.metrics import confusion_matrix
import seaborn as sns

cm = confusion_matrix(y_true, y_pred)
fig, ax = plt.subplots(figsize=(6, 5))
sns.heatmap(cm, annot=True, fmt="d", cmap="Blues",
             xticklabels=class_names, yticklabels=class_names, ax=ax)
ax.set_xlabel("Predicted")
ax.set_ylabel("True")
ax.set_title(f"Confusion Matrix (accuracy={cm.trace()/cm.sum():.3f})")
```

### 9.3 ROC / PR curve

```python
from sklearn.metrics import roc_curve, auc, precision_recall_curve

# ROC
fpr, tpr, _ = roc_curve(y_true, y_score)
roc_auc = auc(fpr, tpr)

fig, ax = plt.subplots(figsize=(6, 5))
ax.plot(fpr, tpr, label=f"AUC = {roc_auc:.3f}")
ax.plot([0, 1], [0, 1], color="gray", linestyle="--", label="random")
ax.set_xlabel("False Positive Rate")
ax.set_ylabel("True Positive Rate")
ax.set_title("ROC Curve")
ax.legend()

# PR
precision, recall, _ = precision_recall_curve(y_true, y_score)
fig, ax = plt.subplots()
ax.plot(recall, precision)
ax.set_xlabel("Recall"); ax.set_ylabel("Precision")
```

### 9.4 residual plot (회귀)

```python
residuals = y_true - y_pred

fig, axes = plt.subplots(1, 2, figsize=(12, 4))
axes[0].scatter(y_pred, residuals, alpha=0.5)
axes[0].axhline(y=0, color="red", linestyle="--")
axes[0].set_xlabel("Predicted"); axes[0].set_ylabel("Residual")
axes[0].set_title("Residuals vs Predicted")

# Q-Q plot (정규성)
from scipy import stats
stats.probplot(residuals, dist="norm", plot=axes[1])
axes[1].set_title("Q-Q Plot (normality of residuals)")
```

### 9.5 feature 중요도

```python
feature_importance = pd.DataFrame({
    "feature": feature_names,
    "importance": model.feature_importances_,
}).sort_values("importance", ascending=True)

fig, ax = plt.subplots(figsize=(8, max(4, len(feature_importance) * 0.3)))
ax.barh(feature_importance.feature, feature_importance.importance, color="steelblue")
ax.set_xlabel("Importance")
ax.set_title("Feature Importance")
```

### 9.6 SHAP

```python
import shap

explainer = shap.TreeExplainer(model)
shap_values = explainer.shap_values(X_test)

shap.summary_plot(shap_values, X_test)            # global
shap.dependence_plot("age", shap_values, X_test)  # 특정 feature
shap.force_plot(explainer.expected_value, shap_values[0], X_test.iloc[0])    # 단일 prediction
```

### 9.7 임베딩 시각화

```python
import umap
import seaborn as sns

reducer = umap.UMAP(n_components=2, n_neighbors=15, min_dist=0.1)
emb_2d = reducer.fit_transform(embeddings)

fig, ax = plt.subplots(figsize=(10, 8))
sns.scatterplot(x=emb_2d[:, 0], y=emb_2d[:, 1],
                 hue=labels, palette="tab10", s=20, alpha=0.7, ax=ax)
ax.set_title("UMAP embedding")
```

---

## 10. small multiples — facet 비교

여러 그룹을 같은 차트로 비교할 때 한 plot 안 hue 보다 facet 이 명확.

```python
# seaborn FacetGrid
g = sns.FacetGrid(df, col="country", col_wrap=4, height=3, sharey=True)
g.map(sns.histplot, "score", bins=30)
g.set_titles("{col_name}")

# matplotlib subplots
fig, axes = plt.subplots(2, 4, figsize=(16, 8), sharey=True)
for ax, (name, group) in zip(axes.flat, df.groupby("country")):
    ax.hist(group.score, bins=30)
    ax.set_title(name)
```

---

## 11. 저장 / 출력 표준

```python
# 비트맵 (PNG / JPG)
fig.savefig("chart.png", dpi=150, bbox_inches="tight")
fig.savefig("chart.jpg", dpi=150, bbox_inches="tight", facecolor="white")

# 벡터 (PDF / SVG) — 무손실 확대
fig.savefig("chart.pdf", bbox_inches="tight")
fig.savefig("chart.svg", bbox_inches="tight")

# plotly
fig.write_html("chart.html")              # interactive
fig.write_image("chart.png", scale=2)     # kaleido 필요
```

| 용도 | 포맷 |
| --- | --- |
| 논문 / 보고서 | PDF / SVG (벡터) |
| 웹 / 슬라이드 | PNG (dpi=150+) |
| interactive | HTML (plotly) |
| 큰 사진 같이 | JPG |

---

## 12. 안티패턴 — 피해야 할 차트

| 안티패턴 | 문제 | 대안 |
| --- | --- | --- |
| 3D bar / pie | 비교 어려움 / 왜곡 | 2D bar / 정렬 bar |
| pie chart (slice > 5) | 비율 비교 어려움 | bar (sorted) |
| dual y-axis | 잘못된 결론 유도 | normalize / 두 plot 분리 |
| 시작이 0 이 아닌 bar | 차이 과장 | y=0 시작 |
| 너무 많은 색 | 구분 어려움 | 6 색 이하 / facet |
| chart junk (3D / 패턴 채움 / 광고) | 인지 부담 | 단순화 |
| 회색 + 회색 | 대비 부족 | 한 강조 색 |
| 정렬 안 된 bar | 비교 어려움 | 값 기준 정렬 |
| y / x 라벨 없음 | 단위 불명 | 항상 라벨 + 단위 |
| 색만으로 정보 | 색맹 / 흑백 출력 | shape / pattern / label 보조 |

→ Edward Tufte 의 "data-ink ratio" 원칙 — 잉크 / 픽셀의 최대한이 데이터 표현에 쓰여야 함.

---

## 13. 색 / 색맹 대응

| 권장 팔레트 | 사용 |
| --- | --- |
| `viridis` / `plasma` / `cividis` | 연속 수치 (색맹 친화) |
| `coolwarm` / `RdBu_r` | 0 중심 양/음 |
| `Set2` / `tab10` | 카테고리 (10 이내) |
| `Greys` | 단색 grayscale |

회피:
- `rainbow` / `jet` — 색맹 친화 X, 수치 선형성 잘못 표현

```python
# 색맹 시뮬레이션 (matplotlib 4.x+)
import matplotlib.pyplot as plt
plt.rcParams["axes.prop_cycle"] = plt.cycler(color=["#4C72B0", "#DD8452", "#55A868"])
```

---

## 14. 노트북에서 inline 표시 / 크기

```python
# Jupyter 기본 (matplotlib)
%matplotlib inline

# 고해상도 (retina)
%config InlineBackend.figure_format = "retina"

# figure 크기 default 조정
import matplotlib as mpl
mpl.rcParams["figure.figsize"] = (8, 5)
mpl.rcParams["font.size"] = 11
```

---

## 15. 흔한 실패 모드

| 실패 | 원인 | 정정 |
| --- | --- | --- |
| `plt.show()` 후 다시 plot | Figure 닫힘 | 다시 `plt.subplots()` |
| `ax.set_xticklabels()` 라벨 어긋남 | tick 위치 명시 안 함 | `ax.set_xticks()` + `set_xticklabels()` 같이 |
| 한글 깨짐 | font 미설정 | `mpl.rcParams["font.family"] = "AppleGothic"` (mac) / `NanumGothic` (linux) |
| 큰 데이터 scatter 가 검은 덩어리 | overlap | `alpha=0.1` + `s=5` 또는 hexbin / 2D density |
| histogram bin 너무 적 / 많음 | bins 선택 잘못 | Freedman-Diaconis / Sturges (auto) |
| dual y-axis 로 왜곡 | scale 다름 | normalize / 분리 |
| seaborn 이 데이터 큰데 느림 | 모든 sample plot | sample / hexbin |
| plotly HTML 파일 매우 큼 | 모든 데이터 embed | sample / `to_json` + WebGL |
| 색만으로 정보 | 색맹 / 흑백 | shape / pattern 추가 |
| 시계열 x 축 라벨 겹침 | tick 너무 많음 | `MaxNLocator` / `DateLocator` / rotation |

---

## 16. 참고

- [[foundations|↑ foundations hub]]
- [[data-analysis]]
- [[../machine-learning/concepts]]
- [[../machine-learning/training-and-evaluation]]
- matplotlib gallery — https://matplotlib.org/stable/gallery/
- seaborn gallery — https://seaborn.pydata.org/examples/
- plotly express docs — https://plotly.com/python/plotly-express/
- "Fundamentals of Data Visualization" (Claus Wilke) — 무료 온라인 책
- Edward Tufte — "The Visual Display of Quantitative Information"
- ColorBrewer — https://colorbrewer2.org (색 팔레트)

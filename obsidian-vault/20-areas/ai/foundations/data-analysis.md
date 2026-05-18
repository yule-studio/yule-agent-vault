---
title: "data-analysis — NumPy + Pandas 실용 가이드"
kind: knowledge
project: ai
agent: engineering-agent/tech-lead
status: current
created_at: 2026-05-19T04:30:00+09:00
tags: [ai, foundations, numpy, pandas, data-analysis, practice]
home_hub: foundations
related:
  - "[[foundations]]"
  - "[[data-visualization]]"
  - "[[../machine-learning/concepts]]"
  - "[[../machine-learning/training-and-evaluation]]"
---

# data-analysis — NumPy + Pandas 실용 가이드

**[[foundations|↑ foundations]]**

---

## 1. 목적

본 문서는 ML / 데이터 분석에서 사용하는 NumPy 와 Pandas 의 표준 패턴을 정의한다.

본 문서가 정의하는 것:
- NumPy array / shape / broadcasting / vectorization
- Pandas DataFrame 의 생성 / 조회 / 변환 / 결합
- groupby / merge / pivot / window 함수
- 결측치 / 이상치 / 중복 처리
- 기술 통계 / 분포 / 상관 분석
- 시간 / 카테고리 데이터의 다루기
- 메모리 / 속도 최적화

본 문서가 정의하지 않는 것:
- 시각화 — [[data-visualization]]
- ML 모델 학습 / 평가 — [[../machine-learning/concepts]], [[../machine-learning/training-and-evaluation]]
- SQL 쿼리 — [[../../database/database]]

---

## 2. 범위

| 구분 | 포함 |
| --- | --- |
| 대상 도구 | NumPy 1.26+ / Pandas 2.x |
| 데이터 규모 | 단일 머신 메모리 (수 GB 까지) |
| 제외 | Dask / Polars / Spark (대용량) — 별도 |
| 제외 | 텍스트 / 이미지 specific 처리 — 별도 |

---

## 3. 용어

| 용어 | 정의 |
| --- | --- |
| **ndarray** | NumPy 의 다차원 배열. |
| **shape** | 각 축의 길이 tuple. `(B, C, H, W)` 등. |
| **broadcasting** | 다른 shape 사이의 element-wise 연산을 자동 확장. |
| **vectorization** | for loop 없이 array 전체 연산으로 처리. |
| **Series** | Pandas 의 1D 라벨 array. |
| **DataFrame** | Pandas 의 2D 라벨 table. |
| **Index** | row / column 의 라벨. |
| **dtype** | tensor / array 의 원소 type (`int64`, `float32`, `object`, `category`). |
| **NaN** | 결측치. float / object 만 표현 가능. |

---

## 4. NumPy 핵심

### 4.1 array 생성

```python
import numpy as np

a = np.array([1, 2, 3])                       # 1D
b = np.array([[1, 2, 3], [4, 5, 6]])          # 2D, shape (2, 3)
c = np.zeros((3, 4))                           # shape (3, 4), 모두 0
d = np.ones((2, 2), dtype=np.float32)
e = np.arange(0, 10, 2)                        # [0, 2, 4, 6, 8]
f = np.linspace(0, 1, 5)                       # [0., 0.25, 0.5, 0.75, 1.]
g = np.random.randn(100, 10)                   # 표준 정규 (100x10)
h = np.eye(3)                                   # 3x3 단위 행렬
```

### 4.2 shape 조작

```python
a.shape                  # (10,)
a.reshape(2, 5)          # (2, 5)
a.reshape(-1, 5)         # -1 = 자동 계산
a[:, None]               # (10, 1) — 새 축 추가 (a.reshape(-1, 1) 와 같음)
a.flatten()              # (10,)
np.transpose(a, (1, 0))  # 축 순서 변경 (.T)
np.concatenate([a, b], axis=0)
np.stack([a, b], axis=0) # 새 축으로 쌓기
np.split(a, 5, axis=0)
```

### 4.3 indexing

```python
a = np.arange(20).reshape(4, 5)
# array([[ 0,  1,  2,  3,  4],
#        [ 5,  6,  7,  8,  9],
#        [10, 11, 12, 13, 14],
#        [15, 16, 17, 18, 19]])

a[0]                  # 첫 행
a[0, 2]               # 0행 2열 = 2
a[1:3, :]             # 1,2 행 (슬라이싱)
a[:, ::-1]            # 열 역순
a[[0, 2]]             # 0,2 행 (fancy indexing)
a[a > 10]             # boolean mask
a[a > 10] = 0         # 조건 할당
```

### 4.4 broadcasting

shape 가 다른 array 끼리 자동 확장.

```python
a = np.array([[1, 2, 3], [4, 5, 6]])   # (2, 3)
b = np.array([10, 20, 30])              # (3,)

a + b                                    # (2, 3)
# array([[11, 22, 33],
#        [14, 25, 36]])

# 규칙: 뒤쪽 축부터 비교, 크기 같거나 한 쪽이 1 이면 broadcast 가능
# (2, 3) vs (3,) → (2, 3) vs (1, 3) → OK
# (2, 3) vs (2,) → ✗ (axis=1 의 3 vs 2 충돌)
```

### 4.5 vectorization — for loop 제거

```python
# 느림 (Python for loop)
result = []
for i in range(len(a)):
    result.append(a[i] ** 2 + 1)

# 빠름 (NumPy vectorize)
result = a ** 2 + 1     # element-wise

# 통계
a.sum(), a.mean(), a.std(), a.var(), a.min(), a.max()
a.sum(axis=0), a.mean(axis=1)
np.median(a), np.percentile(a, [25, 50, 75])

# linear algebra
np.dot(a, b)           # 행렬 곱 / 내적
a @ b                  # 같은 의미 (Python 3.5+)
np.linalg.inv(M)       # 역행렬
np.linalg.svd(M)       # SVD
np.linalg.eigh(M)      # 고유값 (symmetric)
```

→ NumPy 의 vectorized 연산은 C 로 구현되어 Python for loop 보다 100-1000 배 빠름.

---

## 5. Pandas DataFrame 핵심

### 5.1 생성

```python
import pandas as pd

# dict
df = pd.DataFrame({
    "id": [1, 2, 3, 4],
    "name": ["A", "B", "C", "D"],
    "score": [85.5, 90.0, 78.3, 92.1],
    "passed": [True, True, False, True],
})

# CSV
df = pd.read_csv("data.csv")
df = pd.read_csv("data.csv", dtype={"id": "int32", "name": "category"})
df = pd.read_csv("data.csv", parse_dates=["created_at"])

# 다른 포맷
df = pd.read_parquet("data.parquet")
df = pd.read_excel("data.xlsx", sheet_name="Sheet1")
df = pd.read_json("data.json")
df = pd.read_sql("SELECT * FROM users", conn)
```

### 5.2 검사

```python
df.shape                # (행, 열)
df.dtypes
df.info()
df.describe()           # 기술 통계 (숫자만)
df.describe(include="all")
df.head(10)
df.tail(10)
df.sample(5)
df.columns
df.index
df.memory_usage(deep=True)
```

### 5.3 indexing

```python
df["score"]                   # 1 column (Series)
df[["id", "score"]]           # 다수 column
df[df.score > 80]             # boolean filter

# .loc — 라벨 기반
df.loc[0]                     # index=0 행
df.loc[0, "score"]            # row=0, col="score"
df.loc[df.score > 80, "name"]
df.loc[:, "id":"score"]

# .iloc — 위치 기반
df.iloc[0]                    # 첫 행
df.iloc[0, 2]                 # 0행 2열
df.iloc[:5, :3]               # 처음 5행, 처음 3열

# 다중 조건
df[(df.score > 80) & (df.passed)]
df[df.name.isin(["A", "B"])]
df.query("score > 80 and passed")    # SQL-like
```

### 5.4 변환

```python
# column 추가 / 변환
df["score_rank"] = df.score.rank()
df["grade"] = pd.cut(df.score, bins=[0, 60, 75, 90, 100],
                      labels=["F", "C", "B", "A"])
df["log_score"] = np.log(df.score)

# 함수 적용
df["name_upper"] = df.name.apply(str.upper)
df["score_z"] = df.score.transform(lambda x: (x - x.mean()) / x.std())

# 여러 column 동시
df["score_norm"] = df.score / df.score.max()

# string method
df["name_lower"] = df.name.str.lower()
df["domain"] = df.email.str.extract(r"@(.+)$")

# 타입 변환
df["id"] = df.id.astype("int32")
df["category"] = df.category.astype("category")    # 메모리 효율
```

### 5.5 결측치 처리

```python
# 검사
df.isna().sum()           # column 별 결측 수
df.isna().mean()          # column 별 결측 비율
df.dropna()               # 결측 있는 행 제거
df.dropna(subset=["score"])  # 특정 column 만
df.fillna(0)
df.fillna(df.mean())      # 평균으로
df.fillna(method="ffill") # 앞 값으로
df.fillna({"score": df.score.median(), "name": "unknown"})

# interpolation (시계열)
df.score.interpolate(method="linear")
df.score.interpolate(method="time")
```

### 5.6 중복 처리

```python
df.duplicated().sum()                # 중복 행 수
df.drop_duplicates()                  # 중복 제거
df.drop_duplicates(subset=["id"])     # 특정 column 기준
df.drop_duplicates(subset=["id"], keep="last")
```

### 5.7 이상치 (outlier) — IQR

```python
Q1 = df.score.quantile(0.25)
Q3 = df.score.quantile(0.75)
IQR = Q3 - Q1
mask = (df.score < Q1 - 1.5 * IQR) | (df.score > Q3 + 1.5 * IQR)
df_clean = df[~mask]
```

---

## 6. groupby / aggregation

### 6.1 기본

```python
df.groupby("category").size()
df.groupby("category").mean(numeric_only=True)
df.groupby("category").agg({"score": "mean", "id": "count"})
df.groupby("category")["score"].describe()

# 다중 column 그룹
df.groupby(["category", "grade"]).score.mean()

# 다중 aggregation
df.groupby("category").agg(
    n=("id", "count"),
    avg_score=("score", "mean"),
    max_score=("score", "max"),
    passed_rate=("passed", "mean"),
)
```

### 6.2 transform — 행 수 유지

```python
df["score_minus_group_mean"] = df.score - df.groupby("category").score.transform("mean")
df["rank_in_group"] = df.groupby("category").score.rank(ascending=False)
```

### 6.3 apply — custom 함수

```python
def top_n(group, n=3):
    return group.nlargest(n, "score")

df.groupby("category").apply(top_n, n=3)
```

---

## 7. merge / join / concat

### 7.1 merge — SQL join

```python
left = pd.DataFrame({"id": [1, 2, 3], "x": [10, 20, 30]})
right = pd.DataFrame({"id": [2, 3, 4], "y": [200, 300, 400]})

pd.merge(left, right, on="id", how="inner")    # 교집합 (default)
pd.merge(left, right, on="id", how="left")     # left 모두 유지
pd.merge(left, right, on="id", how="outer")    # 합집합

# 다른 column 이름
pd.merge(left, right, left_on="id", right_on="user_id")

# index 기준
pd.merge(left, right, left_index=True, right_index=True)
```

### 7.2 concat — 수직 / 수평 결합

```python
pd.concat([df1, df2], axis=0)      # 수직 (행 추가)
pd.concat([df1, df2], axis=1)      # 수평 (열 추가)
pd.concat([df1, df2], ignore_index=True)
```

---

## 8. pivot / melt

### 8.1 pivot — long → wide

```python
df = pd.DataFrame({
    "date": ["2026-01", "2026-01", "2026-02", "2026-02"],
    "metric": ["A", "B", "A", "B"],
    "value": [10, 20, 30, 40],
})

df.pivot(index="date", columns="metric", values="value")
# metric   A   B
# date
# 2026-01 10  20
# 2026-02 30  40

# pivot_table — 중복 시 aggregation
df.pivot_table(index="date", columns="metric", values="value", aggfunc="mean")
```

### 8.2 melt — wide → long

```python
wide.melt(id_vars=["date"], value_vars=["A", "B"],
           var_name="metric", value_name="value")
```

---

## 9. window 함수 — rolling / expanding / ewm

```python
# rolling — 고정 크기 window
df["ma_7"] = df.value.rolling(window=7).mean()
df["std_7"] = df.value.rolling(window=7).std()
df["max_7"] = df.value.rolling(window=7).max()

# expanding — 누적
df["cumsum"] = df.value.expanding().sum()
df["cummax"] = df.value.expanding().max()

# ewm — 지수 가중
df["ewma"] = df.value.ewm(span=10).mean()

# shift / diff — 시점 이동
df["yesterday"] = df.value.shift(1)
df["daily_diff"] = df.value.diff()
df["pct_change"] = df.value.pct_change()
```

---

## 10. 시간 데이터

### 10.1 datetime 변환

```python
df["created"] = pd.to_datetime(df.created)
df["created"] = pd.to_datetime(df.created, format="%Y-%m-%d %H:%M:%S")
df["unix"] = df.created.astype("int64") // 10**9
```

### 10.2 추출

```python
df.created.dt.year
df.created.dt.month
df.created.dt.day_of_week     # 0=Monday
df.created.dt.hour
df.created.dt.is_month_start
df.created.dt.strftime("%Y-%m-%d")
```

### 10.3 시간 그룹

```python
df.set_index("created").resample("D").mean()        # 일별
df.set_index("created").resample("W").sum()         # 주별
df.set_index("created").resample("ME").last()       # 월말
```

---

## 11. 카테고리 데이터

### 11.1 category dtype — 메모리 절약

```python
df["country"] = df.country.astype("category")
# 1M 행에서 string vs category → 메모리 10배 이상 감소
```

### 11.2 one-hot encoding

```python
pd.get_dummies(df, columns=["category"], drop_first=True)

# sklearn 의 OneHotEncoder 도 가능
from sklearn.preprocessing import OneHotEncoder
encoder = OneHotEncoder(sparse_output=False, handle_unknown="ignore")
encoded = encoder.fit_transform(df[["category"]])
```

### 11.3 label encoding

```python
from sklearn.preprocessing import LabelEncoder
le = LabelEncoder()
df["category_id"] = le.fit_transform(df.category)
```

---

## 12. 통계 / 분포 / 상관

### 12.1 기술 통계

```python
df.describe()                         # 수치형
df.describe(include="all")            # 모든 column

df.score.skew(), df.score.kurt()      # 왜도 / 첨도
df.score.quantile([0.1, 0.5, 0.9])
df.value_counts(normalize=True)       # 비율
df.nunique()                           # 고유값 수
```

### 12.2 상관 / 공분산

```python
df.corr()                              # Pearson
df.corr(method="spearman")             # 순위 상관
df.corr(method="kendall")
df.cov()
```

### 12.3 카이제곱 / t-test

```python
from scipy.stats import chi2_contingency, ttest_ind

# 두 group 평균 차이
group_a = df[df.group == "A"].score
group_b = df[df.group == "B"].score
t, p = ttest_ind(group_a, group_b)
```

---

## 13. 메모리 / 속도 최적화

### 13.1 dtype 축소

```python
# float64 → float32 (메모리 절반)
for col in df.select_dtypes(include=["float64"]).columns:
    df[col] = df[col].astype("float32")

# int64 → int32 / int16 / int8
df["count"] = pd.to_numeric(df["count"], downcast="integer")
df["pct"] = pd.to_numeric(df["pct"], downcast="float")

# object → category (고유값 적을 때)
df["country"] = df["country"].astype("category")

# 검증
df.memory_usage(deep=True).sum() / 1024**2     # MB
```

### 13.2 chunking — 큰 CSV

```python
chunks = []
for chunk in pd.read_csv("huge.csv", chunksize=100_000):
    # 각 chunk 처리
    chunks.append(chunk[chunk.score > 50])
result = pd.concat(chunks, ignore_index=True)
```

### 13.3 parquet (CSV 대신)

```python
df.to_parquet("data.parquet", compression="snappy")
df = pd.read_parquet("data.parquet")
# CSV vs Parquet — 크기 1/5 ~ 1/10, 로드 5-10 배 빠름, dtype 보존
```

### 13.4 polars / pyarrow — 대안

수십 GB / 매우 빠른 처리 필요 시 [polars](https://pola.rs) (Rust 기반) 또는 pyarrow.

```python
import polars as pl

df = pl.read_parquet("huge.parquet")
df = df.filter(pl.col("score") > 50).groupby("category").agg(pl.col("value").mean())
```

---

## 14. 표준 EDA 절차 (Exploratory Data Analysis)

새 데이터셋 만나면 다음 순서로.

| 단계 | 작업 | 도구 |
| --- | --- | --- |
| 1 | shape / dtype / 메모리 | `df.shape`, `df.info()`, `df.memory_usage()` |
| 2 | 처음 / 끝 / 무작위 sample | `df.head()`, `df.tail()`, `df.sample()` |
| 3 | 기술 통계 | `df.describe(include="all")` |
| 4 | 결측 / 중복 / unique | `isna().sum()`, `duplicated()`, `nunique()` |
| 5 | 분포 시각화 (수치형) | histogram, KDE, boxplot ([[data-visualization]] §5) |
| 6 | 카테고리 빈도 | `value_counts()`, bar plot |
| 7 | 상관 / 관계 | `df.corr()`, scatter / heatmap |
| 8 | 시간 패턴 | resample / rolling |
| 9 | 이상치 / 노이즈 | IQR / 도메인 지식 |
| 10 | feature ↔ target 관계 | groupby + agg + plot |

---

## 15. 흔한 실패 모드

| 실패 | 원인 | 정정 |
| --- | --- | --- |
| `SettingWithCopyWarning` | view vs copy 혼동 | `.copy()` 또는 `.loc` 사용 |
| 평균 계산이 NaN | NaN 이 누적 | `.mean()` 의 `skipna=True` (default), 또는 사전 처리 |
| dtype object 가 비대 | string 컬럼 | `category` 또는 `string[pyarrow]` |
| `merge` 후 행 수 폭증 | many-to-many 의 의도 안 함 | `validate="one_to_one"` / `"one_to_many"` |
| `groupby.apply` 가 매우 느림 | 행 단위 Python loop | vectorize / `transform` / `agg` |
| 메모리 부족 | float64 / object 사용 | dtype 축소 / chunking / parquet |
| `iterrows()` 사용 | Python 객체 변환 비용 | vectorization / `apply` (그래도 느림) / `itertuples` |
| 시간 datetime 비교 안 됨 | tz-aware vs tz-naive 혼합 | `.dt.tz_convert("UTC")` 통일 |
| `.iloc` 대신 `.loc` | 라벨이 정수일 때 혼동 | 명시적 분리 |
| copy 없이 column 추가 | view 변경이 원본 영향 | `.copy()` |

---

## 16. 참고

- [[foundations|↑ foundations hub]]
- [[data-visualization]]
- [[../machine-learning/concepts]]
- [[../machine-learning/training-and-evaluation]]
- "Python for Data Analysis" 3rd (Wes McKinney)
- NumPy docs — https://numpy.org/doc/
- Pandas user guide — https://pandas.pydata.org/docs/user_guide/
- Polars docs — https://pola.rs
- "Modern Pandas" (Tom Augspurger) — 무료 블로그 시리즈

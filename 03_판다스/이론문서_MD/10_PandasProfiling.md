# Part 10 · Pandas Profiling — 한 줄로 끝내는 자동 EDA (보너스)

> **이 파트의 목표**
> - 데이터 탐색(EDA)을 **자동 리포트 한 방**으로 만들어 주는 `ydata-profiling`을 이해한다.
> - `ProfileReport` 객체를 만들고, HTML로 저장하거나 노트북에 띄우는 법을 익힌다.
> - 지금까지 손으로 한 탐색(Part 2)·결측 확인(Part 5)이 리포트의 어디에 해당하는지 연결한다.
>
> **선행 지식**: Part 2(`describe`, `info`, `value_counts`), Part 5(결측치). 이 파트는 **보너스**로, 새 문법보다 "자동화 도구 활용"이 핵심입니다.

예제는 회원 통합 데이터 `store_member_total.csv`(500행 × 21열)를 사용합니다.

```python
import pandas as pd
df = pd.read_csv('store_member_total.csv')
```

---

## 0. 들어가며 — 손으로 하던 탐색을 자동으로

지금까지 데이터를 파악하려고 `info()`, `describe()`, `value_counts()`, `isnull().sum()` 을 **하나씩** 호출했습니다(Part 2·5). 열이 21개라면 이걸 열마다 반복해야 합니다.

**Pandas Profiling(현재 이름 `ydata-profiling`)** 은 이 모든 탐색을 **단 한 줄**로 자동 수행해, 변수별 분포·통계·결측·상관관계를 담은 **인터랙티브 HTML 리포트**로 만들어 줍니다.

> 📊 **분석 프로세스상 위치**: Part 2(탐색)·Part 5(결측 점검)를 **자동화**하는 도구입니다. 새 데이터를 받았을 때 가장 먼저 돌려 전체 그림을 빠르게 보는 용도로 씁니다.

> 💡 자동 리포트가 편하지만, **결과를 읽고 해석하는 능력**은 앞 단계(Part 2~9)에서 익힌 개념이 있어야 가능합니다. 도구는 탐색을 빠르게 할 뿐, 판단은 사람이 합니다.

---

## 1. 설치와 불러오기

**개념**: 외부 라이브러리이므로 먼저 설치해야 합니다(Part 0의 "라이브러리" 개념).

```python
# 설치 (한 번만) — 노트북에서는 ! 를 붙여 실행
!pip install ydata-profiling
```
```python
from ydata_profiling import ProfileReport
```

> ⚠️ 설치 후 **커널(런타임) 재시작**이 필요할 수 있습니다. 의존성이 많아 설치에 시간이 다소 걸립니다.

### 주요 기능(리포트가 담는 것)

| 기능 | 내용 |
|---|---|
| 종합 요약(Overview) | 행·열 수, 변수 타입, 전체 결측·중복 비율 |
| 변수별(Variables) | 열마다 분포·평균·최소/최대·표준편차·고유값 |
| 결측치(Missing) | 열별 결측 수와 패턴을 시각화 |
| 상관관계(Correlations) | 변수 간 관계를 매트릭스로 |
| 중복(Duplicate) | 중복 행 탐지 |

---

## 2. `ProfileReport()` — 리포트 객체 만들기

**개념**: 데이터프레임을 넣으면 분석을 수행하는 **리포트 객체**를 만듭니다.

**원리**: 객체를 생성하는 순간 모든 열을 훑어 통계·결측·상관을 계산해 둡니다. 이후 그 결과를 파일로 저장하거나 화면에 띄웁니다.

**문법 / 인수**
```python
ProfileReport(df, title=None, minimal=False)
```

| 인수 | 의미 | 기본값 |
|---|---|---|
| `df` | 분석할 데이터프레임 | (필수) |
| `title` | 리포트 제목 | None |
| `minimal` | 가벼운 모드(상관 등 무거운 계산 생략) | `False` |

**사용법**
```python
profile = ProfileReport(df, title="회원 데이터 리포트")
```
> 💡 데이터가 크면 `minimal=True`로 만들면 무거운 상관관계 계산을 줄여 훨씬 빠릅니다.

---

## 3. 리포트 출력 — `to_file()` / `to_notebook_iframe()`

만든 리포트를 **저장**하거나 **화면에 표시**합니다.

### 3-1. `to_file()` — HTML 파일로 저장
```python
profile.to_file("output.html")     # 현재 폴더에 HTML 리포트 생성
```
> 생성된 `output.html`을 브라우저로 열면 클릭하며 탐색할 수 있는 인터랙티브 보고서가 나옵니다. 팀과 공유하기 좋습니다.

### 3-2. `to_notebook_iframe()` — 노트북 안에 바로 표시
```python
profile.to_notebook_iframe()       # 주피터 노트북 셀에 리포트 임베드
```

### 3-3. `df.profile_report()` — 더 짧은 방법
데이터프레임에 바로 붙여 쓰는 단축 형태도 있습니다.
```python
report = df.profile_report()       # ProfileReport(df) 와 같음
report.to_file("output.html")
```

---

## 4. 리포트 읽는 법 — 앞 파트와 연결

자동 리포트의 각 항목은 우리가 손으로 배운 것과 1:1로 대응합니다.

| 리포트 항목 | 우리가 배운 것 | 어디서 |
|---|---|---|
| Overview의 행·열·타입 | `df.shape`, `df.info()` | Part 2 |
| 변수별 평균·분위수·분포 | `df.describe()` | Part 2 |
| 범주형 빈도 | `value_counts()` | Part 2 |
| Missing values | `isnull().sum()` | Part 5 |
| Correlations | 변수 간 상관 | (심화) |
| Duplicate rows | 중복 점검 | (심화) |

> 📊 즉 프로파일링은 **새 도구라기보다, 배운 탐색을 자동화한 결과물**입니다. 리포트의 "Missing"에서 결측이 많은 열을 보면 Part 5의 처리(삭제·대체)로, "Variables"에서 한쪽으로 치우친 분포를 보면 Part 5의 이상치 점검으로 자연스럽게 이어집니다.

---

## 5. 데이터 분석 프로세스에서의 활용

```python
from ydata_profiling import ProfileReport
import pandas as pd

# ① 새 데이터 받자마자 전체 그림 보기
df = pd.read_csv('01_Contract_Data.csv')
ProfileReport(df, title="계약 데이터 개요", minimal=True).to_file("개요.html")

# ② 리포트에서 발견한 문제를 직접 처리 (Part 5)
#   - 결측 많은 열 → fillna / dropna
#   - 치우친 분포·이상치 → IQR 처리

# ③ 정제 후 본격 분석 (Part 6~9)
```

**프로파일링으로 빠르게 훑고 → 손으로 정밀하게 처리**하는 흐름이 실무에서 가장 효율적입니다. 자동 리포트는 분석의 **출발점을 단축**해 줍니다.

---

## 6. 자주 하는 실수 / 주의

- [ ] `pip install ydata-profiling` 후 **커널 재시작**을 했는가
- [ ] 큰 데이터에 기본 모드로 돌려 너무 오래 걸린다면 `minimal=True`를 줬는가
- [ ] `to_file()`의 결과는 **HTML 파일** — 브라우저로 열어야 보인다(노트북 표시는 `to_notebook_iframe`)
- [ ] 리포트를 **맹신하지 말 것** — 결측·이상치 처리 판단은 사람이(앞 파트 개념 활용)
- [ ] 한글 깨짐 등 시각화 폰트 이슈는 별도 폰트 설정이 필요할 수 있다

---

## 7. 핵심 요약

1. `ydata-profiling`은 EDA(탐색)를 **자동 리포트 한 줄**로 만든다.
2. `ProfileReport(df, title=, minimal=)` 로 리포트 객체를 만든다.
3. 저장은 `to_file("x.html")`, 노트북 표시는 `to_notebook_iframe()`, 단축은 `df.profile_report()`.
4. 리포트의 Overview·Variables·Missing·Correlations는 각각 `info`·`describe`·`isnull`·상관에 대응한다.
5. **빠르게 훑고(프로파일링) → 정밀하게 처리(앞 파트)** 가 실전 흐름이다.

### 미니 퀴즈
1. 데이터프레임 `df`로 리포트를 만들어 HTML로 저장하는 두 줄은? (답: `p = ProfileReport(df)` → `p.to_file("output.html")`)
2. 큰 데이터에서 리포트를 빠르게 만들려면 어떤 인수를? (답: `minimal=True`)
3. 리포트의 "Missing values"는 우리가 배운 어떤 코드에 해당하나? (답: `df.isnull().sum()`, Part 5)

---

> **🎉 전체 과정 완주!**
> 선행 NumPy + Part 1~9(데이터 분석 프로세스) + 보너스까지 — 데이터를 **불러오고 → 살펴보고 → 정제하고 → 합치고 → 요약하는** 한 사이클을 모두 익혔습니다. 다음 여정은 시각화(matplotlib·seaborn)와 머신러닝입니다. 이 모든 것이 지금 배운 판다스 위에 세워집니다.

# Part 7 · 데이터 합치기 — concat과 merge

> **이 파트의 목표**
> - 여러 데이터프레임을 **단순히 이어 붙이는** `concat`을 익힌다(행 결합/열 결합, 반복 결합).
> - 공통 키(열)를 기준으로 **표를 엮는** `merge`를 익힌다.
> - **inner·outer·left·right 조인**의 차이를 이해하고 상황에 맞게 고른다.
>
> **선행 지식**: Part 1~6(`read_csv`, `groupby`, 인덱스 개념, `for` 반복문).

예제는 주식 분기 데이터(`07_stock_concat1~4.csv`), 주가·재무 데이터, 회원·구매 데이터(`04_store_member.csv`, `04_store_product_1.csv`)를 사용합니다.

```python
import pandas as pd
```

---

## 0. 들어가며 — 데이터는 보통 흩어져 있다

현실에서 데이터는 한 파일에 다 들어 있지 않습니다.

- **시간/구간이 나뉜 경우**: 1분기·2분기·3분기·4분기 매출이 각각 다른 파일.
- **항목이 나뉜 경우**: 한 파일엔 주가, 다른 파일엔 재무정보. 또는 회원정보와 구매내역.

이들을 하나로 합쳐야 비로소 통합 분석이 가능합니다. 합치는 방법은 두 가지이고, **상황에 따라 골라 씁니다.**

| 방법 | 언제 | 비유 |
|---|---|---|
| **`concat`** | 같은 모양의 데이터를 **단순히 쌓기** | 종이를 위아래(또는 좌우)로 이어 붙이기 |
| **`merge`** | **공통 키**로 서로 다른 정보를 **엮기** | 회원번호로 명단과 구매내역을 짝짓기 |

> 📊 **분석 프로세스상 위치**: `[Part 6] 가공·집계 → [Part 7] 결합(지금) → [Part 8~9] 피벗·구조화`. 여러 출처의 데이터를 모아 분석 범위를 넓히는 단계입니다.

---

## 1. `concat()` — 단순히 이어 붙이기

**개념**: 둘 이상의 데이터프레임을 **위아래(행)** 또는 **좌우(열)** 로 그냥 붙입니다.

**원리**: 키를 맞추지 않습니다. `axis=0`이면 행을 아래로 쌓고, `axis=1`이면 열을 옆으로 붙입니다. 그래서 **같은 구조(열 또는 인덱스)** 일 때 자연스럽습니다.

**문법 / 인수**
```python
pd.concat([df1, df2, ...], axis=0, ignore_index=False)
```

| 인수 | 의미 | 기본값 |
|---|---|---|
| `objs` | 합칠 데이터프레임 **리스트** | (필수) |
| `axis` | `0`=행(아래로 쌓기), `1`=열(옆으로 붙이기) | `0` |
| `ignore_index` | 기존 인덱스 무시하고 0부터 새로 | `False` |

### 1-1. 행 방향 결합 (`axis=0`) — 같은 열을 가진 데이터 쌓기
```python
df1 = pd.read_csv('07_stock_concat1.csv', index_col=0)   # 5행
df2 = pd.read_csv('07_stock_concat2.csv', index_col=0)   # 5행

df_concat = pd.concat([df1, df2], axis=0)    # 위아래로 → 10행
df_concat.shape      # (10, 8)
```
> 1분기·2분기 데이터를 쌓아 상반기 전체를 만드는 식입니다.

### 1-2. 반복 결합 — `for` 문으로 여러 파일
```python
df_all = pd.DataFrame()                       # 빈 데이터프레임에서 시작
for i in range(1, 5):                         # 1~4분기
    dfn = pd.read_csv(f'07_stock_concat{i}.csv', index_col=0)
    df_all = pd.concat([df_all, dfn], axis=0, ignore_index=True)
df_all.shape      # (20, 8)   ← 5행 × 4분기
```
> 💡 `ignore_index=True`를 주면 합친 뒤 인덱스가 0,1,2…로 깔끔하게 다시 매겨집니다(원본 인덱스가 0,0,1,1처럼 겹치는 것을 방지).

### 1-3. 열 방향 결합 (`axis=1`) — 항목 추가
```python
extra = pd.read_csv('07_stock_extra.csv', index_col=0)   # PER, EPS 열 (20행)
df_wide = pd.concat([df_all, extra], axis=1)             # 옆으로 붙이기
df_wide.shape      # (20, 10)   ← 기존 8열 + 2열
```
> ⚠️ **`axis=1` 결합은 인덱스(행 순서)가 맞아야** 올바로 붙습니다. 인덱스가 어긋나면 엉뚱한 행끼리 붙거나 결측이 생깁니다. 필요하면 미리 `reset_index(drop=True)`로 맞춥니다.

---

## 2. `merge()` — 공통 키로 엮기

**개념**: 두 표를 **공통된 열(키)** 을 기준으로 짝지어 합칩니다. 엑셀의 VLOOKUP, 데이터베이스의 JOIN과 같습니다.

**원리**: 예를 들어 양쪽에 모두 '종목코드'가 있으면, 같은 종목코드끼리 행을 맞춰 한 줄로 합칩니다. `concat`이 "위치로 쌓기"라면 `merge`는 **"값(키)으로 짝짓기"** 입니다.

**문법 / 인수**
```python
pd.merge(left, right, on=None, how='inner')
```

| 인수 | 의미 | 기본값 |
|---|---|---|
| `left`, `right` | 합칠 두 데이터프레임 | (필수) |
| `on` | 기준이 되는 공통 열 이름 | 자동(공통 열) |
| `how` | 결합 방식(아래 표) | `'inner'` |

### 2-1. 조인 방식 (`how`) — 가장 중요

| `how` | 남기는 행 | 비유 |
|---|---|---|
| `'inner'`(기본) | **양쪽 모두**에 있는 키만 | 교집합 |
| `'outer'` | **양쪽 전체**(없는 쪽은 NaN) | 합집합 |
| `'left'` | **왼쪽 전부** + 매칭되는 오른쪽 | 왼쪽 기준 |
| `'right'` | **오른쪽 전부** + 매칭되는 왼쪽 | 오른쪽 기준 |

### 2-2. inner / outer 비교 (주가 + 재무)
```python
df_주가  = pd.read_csv('07_stock_merge_주가.csv', index_col=0)     # 10종목
df_재무  = pd.read_csv('07_stock_merge_재무정보.csv', index_col=0)  # 10종목

# inner: 양쪽에 다 있는 종목만
pd.merge(df_주가, df_재무, on='종목코드', how='inner').shape   # (9, 9)  공통 9종목

# outer: 한쪽에만 있어도 모두 포함(빈칸은 NaN)
pd.merge(df_주가, df_재무, on='종목코드', how='outer').shape   # (11, 9) 합집합 11종목
```
> 📊 같은 두 표라도 `how`에 따라 결과 행 수가 달라집니다(여기선 9 vs 11). **"누구를 기준으로 남길지"** 가 분석 의도를 좌우하므로 조인 방식 선택이 핵심입니다.

---

## 3. 실전 — 집계표를 회원정보와 결합

"회원별 구매 합계"를 구해 회원 정보(성별·생년)와 붙이는, 실무에서 가장 흔한 패턴입니다.

```python
df_member  = pd.read_csv('04_store_member.csv')    # 회원 4,396명 (회원번호·성별·생년…)
df_product = pd.read_csv('04_store_product_1.csv')  # 구매내역 130,893건
```

**① 구매내역을 회원번호로 집계**(Part 6 groupby)
```python
구매합 = df_product.groupby('회원번호')[['구매금액', '구매수량']].sum()   # 1,383명
```

**② 회원 정보와 merge**
```python
df_inner = pd.merge(구매합, df_member, on='회원번호', how='inner')   # (1383, 8) 구매한 회원만
df_right = pd.merge(구매합, df_member, on='회원번호', how='right')   # (4396, 8) 전체 회원(미구매는 NaN)
```
> 구매가 있는 회원만 보려면 `inner`, 구매가 없는 회원까지 포함해 "미구매 회원"을 분석하려면 `right`(전체 회원 기준).

**③ 결합 후 분석**(Part 6 응용)
```python
df_inner['연령'] = 2024 - df_inner['생년']
df_inner.groupby('성별')[['구매금액', '구매수량']].sum()
```
```
      구매금액        구매수량
성별
남     36626115     7401.65
여    809091939   144515.65
```
> 결합 덕분에 "성별 구매액" 같은, **원래 한 표만으로는 불가능했던 분석**이 가능해집니다.

---

## 4. concat vs merge — 한눈에

| 구분 | `concat` | `merge` |
|---|---|---|
| 기준 | **위치**(행/열로 쌓기) | **공통 키(값)** 로 짝짓기 |
| 입력 | 리스트 `[df1, df2, …]` | 두 개 `left, right` |
| 핵심 인수 | `axis`, `ignore_index` | `on`, `how` |
| 쓸 때 | 같은 구조 데이터 합산 | 서로 다른 정보 연결 |

> 💡 한 문장 요약: **"같은 모양이면 `concat`, 다른 정보를 키로 엮으면 `merge`."**

---

## 5. 데이터 분석 프로세스에서의 활용

```python
# ① 흩어진 기간 데이터 통합 (concat)
전체 = pd.concat([q1, q2, q3, q4], axis=0, ignore_index=True)

# ② 집계 (groupby)
회원별 = df_product.groupby('회원번호')[['구매금액','구매수량']].sum()

# ③ 다른 정보와 결합 (merge)
분석표 = pd.merge(회원별, df_member, on='회원번호', how='inner')

# ④ 결합된 표로 새로운 분석
분석표.groupby('성별')['구매금액'].sum()
```

흩어진 데이터를 **통합(concat) → 집계(groupby) → 연결(merge)** 하면, 단일 표로는 보이지 않던 관계(성별·연령별 구매 패턴 등)가 드러납니다.

---

## 6. 자주 하는 실수 체크리스트

- [ ] `concat`의 입력은 **리스트** `[df1, df2]` 로 감쌌는가
- [ ] 행 결합은 `axis=0`, 열 결합은 `axis=1` 로 맞췄는가
- [ ] 행 결합 후 인덱스가 겹치면 `ignore_index=True` 를 줬는가
- [ ] `axis=1` 결합 시 인덱스(행 순서)가 맞는지 확인했는가
- [ ] `merge`의 `on` 키가 양쪽에 같은 이름으로 존재하는가
- [ ] 결과 행 수가 의도와 맞는지(`inner`로 데이터가 너무 줄지 않았는지) 확인했는가

---

## 7. 핵심 요약

1. `concat([df1, df2], axis=0/1)` — 같은 구조 데이터를 **행/열로 쌓기**. `ignore_index`로 인덱스 정리.
2. `for` + `concat` 으로 여러 파일을 반복 결합한다.
3. `merge(left, right, on='키', how=...)` — **공통 키로 표를 엮기**.
4. `how`: `inner`(교집합), `outer`(합집합), `left`/`right`(한쪽 전체 기준). 결과 행 수가 달라진다.
5. **같은 모양이면 concat, 다른 정보를 키로 엮으면 merge.**
6. groupby 집계표를 merge로 마스터 정보와 연결하는 패턴이 실무의 핵심.

### 미니 퀴즈
1. 1~4분기 파일을 위아래로 합치고 인덱스를 새로 매기려면? (답: `pd.concat([q1,q2,q3,q4], axis=0, ignore_index=True)`)
2. 두 표를 종목코드 공통값만 남겨 합치는 코드는? (답: `pd.merge(a, b, on='종목코드', how='inner')`)
3. 회원 전체를 남기되 구매 정보를 붙이려면 `how`는? (답: 회원 표가 오른쪽이면 `'right'`, 왼쪽이면 `'left'` — 회원 표 기준 전체 유지)

---

> **다음 파트 → Part 8 · 피벗테이블(pivot_table)**
> 엑셀 피벗처럼 기준별로 데이터를 요약·집계합니다. index·columns·values 배치, `aggfunc` 집계 방식, `margins` 총합을 배웁니다.

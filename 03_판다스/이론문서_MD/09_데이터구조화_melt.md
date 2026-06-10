# Part 9 · 데이터 구조화 — melt로 Wide를 Long으로

> **이 파트의 목표**
> - 데이터의 두 가지 모양 **넓은 형식(Wide)** 과 **긴 형식(Long)** 의 차이와 쓰임을 이해한다.
> - 여러 열에 흩어진 값을 한 열로 모으는 **`melt`** 의 문법과 인수를 익힌다.
> - `str.split`으로 합쳐진 열을 다시 나누어 분석하기 좋은 구조를 완성한다.
>
> **선행 지식**: Part 8(`pivot_table`), Part 6(`str.split`, `drop`).

예제는 분기별 제품 판매 데이터(`pandas_09_melt.csv`, `pandas_09_melt2.csv`)를 사용합니다.

```python
import pandas as pd
df = pd.read_csv('pandas_09_melt.csv', index_col=0)
```

---

## 0. 들어가며 — 데이터에도 "좋은 모양"이 있다

같은 데이터라도 모양(구조)에 따라 분석 난이도가 완전히 달라집니다. 두 가지 형식이 있습니다.

**넓은 형식(Wide Format)** — 분기마다 열이 따로
```
       제품     Q1_판매량  Q2_판매량  Q3_판매량
삼성 갤럭시 S21     100      110      130
```
사람 눈에는 보기 좋지만, **"분기별 판매량 평균"** 같은 집계를 하려면 열을 일일이 다뤄야 해서 불편합니다.

**긴 형식(Long Format)** — 분기를 값(행)으로
```
       제품      분기    판매량
삼성 갤럭시 S21     Q1     100
삼성 갤럭시 S21     Q2     110
삼성 갤럭시 S21     Q3     130
```
열이 적고 행으로 쌓여 있어, `groupby`·`pivot_table`·시각화 도구가 **곧바로** 처리할 수 있습니다.

> 💡 대부분의 분석 도구(특히 시각화 라이브러리 seaborn)는 **Long Format**을 좋아합니다. "분석에 적합한 구조 = Long"으로 기억하세요.

> 📊 **분석 프로세스상 위치**: `[Part 8] 피벗(Long→Wide 요약) ↔ [Part 9] melt(Wide→Long 변환)`. melt는 피벗의 **반대 방향**입니다.

---

## 1. Wide 형식의 문제

```python
df      # pandas_09_melt.csv
```
```
           제품  Q1_판매량  Q2_판매량  Q3_판매량
0  삼성 갤럭시 S21     100     110     130
1  LG OLED TV     150     160     180
2   애플 아이폰 12     200     210     230
```

문제점:

- **같은 변수(판매량)가 여러 열로 쪼개져** 있음 (Q1/Q2/Q3).
- 분기별로 묶어 집계하거나 시각화하기 어려움.
- 분기가 늘어날 때마다 열이 계속 추가됨.

이걸 한 열("분기")로 모아야 다루기 쉬워집니다. 그 변환이 `melt`입니다.

---

## 2. `melt()` — 여러 열을 한 열로 녹이기

**개념**: 가로로 펼쳐진 여러 열을 **"이름 열 + 값 열"** 한 쌍으로 녹여(melt) 세로로 쌓습니다.

**원리**: 유지할 기준 열(`id_vars`)은 그대로 두고, 나머지 열들의 **열 이름은 한 열로**, **값은 다른 한 열로** 옮깁니다. 그래서 열이 줄고 행이 늘어납니다.

**문법 / 인수**
```python
pd.melt(frame, id_vars=None, value_vars=None, var_name=None, value_name='value')
```

| 인수 | 의미 | 기본값 |
|---|---|---|
| `frame` | 대상 데이터프레임 | (필수) |
| `id_vars` | **유지**할 기준 열(고정) | None |
| `value_vars` | 녹일 열(지정 안 하면 id_vars 외 전부) | None |
| `var_name` | 열 이름이 담길 새 열의 이름 | `'variable'` |
| `value_name` | 값이 담길 새 열의 이름 | `'value'` |

**사용법**
```python
df_long = pd.melt(df, id_vars='제품', var_name='분기', value_name='판매량')
df_long
```
```
           제품      분기  판매량
0  삼성 갤럭시 S21  Q1_판매량  100
1  LG OLED TV  Q1_판매량  150
2   애플 아이폰 12  Q1_판매량  200
3  삼성 갤럭시 S21  Q2_판매량  110
...                          (총 9행)
```
> 3행×4열(Wide)이 **9행×3열(Long)** 로 바뀌었습니다. 제품은 유지되고, Q1/Q2/Q3 열이 '분기' 한 열로 녹았습니다.

> ⚠️ `id_vars`를 빼먹으면 제품까지 녹아버립니다. **기준으로 남길 열은 꼭 `id_vars`로 지정**하세요.

---

## 3. 합쳐진 열 분리 — `str.split(expand=True)`

melt 후 `분기` 값이 `'Q1_판매량'`처럼 **두 정보가 붙어** 있을 때가 많습니다. 이를 다시 쪼갭니다.

예제 데이터 `melt2`는 판매량·매출액이 섞여 있습니다.
```python
df = pd.read_csv('pandas_09_melt2.csv', index_col=0)   # Q1_판매량, Q1_매출액, Q2_판매량, Q2_매출액

df_long = pd.melt(df, id_vars='제품', var_name='분기_항목', value_name='값')
```

**`str.split('_', expand=True)`** — 구분자로 쪼개 **여러 열로** 펼치기
```python
df_long[['분기', '항목']] = df_long['분기_항목'].str.split('_', expand=True)
df_long = df_long.drop(columns=['분기_항목'])      # 원래 합쳐진 열 제거
df_long.head(6)
```
```
           제품        값  분기   항목
0  삼성 갤럭시 S21      100  Q1  판매량
1  LG OLED TV      150  Q1  판매량
2   애플 아이폰 12      200  Q1  판매량
3  삼성 갤럭시 S21  1000000  Q1  매출액
...                              (총 12행)
```
> 💡 `expand=True`가 핵심입니다. 이게 있으면 쪼갠 조각이 **여러 열**이 되어 `[['분기','항목']]`에 나눠 담깁니다. 없으면 리스트 하나로 묶여버립니다.

---

## 4. `value_vars` — 일부 열만 변환

**개념**: 모든 열이 아니라 **특정 열만** 녹이고 싶을 때 지정합니다.

```python
pd.melt(df, id_vars='제품',
        value_vars=['Q1_판매량', 'Q2_판매량'],   # 이 두 열만 녹임
        var_name='분기', value_name='판매량')
```
> 매출액 열은 두고 판매량 열만 Long으로 바꿉니다.

---

## 5. 다중 `id_vars` — 여러 기준 유지

기준이 둘 이상이면 리스트로 줍니다.
```python
pd.melt(df, id_vars=['제품', '지역'],            # 제품·지역 둘 다 유지
        value_vars=['Q1_판매량', 'Q2_판매량', 'Q1_매출액', 'Q2_매출액'],
        var_name='분기_항목', value_name='값')
```
> 제품·지역을 고정한 채 나머지를 녹입니다. 이후 `str.split`으로 분기/항목을 분리하면 완전한 Long 구조가 됩니다.

---

## 6. melt ↔ pivot_table — 서로 반대

| 방향 | 함수 | 변화 |
|---|---|---|
| Wide → Long | **`melt`** | 열 → 행 (녹이기) |
| Long → Wide | **`pivot_table`**(Part 8) | 행 → 열 (펼치기) |

> 💡 데이터를 받았는데 분석이 안 되면 `melt`로 **Long으로 녹였다가**, 보고서를 만들 때 `pivot_table`로 **Wide로 펼치는** 식으로 둘을 오갑니다. 이 둘은 데이터 구조 변환의 양 날개입니다.

---

## 7. 데이터 분석 프로세스에서의 활용 (전처리 종합)

실무에서는 지저분한 엑셀(Wide)을 받아 Long으로 정리한 뒤 분석합니다.

```python
# ① 엑셀 불러오기 (머리글이 둘째 줄이면 skiprows로 건너뜀)
df = pd.read_excel('05_Stack.xlsx', skiprows=1)

# ② 불필요한 열 제거 (Part 1)
df = df.drop(columns=['구분', '카테고리명', '단위'])

# ③ Wide → Long 변환 (melt)
long = pd.melt(df, id_vars=['제품명'], var_name='월', value_name='매출액')

# ④ 이제 집계·시각화가 쉬움 (Part 6·8)
long.groupby('제품명')['매출액'].sum()
```

**불러오기 → 정리 → melt로 구조화 → 집계**. 데이터 구조를 바로잡는 것만으로 이후 모든 분석이 쉬워집니다. 이것이 전처리의 마지막 퍼즐입니다.

---

## 8. 자주 하는 실수 체크리스트

- [ ] 유지할 기준 열을 `id_vars`로 지정했는가(안 하면 기준까지 녹음)
- [ ] `var_name`(이름 열)과 `value_name`(값 열)을 의미에 맞게 지정했는가
- [ ] 합쳐진 열을 나눌 때 `str.split('_', expand=True)`의 **`expand=True`** 를 줬는가
- [ ] split 후 원래의 합쳐진 열을 `drop`으로 제거했는가
- [ ] 일부만 변환할 땐 `value_vars`로 대상 열을 한정했는가
- [ ] melt(Wide→Long)와 pivot_table(Long→Wide)의 방향을 혼동하지 않았는가

---

## 9. 핵심 요약

1. 데이터는 **Wide(열이 많음)** 와 **Long(행으로 쌓임)** 두 모양이 있고, 분석엔 보통 **Long**이 유리하다.
2. `melt(id_vars=, value_vars=, var_name=, value_name=)` 로 Wide를 Long으로 녹인다.
3. `id_vars`=유지할 기준, 나머지는 **이름 열 + 값 열** 한 쌍으로 변환된다.
4. 합쳐진 열은 `str.split('구분자', expand=True)` 로 여러 열로 분리한다.
5. `melt`(녹이기)와 `pivot_table`(펼치기)은 **서로 반대 방향**의 구조 변환이다.
6. 지저분한 Wide 데이터를 Long으로 정리하면 집계·시각화가 쉬워진다.

### 미니 퀴즈
1. 제품을 유지하고 분기 열들을 녹이는 코드는? (답: `pd.melt(df, id_vars='제품', var_name='분기', value_name='판매량')`)
2. `'Q1_판매량'`을 '분기'와 '항목'으로 나누려면? (답: `df['열'].str.split('_', expand=True)`)
3. melt와 반대로 Long을 Wide로 바꾸는 함수는? (답: `pivot_table`)

---

> **다음 파트 → Part 10 · Pandas Profiling (보너스)**
> 지금까지 손으로 한 데이터 탐색을, 한 줄로 자동 리포트화하는 YData Profiling을 살펴보며 과정을 마무리합니다.

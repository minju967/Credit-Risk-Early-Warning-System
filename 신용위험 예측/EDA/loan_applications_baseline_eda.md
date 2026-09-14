# Loan Applications Baseline EDA

## 1. 분석 목적

대출 신청 데이터를 기반으로 타깃 변수의 정의와 분석 모집단을 확인하고, 기본 신청 정보만으로 향후 12개월 이내 부도·연체 위험을 설명할 수 있는지 탐색한다.

### 기존 프로젝트 목표

> 대출 신청자의 향후 12개월 이내 부도·연체 위험을 예측한다.

### 수정된 프로젝트 목표

> **대출 승인 고객을 대상으로, 대출 신청 시점까지 관측 가능한 금융·신용 정보를 활용하여 승인 후 12개월 이내 부도·연체 발생 위험을 예측한다.**

---

## 2. 타깃 변수 및 분석 모집단 정의

### 2.1 `default_12m_label` 결측치 확인

`default_12m_label`의 결측치는 특정 신청 시점에 집중되지 않고, 월별 약 23~25% 수준으로 비교적 고르게 분포한다. 따라서 최근 신청 건의 타깃 관찰 기간(Target Window)이 아직 끝나지 않아 발생한 결측이라고 보기는 어렵다.

대출 승인 여부인 `approved`와의 관계를 확인한 결과, 결측치는 대출이 승인되지 않은 신청 건에서 발생했다. 즉, `default_12m_label`은 **승인된 대출 신청 건에 대해서만 정의된 타깃 변수**로 판단된다.

### 2.2 분석 모집단

- 조건: `approved = 1`
- 분석 단위: 승인된 대출 신청 건
- 데이터셋: `model_df`
- 관측치 수: **137,094건**

---

## 3. 타깃 분포

| Target | 의미 | 건수 | 비율 |
|---:|---|---:|---:|
| `0` | 정상 | 131,539 | 95.95% |
| `1` | 승인 후 12개월 이내 부도·연체 발생 | 5,555 | 4.05% |
| **합계** |  | **137,094** | **100.00%** |

전체 승인 신청 건 중 평균 **4.05%**에서 승인 후 12개월 이내 부도 또는 연체가 발생했다. 정상과 부도·연체 클래스의 비율 차이가 크므로, 모델링 단계에서 **클래스 불균형**을 고려해야 한다.

### 3.1 월별 Default Rate

월별 Default Rate는 약 3~5% 범위에서 변동했으며, 특정 기간에 집중된 뚜렷한 변화는 관찰되지 않았다. 따라서 타깃 비율은 시간에 따라 비교적 안정적으로 유지되는 것으로 판단된다.

![Monthly Loan Default Ratio](/신용위험%20예측/images/monthly_loan_default_ratio.png)


| 통계량 | Default Rate (%) |
|---|---:|
| 평균 | 4.05 |
| 최솟값 | 2.98 |
| 최댓값 | 5.00 |
| 표준편차 | 0.40 |

---

## 4. Baseline 변수 선정 및 데이터 품질 확인

### 4.1 Baseline Features

```python
baseline_features = [
    "requested_amount",
    "annual_income",
    "employment_years",
    "debt_to_income",
    "bureau_score",
    "internal_css_score",
    "product_type",
    "loan_purpose",
]
```

선정한 Baseline 변수에서는 결측치가 확인되지 않았다.

### 4.2 변수 구성

| 구분 | 변수 |
|---|---|
| 수치형 | `requested_amount`, `annual_income`, `employment_years`, `debt_to_income`, `bureau_score`, `internal_css_score` |
| 범주형 | `product_type`, `loan_purpose` |

`product_type`의 범주는 다음과 같다.

| 값 | 의미 |
|---|---|
| `AUTO_LOAN` | 자동차 대출 |
| `CREDIT_LINE` | 한도 대출 |
| `MORTGAGE` | 주택담보대출 |
| `PERSONAL_LOAN` | 개인 신용대출 |
| `SME_LOAN` | 중소기업 대출 |

---

## 5. 변수별 분석 결과

### 5.1 신청금액: `requested_amount`

`requested_amount`는 평균이 중앙값보다 크고 최댓값이 약 **17.2억 원**으로, 오른쪽 꼬리가 긴 분포를 보인다.
![Distribution of Requested Amount](/신용위험%20예측/images/distribution_of_requested_amount.png)
![Distribution of Requested Amount (≤ 99th Percentile)](/신용위험%20예측/images/requested_amount_distribution_99pct.png)
![Default Rate by Requested Amount Quantile and Product Type](/신용위험%20예측/images/default_rate_by_product_amount.png
)


신청금액 상위 1%의 상품 구성을 확인한 결과, `MORTGAGE`가 94.82%, `SME_LOAN`이 5.18%를 차지했다. 상품별 신청금액을 비교해도 `MORTGAGE`와 `SME_LOAN`의 금액 수준이 상대적으로 높았다.

따라서 신청금액의 극단값은 데이터 오류라기보다 **대출 상품별 금액 규모 차이**에서 발생한 것으로 판단되며, 현 단계에서는 이상치로 제거하지 않는다.

상품별 신청금액 분위에 따른 Default Rate를 비교한 결과, 신청금액과 부도·연체 위험의 관계는 상품에 따라 다르게 나타났다.

- `MORTGAGE`: Q1 6.31%에서 Q5 16.18%로 신청금액이 증가할수록 Default Rate가 뚜렷하게 상승했다.
- `SME_LOAN`, `AUTO_LOAN`: 신청금액 증가에 따라 Default Rate가 상승하는 유사한 경향이 관찰됐다.
- `CREDIT_LINE`, `PERSONAL_LOAN`: 신청금액에 따른 뚜렷한 위험 차이가 확인되지 않았다.

따라서 `requested_amount`의 영향은 단독 효과뿐 아니라 **`product_type`과의 상호작용**을 함께 고려할 필요가 있다.

### 5.2 부채상환부담: `debt_to_income`

`debt_to_income`의 최댓값은 1.5이며, 전체 분석 모집단 중 2,646건(약 2%)이 정확히 1.5의 값을 갖는다. 또한 99% 분위수부터 최댓값까지 모두 1.5로 나타나, 해당 변수가 1.5를 상한으로 제한한 값일 가능성이 있다.

데이터 정의서가 없어 상한 처리 여부를 확정할 수 없으므로 현 단계에서는 제거하지 않고 유지한다. 다만 모델 해석 시 **1.5가 실제 관측값인지 상한 처리된 값인지 확인이 필요하다.**

`debt_to_income`이 증가할수록 Default Rate가 전반적으로 상승했다. 따라서 이 변수는 부도·연체 위험을 구분하는 주요 Baseline Feature 후보로 판단된다.

### 5.3 신용점수: `bureau_score`, `internal_css_score`

외부 신용점수인 `bureau_score`와 내부 신용평가점수인 `internal_css_score` 모두 점수가 낮을수록 Default Rate가 높아지는 경향이 확인됐다.

특히 `internal_css_score`는 점수 구간에 따른 Default Rate 차이가 크게 나타나, 향후 부도·연체 위험을 구분하는 데 강한 설명력을 가질 가능성이 있다.

다만 `internal_css_score`가 타깃과 강하게 연결되어 있으므로 **점수 산출 시점과 입력 정보에 대한 확인이 필요하다.** 승인 결과 또는 대출 신청 이후에 발생한 정보를 반영해 산출된 값이라면 Data Leakage가 발생할 수 있다.

![Distribution of internal_css_score](/신용위험%20예측/images/distribution_of_internal_css_score.png)
![Distribution of Bureau Score](/신용위험%20예측/images/distribution_of_internal_Bureau_score.png)

### 5.4 연소득: `annual_income`

연소득이 증가할수록 향후 12개월 Default Rate가 대체로 감소하는 경향이 확인됐다.

- 연소득 5천만 원 이하: Default Rate 5.01%로 가장 높게 나타남
- 연소득 1억 원 이상: Default Rate가 약 2%대로 감소

다만 고소득 구간은 표본 수가 적어 Default Rate의 변동성이 크므로 해석에 주의가 필요하다.

### 5.5 근속기간: `employment_years`

근속기간이 길어질수록 Default Rate가 점진적으로 감소하는 경향이 확인됐다. 다만 구간 간 차이는 신용점수나 DTI에 비해 크지 않아 상대적으로 약한 관계를 보인다.

20년 이상 구간은 표본 수가 매우 적으므로, 해당 구간의 Default Rate 0%를 의미 있는 결과로 해석하지 않는다.

### 5.6 소득 대비 신청금액: `amount_income_ratio`

고객의 상환 능력 대비 대출 신청 규모를 나타내기 위해 다음 파생변수를 생성했다.

```text
amount_income_ratio = requested_amount / annual_income
```

비율이 증가할수록 Default Rate가 점진적으로 상승하는 패턴은 나타나지 않았다. 다만 소득 대비 신청금액이 가장 높은 상위 20% 구간(Q5)에서 Default Rate가 **6.89%**로 크게 상승했다.

따라서 소득 대비 과도한 대출 신청은 향후 부도·연체 위험 증가와 관련이 있을 가능성이 있다.

### 5.7 대출 목적: `loan_purpose`

대출 목적별 Default Rate는 3.78~4.28% 수준으로, 범주 간 차이가 크지 않았다. 따라서 `loan_purpose`가 단독으로 부도·연체 위험을 구분하는 설명력은 상대적으로 낮을 것으로 판단된다.

다만 다른 변수와의 상호작용 가능성이 있으므로 모델링 후보에서는 제외하지 않는다.

---

## 6. 상관관계 분석

| 변수 관계 | 상관계수 | 해석 |
|---|---:|---|
| `requested_amount` ↔ `debt_to_income` ↔ `amount_income_ratio` | 0.70~0.81 | 높은 양의 상관관계로, 일부 중복 정보를 포함할 가능성이 있음 |
| `bureau_score` ↔ `internal_css_score` | 0.20 | 상관관계가 낮아 서로 다른 신용위험 정보를 포함할 가능성이 있음 |
| `annual_income` ↔ `bureau_score` | 0.42 | 중간 수준의 양의 상관관계 |
| `annual_income` ↔ `debt_to_income` | -0.23 | 약한 음의 상관관계 |

`requested_amount`, `debt_to_income`, `amount_income_ratio`는 서로 높은 상관관계를 보이므로 모델링 단계에서 다중공선성과 변수 중요도의 분산 여부를 확인할 필요가 있다. 반면 외부 신용점수와 내부 CSS 점수는 상관관계가 낮아 상호 보완적인 정보를 제공할 가능성이 있다.

---

## 7. Baseline EDA 요약

| 관계 수준 | 변수 | 주요 해석 |
|---|---|---|
| 강한 관계 | `internal_css_score`, `debt_to_income` | 위험군 구분력이 높을 가능성이 있음 |
| 명확한 관계 | `bureau_score` | 점수가 낮을수록 Default Rate가 상승 |
| 관계 있음 | `annual_income`, `employment_years` | 소득과 근속기간이 증가할수록 Default Rate가 대체로 감소 |
| 상품별 차이 | `requested_amount` | `product_type`과의 상호작용을 고려해야 함 |
| 단독 관계가 약함 | `loan_purpose` | 범주별 Default Rate 차이가 작음 |
| 파생변수 후보 | `amount_income_ratio` | 상위 20% 구간에서 Default Rate가 크게 상승 |

## 8. 모델링 전 확인 사항

1. **분석 모집단 유지**: `approved = 1`인 승인 신청 건만 모델링에 사용한다.
2. **Data Leakage 점검**: `internal_css_score`의 산출 시점과 입력 변수를 확인한다.
3. **DTI 상한 여부 확인**: `debt_to_income = 1.5`가 실제 값인지 상한 처리된 값인지 확인한다.
4. **클래스 불균형 대응**: Accuracy 외에 ROC-AUC, PR-AUC, Recall, Precision 등의 평가지표를 함께 사용한다.
5. **변수 간 중복 검토**: `requested_amount`, `debt_to_income`, `amount_income_ratio`의 높은 상관관계를 모델링 단계에서 점검한다.
6. **상호작용 검토**: `requested_amount`와 `product_type`의 상호작용 또는 상품별 패턴을 반영한다.

---

## 9. 결론

승인된 대출 신청 건을 대상으로 Baseline EDA를 수행한 결과, `internal_css_score`, `debt_to_income`, `bureau_score`가 향후 12개월 부도·연체 위험을 구분하는 주요 변수로 확인됐다. `annual_income`과 `employment_years`도 일정한 관계를 보였으며, `requested_amount`는 상품 유형에 따라 위험과의 관계가 달라 단독 해석보다 상호작용을 고려하는 것이 적절하다.

향후 모델링에서는 데이터 누수 가능성, 클래스 불균형, 변수 간 중복 정보 및 상품별 차이를 함께 검토해야 한다.

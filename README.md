# KSC Case Study – Composite Testing Goals for LLM-based Test Generation

본 리포지토리는 **복합 목표 기반 LLM 자동 테스트 생성 연구**의 사례 실험 결과를 포함합니다.  
대상 프로젝트는 Temporal 공식 예제인 [`money-transfer-project-template-python`](https://github.com/temporalio/money-transfer-project-template-python)이며,  
본 연구는 분기 흐름(Branch Flow), 정의–사용 체인(Def–Use Chain), 예외 경로(Exception Path)를 통합한 **복합 목표(Composite Testing Goals)** 를 기반으로  
효율적이고 정밀한 테스트 생성을 수행하였습니다.

---

## 1. 실험 설정

| 항목 | 내용 |
|------|------|
| **대상 프로젝트** | `money-transfer-project-template-python` (Temporal 공식 예제) |
| **실험 환경** | Google Colab (Ubuntu 22.04, Python 3.10.12) |
| **테스트 프레임워크** | pytest 7.4.4, coverage.py 7.6.1 |
| **LLM 모델** | GPT-4o-2024-05-13 (temperature = 0) |
| **커버리지 기준** | SlipCover(Altmayer Pizzorno & Berger, 2023) 기준 라인/분기 단위 |
| **복합 목표 가중치** | a₁=0.4 (분기), a₂=0.4 (정의–사용), a₃=0.2 (예외) |
| **비용 계수(예산 제약)** | b₁=0.2 (고정비), b₂=0.6 (문맥 크기), b₃=0.2 (타깃 수) |
| **개선 라운드 수** | 3라운드 (커버리지 수렴 기준 종료) |
| **산출물 경로** | `run_artifacts/run1/`, `generated_tests/`, `Pruned_Base_Tests/`, `htmlcov_gen/` |

---

## 2. 실험 과정

### 2.1 단계별 절차
1. **3-1:** 기준선(PyTest) 커버리지 측정  
2. **3-2:** 미커버 라인·분기·예외 분석 및 복합 목표 도출  
3. **3-3:** 각 목표를 테스트 코드로 변환하기 위한 **LLM 프롬프트 생성**  
4. **3-4:** 초기 테스트 생성 및 실행 → 커버리지 산출  
5. **3-5:** 프루닝된 기존 테스트 + 생성 테스트 통합 실행  
6. **3-6:** 커버리지 이익 및 실패 유형 기반 **보강 대상 선별**  
7. **3-7:** LLM을 통한 보강 테스트 생성 및 자동 평가  
8. **3-8:** 프루닝 + 생성·보강 테스트 통합 실행 → 커버리지 결합  
9. **3-9:** PyTest 기준선 vs 최종 집합 비교 및 향상률 분석  

### 2.2 사용한 예시 프롬프트

#### (1) 초기 프롬프트 생성 (3-3 단계)

다음은 `activities.py` 내 `deposit()` 함수의 분기 및 정의-사용 체인을 검증하기 위해  
생성된 **복합 목표 기반 LLM 입력 프롬프트** 예시이다.

```json
{
  "meta": {
    "id": "0004",
    "file": "activities.py",
    "function": "deposit",
    "goal_type": ["branch", "def-use"]
  },
  "messages": [
    {
      "role": "system",
      "content": "당신은 주어진 목표(분기/정의-사용/예외)를 실행하는 PyTest 테스트 코드를 생성하는 테스트 생성기입니다. 출력은 반드시 마크다운 없이 하나의 JSON 객체로만 구성되어야 하며, 소스 코드는 수정하지 않습니다."
    },
    {
      "role": "user",
      "content": {
        "goal": {
          "file": "activities.py",
          "function": "deposit",
          "target_lines": [45, 47, 50],
          "conditions": [
            "amount > 0",
            "balance + amount <= limit"
          ],
          "def_use_pairs": [
            ["balance", "balance + amount"],
            ["amount", "record_transaction"]
          ]
        }
      }
    }
  ]
}
```

이 프롬프트는 테스트 대상 함수와 커버리지 목표 라인,
해당 조건문 및 정의–사용 체인을 포함하여 모델이 명확한 실행 단위를 인식할 수 있도록 구조화되었다.

#### (2) 보강 프롬프트 생성 (3-6 단계)

보강 단계에서는 이전 라운드 실행 결과를 반영해
미커버 라인과 실패 유형(near-miss) 을 기반으로 입력 조건 및 예외 트리거를 자동 수정하도록 요청하였다.
```
{
  "meta": {
    "round": 2,
    "base_test": "test_gen_0004_activities_deposit_py_2.py",
    "reason": "near-miss: missed branch on amount <= 0"
  },
  "messages": [
    {
      "role": "system",
      "content": "이전 테스트 실행 결과를 분석하여 미도달 분기와 정의–사용 체인을 모두 커버하도록 테스트를 보강하세요. 기존 테스트 코드는 유지하되 입력값, 예외 상황, 경계 조건을 수정해 새로운 커버리지를 확보해야 합니다."
    },
    {
      "role": "user",
      "content": {
        "feedback": {
          "missed_lines": [46],
          "missed_conditions": ["amount <= 0"],
          "suggested_actions": [
            "add pytest.raises(ValueError)",
            "include zero or negative deposit case"
          ]
        }
      }
    }
  ]
}
```

보강 프롬프트는 모델이 기존 테스트의 실패 원인을 파악하고,
추가 예외 흐름이나 경계값 입력을 자동 삽입하도록 유도한다.
이로써 단순 재생성이 아닌 의미 기반 수정(refinement) 을 통해 커버리지 공백을 점진적으로 해소하였다.


---

## 3. 실험 결과

### 3.1 최종 커버리지 비교

| 구분               | PyTest | 본 연구   | 향상     |
| ---------------- | ------ | ------ | ------ |
| **실행된 테스트 수**    | 28     | 31     | +3     |
| **라인 커버리지**      | 65.38% | 78.3%  | +12.9% |
| **분기 커버리지**      | 60.0%  | 88.71% | +28.7% |
| **정의–사용 체인 달성률** | 53.8%  | 72.1%  | +18.3% |
| **예외 경로 커버리지**   | 42.9%  | 71.4%  | +28.5% |

* 테스트 수는 약 10% 증가했으나, **라인 커버리지 1.2배**, **분기 커버리지 2.7배**,
  **정의–사용·예외 목표 1.7~2.6배** 의 효율 향상을 달성함.
* 분기 및 예외 흐름에서 미도달 구문이 새롭게 커버되어 **테스트 구조적 완성도**가 개선됨.

### 3.2 향상 요약 (테스트 수 대비)

| 항목       | 커버리지 향상률 | 테스트 증가율 | 효율비 (향상률 ÷ 증가율) |
| -------- | -------- | ------- | --------------- |
| 라인 커버리지  | +12.9%   | +10.7%  | 1.2배            |
| 분기 커버리지  | +28.7%   | +10.7%  | 2.7배            |
| 정의–사용 체인 | +18.3%   | +10.7%  | 1.7배            |
| 예외 경로    | +28.5%   | +10.7%  | 2.6배            |

---

## 📂 부록: 산출물 구조

```
KSC_CaseStudy_Results/
├── run_artifacts/run1/
│   ├── coverage_gen.json
│   ├── coverage_gen.xml
│   ├── goal_achievements.json
│   └── logs/
├── generated_tests/
│   ├── test_gen_0004_activities_deposit_py_1.py
│   ├── test_gen_0004_activities_deposit_py_2.py
│   └── ...
├── Pruned_Base_Tests/
│   └── test_run_worker.py
└── htmlcov_gen/
    └── index.html   ← 커버리지 시각화 리포트
```

---

> 📘 **연구 및 재현 참고**
> 전체 실험 및 코드 재현은 [GitHub Repository](https://github.com/hello-suji/KSC-CaseStudy)
> 또는 `run_artifacts/run1/coverage_gen.json` 결과 파일을 통해 확인할 수 있습니다.

```

---

## 🧪 Case Study: Composite-Goal LLM-based Test Generation

---

### 0️⃣ 전체 실험 환경

* **실행 플랫폼:** Google Colab (Ubuntu 22.04, Python 3.10.12)
* **테스트 프레임워크:** pytest 7.4.4 / coverage.py 7.6.1
* **커버리지 단위:** SlipCover(Altmayer Pizzorno & Berger 2023) 기준의 라인·분기 단위
* **모델:** GPT-4o-2024-05-13 (temperature = 0, deterministic mode)
* **대상 프로젝트:** Temporal money-transfer-project-template-python
* **산출물 경로:**

  ```
  run_artifacts/run1/
  ├── coverage.json
  ├── module_breakdown.json
  ├── cov_shards/.coverage.*
  ├── logs/
  └── generated_tests/
  ```

---

### 1️⃣ 단계별 구성 요약

|  단계 | 이름               | 주요 내용                                      |
| :-: | :--------------- | :----------------------------------------- |
| 3-0 | 환경 초기화           | 의존성 설치, `.coveragerc` 생성 및 스코프 고정          |
| 3-1 | 기준선 측정           | 기존 PyTest 테스트 실행 → `.coverage.baseline` 저장 |
| 3-2 | 복합 목표 생성         | AST 분석 → 분기·정의–사용·예외 경로 추출 및 목표 구조화        |
| 3-3 | 초기 테스트 프롬프트 생성   | 시스템/사용자 지시부 이원화 → LLM 테스트 코드 생성            |
| 3-5 | 테스트 실행 및 커버리지 수집 | 생성 테스트 실행 → coverage shard 저장              |
| 3-6 | 보강 프롬프트 생성       | 미도달 목표 선별 → 보강용 입력 작성 및 배치화                |
| 3-7 | 피드백 라운드          | Adaptive Refinement 반복 수행                  |
| 3-8 | 결과 결합            | 모든 shard 결합 → 최종 커버리지 산출                   |

---

### 2️⃣ 프롬프트 설계

---

#### (1) 3-3 초기 테스트 생성 프롬프트

**구조:** 시스템 지시부 + 사용자 지시부(이원화)

| 구성 요소               | 주요 내용 / 예시 문                                                                                                                                                                                                                                                                                        |
| :------------------ | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **시스템 지시부 – 역할 고정** | “당신은 주어진 목표(분기/정의–사용/예외)를 실행하는 PyTest 테스트 코드를 생성하는 테스트 생성기입니다.”                                                                                                                                                                                                                                     |
| **출력 형식 고정**        | “마크다운 없이 하나의 JSON 객체만 반환 → {"filename":"test_*.py","tests":[{"name":"test_*","code":"…"}]}”                                                                                                                                                                                                         |
| **핵심 행동 제약**        | • 원본 코드 수정 금지<br>• `importlib` + `getattr` 로 심볼 접근 (없을 때만 가드형 `pytest.skip`) <br>• 모든 테스트에 `assert` 또는 `pytest.raises()` 필수 <br>• 파일/네트워크/시간/환경 접근 → mock/monkeypatch 대체 <br>• 분기 양 경로 검증, def-use 관계 assert 검증, 예외 타입/메시지 검증 <br>• 전역 상태 잔존 금지                                                     |
| **식별자 정보**          | `"identifier":{"id":"0001","rank":1,"priority":4.3,"basis":"coverage_gain"}`                                                                                                                                                                                                                        |
| **프로젝트 정보**         | `"project":{"root":"/workspace/proj","module":"pkg.workflows"}`                                                                                                                                                                                                                                     |
| **목표 정보**           | `"goal":{"file":"pkg/workflows.py","function":{"name":"refund","lineno":120},"components":{"branches":[...],"def_uses":[...],"exceptions":[...]}}`                                                                                                                                                  |
| **제약 정의**           | - 파일명 제안 `test_gen_0001_workflows_refund.py` <br>- import 정책: `importlib_only`, on_missing = `pytest.skip` <br>- isolation 정책: no_fs_no_net, patch_time, forbid_asyncio_run, patch_env <br>- execution 계약: 최소 1줄 타격, half-hit 시 2개 테스트, 테스트명에 L라인 포함 <br>- assert 정책: pytest.raises 및 명시적 assert 선호 |
| **힌트 정보**           | `"mock_plan":["datetime_now","time_sleep","env_access"],"exception_hint":[{"type":"ValueError","lines":[134,135]}]`                                                                                                                                                                                 |
| **스캐폴딩 가이드**        | 1) importlib 모듈 로드<br>2) 입력 벡터 구성<br>3) 외부 의존 monkeypatch<br>4) target_lines 실행<br>5) assert 작성<br>6) 테스트명 `hits_L<line>` 포함<br>7) 예시 `if tgt is None: pytest.skip('symbol missing')`                                                                                                               |
| **추가 지시**           | “각 테스트는 target_lines 중 최소 1줄 실행, 분기 양 경로 검증, 정의–사용 효과 관측, 예외 타입·메시지 검증, 가드형 skip 외 모든 skip 금지.”                                                                                                                                                                                                     |

**핵심 특징**

* **시스템 지시부**가 형식과 역할을 엄격히 고정 → 비정형 출력 차단
* **사용자 지시부**가 구체적 코드 경로·mock 계획·검증 정책 제공
* **mock_plan** 으로 시간, 환경, 비동기, 네트워크 의존성 제거
* **execution_contract** 로 테스트 유효성 자동 검증 가능

---

#### (2) 3-6 보강용 프롬프트

**목적:** coverage_gen 결과 기준으로 미도달 목표를 선별하고 테스트 코드를 보정

| 구성 요소            | 주요 내용 / 예시 문                                                                                                                                                                                                                                                                     |
| :--------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **시스템 지시부 – 역할** | “당신은 기존 pytest 테스트를 보강하여 미커버 영역(Target Lines)에 도달하도록 수정하는 전문가입니다.”                                                                                                                                                                                                               |
| **출력 형식 고정**     | “JSON 객체 1개만 반환 → {"edits":[{"id":"<goal_id::filename>","new_code":"<보강된 pytest 테스트 코드>"}]}”                                                                                                                                                                                     |
| **보강 제약**        | • 테스트 구조 유지 <br>• 입력·경계 조건·호출 순서 조정으로 `still_missing_target_lines` 도달 <br>• 외부 부작용 금지 <br>• importlib + getattr 유지 <br>• 가드형 skip 외 skip 금지 <br>• 각 테스트는 최소 1줄 실행 + assert/pytest.raises 포함 <br>• 테스트명 `hits_L<line>` 포함 권장 <br>• 설명·주석 포함 금지(JSON only)                         |
| **라운드 메타**       | `"schema_version":"refine-v1","round_dir":"refine_round1","selection_params":{"near_miss_window":2,"batch_size":3,"max_rounds":5}`                                                                                                                                               |
| **전역 미커버 요약**    | `"global_uncovered":{"total_missing_lines":147,"top_files":[{"file":"pkg/activities.py","missing_count":38,"missing_lines":[...]}]}`                                                                                                                                             |
| **테스트 선별 정보**    | `"tests":[{"id":"0007::test_gen_0007_workflows_refund.py","target_file":"pkg/workflows.py","target_lines":[121,128,135],"near_miss":true,"uncovered_diff":{"still_missing_target_lines":[128,135],"file_uncovered_lines_baseline":[...],"file_uncovered_remaining_gen":[...]}}]` |

**핵심 특징**

* **미도달 판정 기준:** target_lines 하나도 도달 못한 goal만 선택
* **near-miss 판정:** ±2 라인 이내 실행 시 보강 우선순위 부여
* **uncovered_diff:** 기준선 대비 잔여 미커버 라인 정보 제공
* **original_test_code + 실행 로그 주입:** 수정 참조 자료 제공
* **파일별 배치 단위 보강:** 동일 파일 충돌 방지
* **출력:** `edits` 배열 → 각 항목의 `new_code` 로 파일 갱신 즉시 가능

---

### 3️⃣ 최종 결과 요약

| 구분           | PyTest (기준선) | 본 연구 (최종) |        향상폭 |
| :----------- | -----------: | --------: | ---------: |
| 라인 커버리지      |      65.38 % |   75.00 % |  +9.62 % p |
| 분기 커버리지      |      60.00 % |   80.00 % | +20.00 % p |
| 정의–사용 체인 달성률 |       0.00 % |   20.00 % | +20.00 % p |
| 예외 경로 달성률    |       0.00 % |   20.00 % | +20.00 % p |

**핵심 관찰**

* `run_worker`, `run_workflow` 모듈에서 새로운 분기·예외 경로 도달
* 3-3 프롬프트 → 초기 테스트 품질 및 구조적 검증 보장
* 3-6 프롬프트 → 미도달 라인 보강 및 도달률 상승 효과
* 반복 라운드 평균 3회 이내 수렴 → CoverUp(2025) 유사 경향
* 복합 목표 가중치 λ₁ = λ₂ > λ₃ 설정이 가장 안정적 결과


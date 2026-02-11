## [2026/02/05] 00_EDA.ipynb

### 목적
- 전체 데이터의 대략적인 분포 확인
- 변수 타입 및 결측 패턴 파악
- 모델링에 활용 가능한 파생변수 후보 탐색

---

### 수행 내용
- 전체 컬럼 구조 및 타입 확인
- 주요 변수 분포 확인 (시술 유형, 시술 횟수, 임신 성공 여부 등)
- 결측치 패턴 분석
  - 배아 이식 관련 변수들은 특정 시술 단계에 도달한 경우에만 값이 존재함을 확인
  - 단순 결측치가 아닌 **시술 단계 차이에 따른 구조적 결측**으로 판단

---

### 파생변수 후보 정리

EDA 결과를 바탕으로, 다음과 같은 파생변수를 생성하는 것이 유의미하다고 판단함.

- **시술_대분류**
  - 세부 시술 유형을 IVF / ICSI / IUI / Other로 재분류
  - 샘플 수가 매우 적은 시술(FER, GIFT, DI, Unknown 등)은 Other로 통합

- **BLASTOCYST_포함**
  - 특정 시술 유형 문자열에 `BLASTOCYST` 포함 여부 (0/1)

- **AH_포함**
  - 특정 시술 유형 문자열에 `AH` 포함 여부 (0/1)

- **배아_이식_여부**
  - 배아 이식 단계까지 도달했는지 여부를 나타내는 이진 변수
  - 배아 관련 주요 변수들의 결측 패턴을 기반으로 생성

- **총시술_bin3**
  - 총 시술 횟수를 다음과 같이 구간화
    - 0회
    - 1–2회
    - 3회 이상
  - 시술 횟수 증가에 따른 임신 성공률 감소 경향 확인

---

## [2026/02/05] 01_EDA기반_LGBM_test.ipynb

### 사용 feature
- 파생변수만 사용 (원본 컬럼은 그대로 유지)
  - 시술_대분류
  - BLASTOCYST_포함
  - 배아_이식_여부
  - 총시술_bin3
  - 나이_3구간
  - 이식배아_구간
  - Day5_이식_여부
  - 불임_원인_개수
  - 불임원인_복잡도
  - 배아_해동_실시_여부

### 모델/셋팅
- 모델: LightGBM (GBDT)
- 검증: StratifiedKFold 5-fold, `random_state=42`
- 주요 하이퍼파라미터
  - `num_leaves=31`
  - `learning_rate=0.05`
  - `feature_fraction=0.8`
  - `bagging_fraction=0.8`, `bagging_freq=5`
  - `num_boost_round=1000`, early stopping 50

### 결과
- Local CV AUC: 0.739735
- LB: 0.74157

---

## [2026/02/05] 02_L&C&X_stacking.ipynb

### 사용 feature
- 파생변수 추가
  - 시술_대분류
  - BLASTOCYST_포함
  - 배아_이식_미도달
  - 배아_이식_여부
  - 배아_진행_단계
  - 총시술_bin3
  - 나이_3구간
  - 이식배아_구간
  - Day5_이식_여부
  - 불임_원인_개수
  - 불임원인_복잡도
  - 배아_해동_실시_여부
  - 배아_생성_효율
  - 배아_이식_비율
  - 배아_저장_비율
  - 나이×Day5
  - 시술횟수×나이

### 모델/셋팅
- 모델: LightGBM + CatBoost + XGBoost 스태킹
- 검증: StratifiedKFold 5-fold, `random_state=42`
- 메타모델: Logistic Regression

### 결과
- Local CV AUC: LGB 0.739818 / CAT 0.739827 / XGB 0.738716 / Stacking 0.740306
- LB: 미제출

---

## [2026/02/06] 03_autogluon.ipynb

### 사용 feature
- 원본 컬럼 + 기존 파생변수 포함 (AutoGluon 기본 전처리)

### 모델/셋팅
- 모델: AutoGluon Tabular (다중 모델 + WeightedEnsemble)
- 평가 지표: `roc_auc`

### 결과
- Local CV AUC: 0.7407
- LB: 0.742

---

## [2026/02/06] 04_5model_test.ipynb

### 사용 feature
- 02번과 동일한 파생변수 + 교차특성(나이×Day5, 시술횟수×나이)

### 모델/셋팅
- 모델: LightGBM / CatBoost / XGBoost / RandomForest / LogisticRegression
- 검증: StratifiedKFold 5-fold
- 앙상블: simple, weighted, rank, weighted_rank, stacking 전체 조합 테스트

### 결과
- Local CV AUC (Best): 0.740287 (LGB+CAT+XGB, stacking)
- LB: 미제출

---

## [2026/02/06] 05_lgbm&cat.ipynb

### 사용 feature
- 파생변수 추가
  - 시술_대분류
  - BLASTOCYST_포함
  - AH_포함
  - 배아_stage_missing_count
  - 배아_stage_all_missing (학습에서 제외)
  - 배아_이식_여부
  - 총시술_bin3
  - 나이_3구간
  - Day5_이식_여부
  - 불임_원인_개수
  - 전체_missing_count
  - 결측 flag: `PGD 시술 여부_isna`, `PGS 시술 여부_isna`, `배아 이식 경과일_isna`, `난자 채취 경과일_isna`, `배아 해동 경과일_isna`
  - 비율 변수: `이식/생성`, `저장/생성`, `해동/저장`, `미세주입_배아/난자`, `미세주입_이식/이식`

### 모델/셋팅
- 모델: CatBoost + LightGBM 앙상블
- 검증: StratifiedKFold 5-fold, `random_state=42` (OOF-safe)
- 불균형 대응: class weight / `scale_pos_weight`

### 결과
- Local CV AUC: CatBoost 0.740285 / LightGBM 0.739259 / Ensemble 0.740363

---

## [2026/02/07] 06_ML&DL.ipynb

### 사용 feature
- 05_lgbm&cat 파생 + 고결측 5개 제거
- 추가 파생: `배아_품질_점수`, `최적_이식_조건`, `고령_여부`, `IVF_과거_성공률`, `배아_선별률`, `확정_실패_케이스`

### 모델/셋팅
- 모델: CatBoost + LightGBM + TabNet 앙상블
- 검증: StratifiedKFold 5-fold
- 후처리: `확정_실패_케이스` 예측값 0 처리

### 결과
- Local CV AUC: CatBoost 0.740026 / LightGBM 0.739016 / TabNet 0.728803 / Ensemble 0.739417
- LB: 미제출

---

## [2026/02/07] 07_autogluonV2.ipynb

### 사용 feature
- 총 101개 (원본 64 + 파생 37, 고결측 5개 제거)

### 모델/셋팅
- 모델: AutoGluon Tabular (stacking, `roc_auc`)

### 결과
- Local CV AUC: 0.738555 (Best: LightGBMLarge_BAG_L1)
- LB: 미제출

---

## [2026/02/07] 08_autogluonV3.ipynb

### 사용 feature
- 검증된 20개 중심 구성, 총 83개

### 모델/셋팅
- 모델: AutoGluon Tabular (stacking, `roc_auc`)

### 결과
- Local CV AUC: 0.738185 (Best: LightGBMLarge_BAG_L1)
- LB: 미제출

---

## [2026/02/09] 03.1_autogluon_top20

### 사용 feature
- 03번 Feature Importance Top 20만 사용

### 모델/셋팅
- 모델: AutoGluon Tabular

### 결과
- Local CV AUC: 0.734957 (Best: WeightedEnsemble_L2)
- LB: 미제출

---

## [2026/02/09] 03.2_autogluon_hypermax

### 사용 feature
- 03번 Feature 유지

### 모델/셋팅
- 모델: AutoGluon Tabular (time/stacking 확장)

### 결과
- Local CV AUC: 0.740810 (Best: WeightedEnsemble_L2)
- LB: 0.7319

---

## [2026/02/09] 03.3_autogluon_seed123

### 사용 feature
- 03번 Feature 유지 (Seed=123)

### 모델/셋팅
- 모델: AutoGluon Tabular

### 결과
- Local CV AUC: 0.73 (Best: RandomForest_BAG_L2)
- LB: 미제출
- RandomForest만 학습됨. overfitting 되어 CV가 저렇게 나온 것 같음



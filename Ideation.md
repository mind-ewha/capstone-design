# Ideation: AI 기반 뇌혈류 영상 분석을 통한 알츠하이머 조기 진단

## 요약 (TL;DR)
BOLD rs-fMRI 및 ASL 기반 뇌혈류(hemodynamics) 대리지표 + 기능 연결망(FC)을 이용해 **경도인지장애(MCI)/초기 AD**를 조기 탐지하는 AI 모델을 구축·해석한다. 
**베이스라인(전통 ML)** → **그래프 신경망(GNN)** → **시계열 Transformer**로 고도화하며, **해석가능성·도메인 보정(ComBat)·재현성**을 핵심 기여로 삼는다.

---

## 1. 배경 및 문제정의
- 알츠하이머병(AD)은 진단 시점이 늦어지기 쉬우며, MCI 단계에서 조기 탐지가 중요하다.
- 뇌 MRI 중 rs-fMRI(BOLD)는 신경혈관활성(cerebrovascular responses)을 간접 반영하고, ASL(Arterial Spin Labeling)은 CBF(cerebral blood flow)를 보다 직접적으로 측정한다.
- **가설**: AD/MCI에서는 특정 네트워크(예: 기본모드 네트워크, DMN)에서 **혈류/기능 연결 패턴**에 체계적 변화가 있으며, 이를 AI로 포착 가능하다.

**연구 목표**
1) 공개 MRI 데이터에서 **혈류 대리지표 및 기능 연결망**을 추출하여 **조기 진단 분류기**를 만든다.  
2) **사이트/스캐너 차이**에 견고한 성능(일반화)을 달성한다.  
3) **모델 해석**을 통해 신경과학적 시사점을 도출한다.

---

## 2. 연구 질문 (RQs)
- **RQ1.** rs-fMRI 기반 **정적/동적 FC**와 **ALFF/fALFF/ReHo**가 MCI/AD 조기 탐지에 얼마나 기여하는가?
- **RQ2.** **GNN**(ROI 그래프)과 **3D-CNN**(구조 MRI) 혹은 **시계열 Transformer** 중 어떤 접근이 일반화 성능 및 해석가능성에서 우수한가?
- **RQ3.** **ComBat 등 배치 효과 보정**과 **교란변수(나이/성별/모션)** 회귀가 성능과 특성 중요도에 미치는 영향은?
- **RQ4.** 모델 해석(엣지/ROI 중요도, Grad-CAM 등)으로 **DMN 등 특정 네트워크의 역할**을 정량화할 수 있는가?

---

## 3. 데이터 전략 (공개 데이터셋 중심)
- **주 데이터**: 공개 rs-fMRI (예: AD/MCI/정상 대조군 포함 데이터셋)  
- **보조 데이터**: 구조 MRI(T1) — 3D-CNN 비교군 및 confound 보정에 활용  
- **대상 레이블**: CN(정상), MCI, AD (이진/다중분류 모두 실험)  
- **표본 규모**: N≈100–500 (데이터셋 조합/필터링에 따라 변동)  
- **윤리**: 공개 데이터의 라이선스 준수, PHI 제거, IRB 불요(공개·비식별 데이터 사용 시)

> 참고 개념: ADNI/HCP/OpenNeuro 등 공개 리포지터리에서 rs-fMRI/T1을 확보. (실제 사용셋은 다운받으며 세부 명시)

---

## 4. 전처리 파이프라인
1) **fMRI 표준 전처리**: 슬라이스 타이밍 보정 → 모션 보정 → 정규화(MNI) → 드리프트/잡음 회귀(CompCor 등) → (선택)공간 스무딩  
2) **파셀레이션**: AAL 또는 Schaefer(100–400 parcel) -> ROI 평균 시계열 추출  
3) **특징 계산**
   - **정적 FC**: ROI 간 상관행렬(피어슨/부분상관) Fisher z 변환  
   - **동적 FC(dFC)**: 슬라이딩 윈도우 또는 HMM 기반 상태 추정  
   - **지역 지표**: **ALFF/fALFF**, **ReHo**  
   - (옵션) **Radiomics(구조 MRI)**: 강도·텍스처 등 정량 특징
4) **보정/정규화**
   - **교란 회귀**: 나이/성별/모션 파라미터  
   - **배치 보정**: **ComBat**로 센터/스캐너 효과 완화  
5) **분할**: **사이트 분리**를 고려한 train/val/test (가능하면 **센터 외부검증**)  

---

## 5. 모델링 계획 (3-트랙 비교)
### Track A — 전통 ML 베이스라인 (빠르고 해석 용이)
- 입력: 상삼각 FC 벡터 + (옵션) ALFF/ReHo
- 모델: **SVM/LogReg/XGBoost**
- 목적: 신속한 baseline; 특징 중요도/해석 기반

### Track B — **GNN (Graph Neural Networks)**
- 그래프: 노드=ROI, 엣지=FC(임계값/Top-k/연속가중)
- 모델: **GCN/GAT/GraphSAGE**, (선택) **Temporal GNN**(dFC)
- 장점: 네트워크 관점 해석(허브, 서브네트워크)

### Track C — **시계열 직접 모델링**
- 입력: ROI 시계열(또는 voxel-level 축약)
- 모델: **LSTM/GRU** → **Transformer Encoder/TimesNet**
- 목적: hemodynamics의 시간적 패턴을 직접 포착

(보너스) **3D-CNN(구조 MRI)**: DenseNet3D/UNETR로 멀티모달 비교군

---

## 6. 성능평가 & 통계
- **지표**: AUC(주), Accuracy, F1, Sensitivity@고정Specificity(임상적 관점), Calibration(ECE)
- **검증**: Stratified k-fold (환자 기준), **사이트 OOD 테스트**(가능 시)
- **통계**: Delong test로 AUC 비교, McNemar for Acc, 다중비교 보정
- **재현성**: 시드 고정, 데이터 분할 명시, 코드·환경 스냅샷

---

## 7. 해석가능성(Explainability)
- **GNN**: 엣지/노드 중요도, attention weights → **상위 ROI/경로 시각화**
- **전통 ML**: SHAP/Permutation importance  
- **3D-CNN**: 3D Grad-CAM/IG (구조 MRI 비교 시)  
- **신경과학적 해석**: DMN/전두-두정 네트워크, 해마 관련 회로 등과 연결

---

## 8. 기대 기여(Contributions)
1) **조기 탐지** 성능: 공개 데이터에서 **일반화 가능한** 조기 진단 신뢰도 보고  
2) **해석가능성**: 네트워크/ROI 수준의 **생물학적 시사점** 제공  
3) **견고성**: **ComBat/교란 회귀**의 효과 정량화, **사이트 분리 검증** 리포트  
4) **재현성 패키지**: 전처리→특징→모델→시각화까지 **오픈 템플릿** 제공

---

## 9. 리스크 & 완화
- **데이터 이질성/소표본** → ComBat, 강한 규제(L2/Dropout), 단순 모델 병행
- **과적합** → 엄격한 분할, Early stopping, 외부 세트 검증
- **전처리 비용** → fMRIPrep/niLearn 파이프라인 재사용, 캐시/병렬화
- **ASL 부족** → rs-fMRI 기반 대리지표 중심으로, ASL은 서브분석

---

## 10. 1년 로드맵 (월 단위)
| 기간 | 목표 |
|---|---|
| 1–2개월 | 주제 확정, 데이터셋 선정 및 라이선스 확인, 환경 세팅(Conda/Poetry, CUDA) |
| 3–4개월 | 전처리 파이프라인 고정(파셀·ALFF/FC 계산), EDA/품질체크 |
| 5–6개월 | **Track A** 베이스라인 완료(SVM/XGB), ablation(파셀 수, 정규화, 보정) |
| 7–8개월 | **Track B** GNN 구현 및 튜닝(임계값/k-NN, 레이어/히든폭) |
| 9–10개월 | **Track C** 시계열 Transformer, dFC/HMM 시도, 성능·해석 비교 |
| 11개월 | 해석(상위 ROI/엣지), 사이트 분리 일반화 실험, 통계검정 |
| 12개월 | 논문 작성(방법/결과/토의/한계/윤리), 코드/데이터카드 정리, 최종 제출 |

---

## 11. 구현 세부 (스택 & 리포 구조 초안)
- **스택**: Python 3.10+, PyTorch, PyTorch Geometric, MONAI(옵션), Nilearn, NiBabel, Scikit-learn, MNE(옵션), pandas, numpy
- **리포 구조 예시**
```
repo/
 ├─ data/                  # 메타/스플릿(비공개), 캐시 설명
 ├─ preprocessing/         # 전처리/파셀/ALFF/FC 스크립트
 ├─ features/              # radiomics, dFC, 지표 계산
 ├─ models/
 │   ├─ baseline/          # SVM/XGB
 │   ├─ gnn/               # GCN/GAT/GraphSAGE
 │   ├─ seq/               # LSTM/Transformer
 ├─ train_eval/            # 공통 루프, k-fold, 로깅
 ├─ analysis/              # 해석/시각화, SHAP, Grad-CAM
 ├─ utils/                 # 공통 함수 (ComBat, split 등)
 ├─ configs/               # 하이퍼파라미터 YAML
 ├─ notebooks/             # EDA/프로토타입
 ├─ Ideation.md            # 아이데이션
 └─ README.md              # 사용법/환경/재현성
```

---

## 12. 평가 기준(성공 정의)
- **AUC ≥ 0.80** (내부 검증), **외부/사이트 분리 세트 AUC ≥ 0.75** 달성  
- 상위 **ROI/엣지**가 **문헌 보고 네트워크(DMN 등)**와 합치되는 해석 결과  
- Ablation으로 **보정(ComBat/Confound)** 효과를 통계적으로 검증

---

## 13. 윤리·저작권
- 공개·비식별 데이터 사용, **IRB 불요**(단, 각 데이터셋 라이선스 준수)  
- 코드/모델 공개 시 **개인정보 불포함** 확인, 모델 오남용(의료행위 대체 아님) 명시

---

## 14. 산출물
- 학부 졸업논문(국/영문 요약 포함)  
- 재현성 패키지(코드/환경 파일/실험 설정)  
- 포스터/슬라이드(그림: 파이프라인, 네트워크 맵, ROC, 해석 시각화)

---

## 15. 관련 키워드
Alzheimer’s disease, Mild Cognitive Impairment, rs-fMRI, ASL, CBF, ALFF, fALFF, ReHo, Functional Connectivity, Dynamic FC, Graph Neural Networks, Transformer, ComBat, Domain Generalization, Explainable AI, Radiomics


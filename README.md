# 경도인지장애(MCI) 환자의 rs-fMRI 뇌혈류 동영상 데이터를 활용한 Radiomics 및 그래프 신경망 기반 알츠하이머 조기 진단 연구

## 1) 논문소개
이 프로젝트는 **rs-fMRI(BOLD) 및(가능 시) ASL 데이터를 활용한 뇌혈류(hemodynamics) 패턴 분석**을 통해 **경도인지장애(MCI)/초기 알츠하이머(AD)를 조기 탐지**하는 AI 방법을 제안합니다.  
핵심 아이디어는 다음과 같습니다.
- 혈류 대리지표(ALFF/fALFF/ReHo)와 기능 연결망(Functional Connectivity; FC)을 결합해 조기 변화 신호를 포착
- 베이스라인(전통 ML) → GNN(그래프) → 시계열 Transformer 순으로 성능·해석력 고도화
- 도메인 보정(ComBat)과 교란변수 회귀로 사이트/스캐너 차이에 견고한 일반화 성능 확보
- 해석가능성(Explainability)로 상위 ROI/네트워크(DMN 등)의 생물학적 시사점 도출

**기대 기여**
1) 공개 데이터 기반으로 **재현성 높은 조기 탐지 성능** 제시  
2) **네트워크 관점 해석**(중요 ROI/엣지 시각화)을 통한 신경과학적 통찰  
3) **일반화/견고성 실험**(센터 분리 검증, ComBat 효과)을 체계적으로 보고

---

## 2) 사용기술
- **언어/프레임워크**: Python 3.10+, PyTorch, PyTorch Geometric, (옵션) MONAI, scikit-learn  
- **신경영상 도구**: Nilearn, NiBabel, (전처리 산출물 재사용 시) fMRIPrep 결과 활용  
- **특징 추출**:  
  - **ALFF/fALFF/ReHo** (지역 자발활동/국소 동기화 지표)  
  - **정적 FC**(ROI 간 상관행렬, Fisher z), **동적 FC**(슬라이딩 윈도우/HMM)  
  - (옵션) **Radiomics(구조 MRI)** 텍스처/형태 특징
- **모델**:  
  - **베이스라인**: SVM / Logistic Regression / XGBoost (FC 벡터 + 지역 지표)  
  - **그래프 신경망**: GCN / GAT / GraphSAGE (노드=ROI, 엣지=FC 가중)  
  - **시계열**: LSTM/GRU → Transformer Encoder / TimesNet (ROI 시계열 직접 모델링)  
  - (옵션) **3D-CNN**: DenseNet3D/UNETR(구조 MRI 비교군)
- **일반화/보정**: **ComBat**(배치효과), 나이/성별/모션 회귀  
- **해석/시각화**: SHAP/Permutation Importance, GNN 엣지/노드 중요도, (옵션) 3D Grad-CAM  
- **평가 지표**: AUC(주), Accuracy, F1, Sensitivity@고정 Specificity, Calibration(ECE)

---

## 3) 문제정의
- **입력(X)**: 정규화된 rs-fMRI에서 추출한 **ROI 시계열**, 파생 **FC 행렬**, **ALFF/fALFF/ReHo** (옵션: 구조 MRI에서 Radiomics)  
- **출력(y)**: 진단 레이블 (예: **CN vs MCI** 이진, 혹은 **CN/MCI/AD** 다중분류)  
- **목표**: 적은 데이터와 강한 도메인 편차에서도 **조기 탐지** 성능과 **해석가능성**을 동시에 달성  
- **제약/도전과제**
  - **도메인 편차**: 다기관/다스캐너 데이터 이질성 → ComBat/강한 정규화 필요  
  - **교란변수**: 나이·성별·모션 파라미터 → 회귀/공변량 통제  
  - **소표본/고차원**: 규제, 특징선택, 단순 모델 병행, 엄격한 데이터 분할
- **평가/실험 디자인**
  - Stratified k-fold(환자 기준), 가능 시 **센터 분리 OOD 테스트**  
  - Delong/McNemar 등 통계 검정과 ablation(파셀 수, 임계값, 보정 유무)

---

## 4) 생각해볼점
- **특징 설계**: FC만으로 충분한가, **ALFF/ReHo** 결합이 실질 이득을 주는가? 동적 FC의 윈도우 길이/중첩 최적값은?  
- **그래프 구성**: 엣지 임계값 vs Top-k vs 연속가중 중 무엇이 일반화에 유리한가? 파셀 수(예: Schaefer 100/200/400)의 trade-off는?  
- **시계열 모델링**: LSTM 대비 Transformer의 장점(긴 의존성)과 데이터 요구량의 균형은?  
- **보정 전략**: ComBat 적용 전/후의 성능 및 특징 중요도 이동은 임상적으로 일관적인가?  
- **해석가능성**: 상위 ROI/엣지가 **DMN/해마 회로** 등 문헌과 합치되는가? 반대로 새로운 신호는 무엇인가?  
- **재현성/윤리**: 코드·스플릿·시드 공개로 재현성 보장. 공개 데이터 라이선스 준수와 “연구용 참고” 고지(의료행위 대체 아님).  
- **확장 가능성**: (옵션) 구조 MRI 멀티모달 융합, 소수샘플 학습(transfer/SSL), 외부 공개셋 추가 검증

> 상세 기획과 일정, 전처리·모델링 스펙은 `Ideation.md`를 참고하세요.

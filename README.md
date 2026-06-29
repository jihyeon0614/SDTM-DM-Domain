# SDTM-DM-Domain
[Clinical SAS] CDISC SDTM DM Domain Mapping - 5-Way Crossover TQT Study
📌 Project Overview
본 프로젝트는 건강한 자원자를 대상으로 한 심전도(ECG) 5기 교차 임상시험(Thorough QT Study, TQT)의 통합 분석용 원시 데이터(Raw Integrated Dataset)를 CDISC SDTM 가이드라인에 맞추어 DM(Demographics) 도메인으로 정규화(Normalization) 및 매핑하는 SAS 프로그래밍 포트폴리오입니다.

원시 데이터는 대상자 1명당 측정 시점(TPT)과 반복 측정(Triplicate)에 의해 수십 개의 행을 가지는 1:N 구조로 이루어져 있습니다. 본 프로젝트에서는 이를 SDTM DM 도메인의 제1원칙인 '1인당 1레코드(1 Record per Subject)' 구조로 변환하고, 표준 메타데이터(Length, Label, Type)를 엄격하게 적용하는 데이터 파이프라인을 구축했습니다.

Study ID: NCT01873950 (가상화 데이터 적용)

Trial Design: 1상(Phase 1), 이중 눈가림(Double-Blind), 무작위 배정(Randomized), 위약 대조(Placebo-Controlled), 5기 교차시험(5-Period Crossover)

Tech Stack: SAS Base, SAS Macro, CDISC SDTM IG 3.4

💡 Key Technical Highlights
1. 빈 쉘 기법 (Empty Shell Approach) 적용
임상 실무의 정석적인 데이터 세팅 방식인 '빈 쉘(Empty Shell)' 기법을 활용했습니다.
데이터 조작 스텝(form_DM) 이전에 SDTM IG 규격에 맞는 변수 길이(Length), 타입, 라벨(Label)을 정의한 shell_DM을 선언하고 이를 최종 데이터와 병합(set shell_DM form_DM)하여, 변수 잘림(Truncation)이나 속성 충돌 에러를 원천 차단하고 무결성을 확보했습니다.

2. 다중 교차시험 치료군(ARM) 동적 생성 (Dynamic String Parsing)
5기 교차시험의 특성상 환자마다 배정된 투약 순서(Sequence)가 모두 다릅니다. (예: A,C,E,D,B vs C,A,B,E,D)
하드코딩을 배제하고 어떠한 무작위 배정 순서가 들어와도 유연하게 처리할 수 있도록 동적 프로그래밍을 구현했습니다.

translate() 함수로 원시 데이터의 구분자(,)를 하이픈(-)으로 전처리하여 ARMCD 생성.

countw(), scan(), put() 함수 및 DO 루프를 결합하여, 개별 투약 코드를 파싱하고 $arm_code. 포맷을 참조해 전체 투약 시퀀스 풀네임(ARM)을 자동 조립.

3. 임상 프로토콜 기반 가상 날짜 산출 (Mock Date Imputation)
제공된 원시 데이터는 절대 날짜(Calendar Date)가 부재하고 투약 기준 상대적 시간(TPT)만 존재하는 형태였습니다. SDTM 날짜 함수(--DTC) 처리 역량을 보여주기 위해, 프로토콜 일정에 기반한 가상 날짜 생성 로직을 설계했습니다.

환자의 스크리닝 나이(AGE)를 기반으로 출생 연도(BRTHDTC) 역산.

mdy(), ranuni() 등 SAS 내장 함수를 활용하여 무작위 최초 투여일(RFSTDTC) 생성.

교차시험의 7일 휴약기(Washout Period) 스케줄과 5번의 투약 기수(Period)를 반영하여 마지막 투여일(RFENDTC)을 논리적으로 추정 연산.

🛠️ Data Processing Pipeline
Data Import & Deduplication

PROC IMPORT: CSV 원시 데이터(.csv) 로드.

PROC SORT NODUPKEY: 대상자 식별자(RANDID) 기준으로 중복 행을 제거하여 1인당 1행 추출.

Metadata Shell Creation

DATA shell_DM: SDTM 25개 필수/권장 변수의 속성 및 라벨 정의.

Variable Mapping & Formatting

PROC FORMAT: 치료 약물(Ranolazine, Dofetilide, Verapamil, Quinidine, Placebo) 변환 포맷 지정.

DATA form_DM: 식별자 매핑, 가상 날짜 생성, 문자열 처리(ARMCD, ARM).

Final Integration

DATA final_DM: Shell 데이터와 Mapped 데이터를 병합 후 KEEP 구문으로 규정된 변수만 보존.

PROC SORT: 최종 도메인 규정에 따라 USUBJID 기준으로 정렬.

⚠️ Disclaimer (Troubleshooting & SDTM Rules)
Mock Data Imputation Note:
본 프로젝트에 사용된 분석용 통합 데이터셋에는 달력 날짜(Calendar Date)가 부재하여, 원래 원칙대로라면 EX(Exposure) 도메인에서 가져와야 할 DM 도메인의 참조 날짜(RFSTDTC, RFXENDTC 등) 산출이 불가능했습니다.
본 코드에 포함된 가상 날짜(mdy, ranuni 활용) 생성 로직은 SAS 날짜 함수 핸들링 스킬과 임상 프로토콜(휴약기)에 대한 도메인 지식을 증명하기 위한 **포트폴리오 목적의 가상 연산(Mock Logic)**입니다. 실제 실무 SDTM 구축 시에는 결측치를 역산(Imputation)하지 않고 원본 데이터(eCRF, EX Domain)의 기록을 100% 준수해야 함을 인지하고 있습니다.

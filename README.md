# MOA · AI 기반 상의 착용 합성 웹 서비스

상의 사진과 내 사진을 올리면, 그 옷을 입은 모습을 AI가 합성해 구매 전에 보여 주는 웹 서비스.

> 팀 저장소 [Jenny5789/CGI_MOA](https://github.com/Jenny5789/CGI_MOA)의 Fork. 코드와 문서는 팀 저장소와 같고, 이 README만 서지섭 담당 범위 기준으로 새로 작성.

<img src="docs/readme/moa_home.jpg" width="860" alt="MOA 홈 화면">

| 항목 | 내용 |
|---|---|
| 형태 | 4인 팀 프로젝트, 팀장 최이재와 함께 구현 · (주)큐빅테크 연계 미래내일일경험 프로젝트 |
| 기간 | 2026.09.28 ~ 2026.11.22 |
| 담당 | 합성 모델 실행 환경·속도 개선, AnyDoor 추가 학습, 합성 품질 개선, 합성하기 화면, 모델 추론 서버 분리·배포 |
| 결과 | 합성 1건 분 단위 → 약 25초 (RTX 3080 10GB) |
| 기술 | SAM2 · AnyDoor · PyTorch · FastAPI · React + Vite · MySQL · Docker |

## 기획 과정

| 단계 | 내용 |
|---|---|
| 문제 발견 | 온라인 의류 구매 시 입어 볼 수 없고 상품 사진·설명에 의존, 모델 착용 사진과 내 모습 차이로 반품·교환 발생 |
| 기존 사례 | 쇼핑 플랫폼 가상 착용(Google 검색 Try on, 국내 패션 플랫폼): 대상 상품 한정 또는 AI 모델 착용 이미지 / 공개 가상 착용 모델(IDM-VTON, OOTDiffusion, CatVTON 등): 옷·사람 영역을 주로 인체 분할·포즈 추정으로 자동 지정 / AnyDoor: 범용 이미지 합성 모델, 공식 저장소에 가상 착용 전용 설계·학습 없음 명시 |
| 개선 방향 | 사용자가 SAM2 점 찍기로 상의와 착용 영역을 지정, 범용 합성 모델 AnyDoor를 착용 데이터(VITON-HD, 쉐이프리스)로 추가 학습 |
| 차별점 | 상품 사진·착용 사진에서 사용자가 고른 영역 기준 합성, 가상 착용 전용 설계가 없는 AnyDoor의 착용 데이터 추가 학습과 배포 |

*기존 사례: 2026년 11월 확인 기준 ([Google 검색 Try on](https://support.google.com/googleshopping/answer/16253678), [IDM-VTON](https://github.com/yisol/IDM-VTON), [OOTDiffusion](https://github.com/levihsu/OOTDiffusion), [CatVTON](https://arxiv.org/abs/2407.15886), [AnyDoor](https://github.com/ali-vilab/AnyDoor)).*

---

## 담당 범위

[작업일지_서지섭.md](docs/작업일지_서지섭.md) 기준. 팀원 단독 작업은 제외하고, 공동 작업은 역할을 나눠 표기.

| 영역 | 담당 내용 |
|---|---|
| 합성 속도·GPU 메모리 | AnyDoor 공식 실행 환경 Windows 재현, 체크포인트 시스템 메모리 로드, float16 혼합 정밀도, 모델 부품 단위 GPU 적재, 작업 후 GPU 메모리 반환 |
| AnyDoor 추가 학습 | 학습 데이터 구성(VITON-HD, AI-Hub 쉐이프리스 의류 데이터), 클라우드 GPU(B200) 학습 실행, 학습 결과 평가, 배포 가중치 선정. 학습 코드와 입력 형식 설계는 팀장 최이재 |
| 합성 품질 | 물체 기준 원본 복원, Real-ESRGAN 초해상도 복원, SAM2 분할 정확도 평가, 합성 설정 비교·고정 |
| 웹 화면 | React 합성하기 4단계 화면, 마스크 다듬기 창, 회원가입, 마이페이지, 로그인 연장 화면 |
| 모델 추론 분리·배포 | 모델 추론 서버와 컨테이너, 웹 서비스 서버의 모델 호출 전환, 운영 서버 서비스 컨테이너 구성·배포 |
| 코드 검토·문서 | 코드 검토와 정리, 화면·개발 환경 설치 문서 |

## 주요 화면

| 1단계 · 상의 사진 선택 | 2단계 · 인물 사진 선택 |
|---|---|
| <img src="docs/readme/moa_screen_step1.jpg" width="420" alt="1단계 상의 사진 선택"> | <img src="docs/readme/moa_screen_step2.jpg" width="420" alt="2단계 인물 사진 선택"> |

| 3단계 · 상의 영역 지정 (왼쪽 클릭 포함 점, 오른쪽 클릭 제외 점) | 합성 결과 (상의 사진, 인물 사진, 결과) |
|---|---|
| <img src="docs/readme/moa_screen_step3.jpg" width="420" alt="3단계 상의 영역 지정"> | <img src="docs/readme/moa_before_after.jpg" width="420" alt="상의 사진, 인물 사진, 합성 결과"> |

## 서비스 구조

<img src="docs/readme/moa_system_architecture.png" width="860" alt="MOA 서비스 구조">

화면(React)·웹 서비스 서버(FastAPI)·DB(MySQL)는 GPU 없는 운영 서버에서, 이미지 합성 모델은 GPU 장비의 모델 추론 서버에서 실행.

---

## 문제 해결

### 1. 합성 시간: 분 단위 → 약 25초

| 구분 | 내용 |
|---|---|
| 문제 | AnyDoor 공식 설정 기준 RTX 3080 10GB에서 합성 1건(50단계) 11~15분 |
| 원인 | 학습용 값까지 포함한 16.8GB 체크포인트 전체의 GPU 적재, 모델 부품 4개(U-Net·ControlNet·VAE·DINOv2) 일괄 적재로 VRAM 10GB 초과 → 공유 GPU 메모리(시스템 메모리) 사용에 따른 연산 지연 |
| 해결 | 아래 3단계 |
| 판단 근거 | 같은 조건으로 수정 전후 측정, float16과 부품 단위 적재는 결과 이미지 차이까지 확인 후 적용 |

| 단계 | 변경 | 확인 |
|---|---|---|
| 체크포인트 시스템 메모리 로드 | 시스템 메모리에서 체크포인트를 연 뒤 추론 가중치 약 9.2GB만 GPU 적재 | 공유 GPU 메모리 0.5GB 수준 안정화 |
| float16 혼합 정밀도 | 행렬 곱·합성곱·어텐션만 16비트, 가중치는 32비트 유지 | 1장 225.0초 → 47.8초, float32 대비 평균 픽셀 차이 0.17/255 |
| 모델 부품 단위 GPU 적재 | 대형 부품을 CPU에 둔 채 적재, 기존 묶음 해제 → 캐시 비움 → 신규 묶음 적재 순서로 변경 | 평균 39.0초 → 24.9초, 공유 GPU 메모리 최대 3,199MB → 218MB, 결과 이미지 픽셀 단위 동일 |

부품 단위 적재는 적재 방식만 바꿨을 때 효과가 없었고, 묶음 교체 순서를 함께 바꾼 뒤 효과 확인. (`features/composition/anydoor_composer.py`, 작업일지 10/12-③·④, 10/21-①)

### 2. 공식 가중치의 엉뚱한 천 → AnyDoor 추가 학습

| 구분 | 내용 |
|---|---|
| 문제 | 공식 가중치 합성 결과에서 입고 있던 상의 자리에 긴소매·늘어진 천이 남는 사례 |
| 해결 | 학습 데이터 3가지 안 구성(VITON-HD 단독·증강 유무, VITON-HD + 쉐이프리스), 클라우드 GPU(B200)에서 안별 학습, 검증 손실·평가 95쌍 지표·결과 그림 판정으로 비교 |
| 결과 | 재학습 가중치 9개 모두 긴소매·늘어진 천 없음. 서비스 합성 비교 후 `mix35000`(3안 35,000단계) 배포 |
| 판단 근거 | 지표는 공식 가중치가 근소하게 우세했지만, 같은 입력·seed 서비스 합성에서 `mix35000`이 자세·상품 형태를 가장 안정적으로 유지하고 깨진 사례 0건 |
| 역할 | 학습 코드(`scripts/retrain`)와 입력 형식 설계는 팀장 최이재, 데이터 준비·학습 실행·평가·앱 적용은 서지섭 |

(작업일지 4장·5장, `가중치_비교_자료/배포_가중치_결정.md`)

### 3. 합성 결과의 잔상·붉은 얼룩 → 물체 기준 원본 복원

| 구분 | 내용 |
|---|---|
| 문제 | 운영 서버 합성 결과에 소매 바깥 세로 잔상, 왼쪽 아래 붉은 얼룩 |
| 원인 | 같은 입력·seed 재현 시 모델 원래 결과에는 잔상이 없고 원본 복원 후처리 뒤에만 발생. 원본 배경과 다시 그린 배경의 밝기 차이가 전환선을 따라 노출, 붉은 얼룩은 경계 보정이 손·청바지 색 차이를 안쪽으로 전파 |
| 해결 | SAM2 분할로 원본 사람·합성 결과 사람·새 상의·원래 상의를 나누고, 배경·얼굴·머리는 원본, 합성 결과 사람과 새 상의만 합성 결과 사용 (`features/composition/restore.py`) |
| 판단 근거 | 전환선 밝기를 맞추는 보정은 경계 구조가 남는 방식이라 미채택, 원인인 영역 재생성 구조 기준으로 복원 |
| 결과 | 다시 그린 영역 안 배경의 원본 유지 비율 85.5~91.3% → 98.1~99.2%, 잔상·얼룩 없음 (모델 원래 결과 6장, 가중치 3종, 셔츠 3종) |

(작업일지 11/10-④)

### 4. GPU 없는 운영 서버 배포 → 모델 추론 서버 분리

| 구분 | 내용 |
|---|---|
| 문제 | 운영 서버에 GPU가 없고, 웹 서비스 서버가 이미지 합성 모델을 적재하는 구조 |
| 해결 | 모델 추론 서버 분리(웹 서비스 서버는 HTTP로 요청), GPU 장비의 모델 추론 컨테이너와 운영 서버의 서비스 컨테이너 3개(화면·웹 서비스 서버·MySQL) 구성 |
| 판단 근거 | 모델을 컨테이너로 실행하면 컨테이너 없이 실행할 때보다 합성이 2~4초 느리지만, 모델 추론 환경까지 Dockerfile로 정의되어 다른 GPU 장비로 옮기기 쉽고 재시작 정책으로 자동 실행 → 배포·시연은 컨테이너, 평소 개발은 컨테이너 없이 실행 |
| 결과 | 두 호출 방식 결과 픽셀 동일, 운영 서버 종단 시험 3회 통과 |

(작업일지 8장, [deploy/README.md](deploy/README.md), [model_server/README.md](model_server/README.md))

---

## 설치와 실행

- 개발 환경 설치: [docs/setup.md](docs/setup.md)
- Docker 실행: [docs/docker.md](docs/docker.md), 운영 서버 배포: [deploy/README.md](deploy/README.md)
- 팀 전체 구조·처리 흐름·API는 팀 저장소 [README](https://github.com/Jenny5789/CGI_MOA#readme)와 [docs/api.md](docs/api.md)

## 관련 문서

| 문서 | 내용 |
|---|---|
| [docs/작업일지_서지섭.md](docs/작업일지_서지섭.md) | 담당 작업의 목적·내용·결과·확인 방법 |
| [features/composition/README.md](features/composition/README.md) | 이미지 합성 모듈 |
| [frontend/README.md](frontend/README.md) | 화면 구성 |
| [model_server/README.md](model_server/README.md) | 모델 추론 서버 |

## 참고 자료

| 대상 | 출처 | 라이선스 | MOA 사용 범위 |
|---|---|---|---|
| AnyDoor | [ali-vilab/AnyDoor](https://github.com/ali-vilab/AnyDoor), [논문 arXiv:2307.09481](https://arxiv.org/abs/2307.09481) | MIT | 착용 합성 모델 코드·가중치 (커밋 `44ca2b2` 고정) |
| SAM 2 | [facebookresearch/sam2](https://github.com/facebookresearch/sam2) | Apache-2.0 | 상의·착용 영역 추출 모델 |
| DINOv2 | [facebookresearch/dinov2](https://github.com/facebookresearch/dinov2) | Apache-2.0 | AnyDoor가 상의 특징을 읽는 데 쓰는 모델 가중치 |
| VITON-HD | [shadow2496/VITON-HD](https://github.com/shadow2496/VITON-HD) | 연구용 | 추출 정확도 평가, AnyDoor 재학습 데이터. 저장소 미포함 |

- AnyDoor는 [ControlNet](https://github.com/lllyasviel/ControlNet) 코드 기반

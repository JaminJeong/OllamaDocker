# 파일 구조 상세 설명

## 디렉토리 트리

```
OllmaRunTime/
├── README.md
├── STRUCTURE.md
├── environment.yml
├── build.sh
├── docker-compose.yml
├── .env.example
└── notebooks/             # (선택) JupyterLab 작업 파일 저장 위치
```

---

## 파일별 역할

### `environment.yml` — Conda 환경 정의

Conda가 읽는 환경 명세 파일입니다. 이 파일 하나로 Python 버전과 모든 패키지를 재현할 수 있습니다.

```
환경 이름: ollama-env
Python:    3.11
```

**포함 패키지 분류**

| 분류 | 패키지 | 용도 |
|------|--------|------|
| Conda 네이티브 | `python`, `pip`, `curl`, `ipykernel`, `jupyterlab`, `notebook`, `ipywidgets` | 기본 환경 및 노트북 |
| Ollama 클라이언트 | `ollama` | Ollama REST API Python 래퍼 |
| HTTP | `httpx`, `requests` | API 직접 호출 |
| LangChain | `langchain`, `langchain-ollama`, `langchain-community` | LLM 체인 구성 |
| LlamaIndex | `llama-index-core`, `llama-index-llms-ollama`, `llama-index-embeddings-ollama` | RAG 파이프라인 |
| 유틸리티 | `python-dotenv`, `pydantic`, `rich`, `tqdm` | 설정 관리, 출력 포맷 |

**환경 재생성 / 업데이트 방법**

```bash
# 최초 생성
conda env create -f environment.yml

# 패키지 추가 후 업데이트
conda env update -n ollama-env -f environment.yml --prune

# 현재 상태를 파일로 내보내기 (버전 고정 스냅샷)
conda env export -n ollama-env > environment.lock.yml
```

---

### `build.sh` — 빌드 및 설치 스크립트

단계별 또는 한 번에 전체 환경을 구성하는 자동화 스크립트입니다.

**실행 흐름 (all 모드)**

```
build.sh all
  │
  ├─ 1. install_ollama()      Ollama 바이너리 설치 (curl 스크립트 실행)
  ├─ 2. detect_gpu()          nvidia-smi로 GPU 감지 → GPU_TYPE 설정
  ├─ 3. setup_conda_env()     environment.yml로 conda 환경 생성 또는 업데이트
  ├─ 4. start_ollama_service() nohup으로 백그라운드 서비스 실행 + 헬스체크
  ├─ 5. pull_default_models() 기본 모델 Pull (GPU 여부에 따라 목록 다름)
  └─ 6. print_usage()         사용법 출력
```

**서브커맨드**

```bash
./build.sh all      # 전체 실행 (기본값)
./build.sh install  # Ollama 바이너리 설치만
./build.sh conda    # Conda 환경 생성/업데이트만
./build.sh start    # Ollama 서비스 시작만
./build.sh models   # 기본 모델 Pull만
```

**환경변수 오버라이드**

```bash
OLLAMA_HOST=127.0.0.1 OLLAMA_PORT=12000 ./build.sh start
```

---

### `docker-compose.yml` — Docker Compose 서비스 정의

프로파일(Profile) 기반으로 필요한 서비스만 선택해서 실행할 수 있습니다.

**서비스 의존 관계**

```
ollama-cpu   ──┐
               ├──► open-webui  (webui 프로파일)
ollama-nvidia ─┘

ollama-cpu 또는 ollama-nvidia ──► jupyter  (jupyter 프로파일)

ollama-pull  (독립 실행, 일회성 모델 Pull)
```

**공통 설정 (`x-ollama-common`)**

YAML 앵커(`&ollama-common`)로 CPU/GPU 서비스 간 중복을 제거합니다.

```yaml
x-ollama-common: &ollama-common
  image: ollama/ollama:latest
  ports:   ["11434:11434"]
  volumes: [ollama_data:/root/.ollama]
  healthcheck: ...
```

**볼륨 구성**

| 볼륨 | 컨테이너 경로 | 용도 |
|------|--------------|------|
| `ollama_data` | `/root/.ollama` | 모델 파일 영구 저장 |
| `webui_data` | `/app/backend/data` | WebUI 대화 기록 저장 |
| `./notebooks` (바인드 마운트) | `/home/jovyan/work` | 노트북 파일 로컬 동기화 |

**프로파일별 실행 예시**

```bash
# CPU + WebUI
docker compose --profile cpu --profile webui up -d

# GPU + WebUI + JupyterLab
docker compose --profile nvidia --profile webui --profile jupyter up -d

# 모델 사전 Pull 후 CPU 서비스 시작
docker compose --profile cpu --profile pull up -d
```

---

### `.env.example` — 환경변수 템플릿

Docker Compose와 build.sh 양쪽에서 참조하는 설정값 템플릿입니다.

```bash
cp .env.example .env
# .env 파일을 열어 WEBUI_SECRET_KEY 등 민감 값 변경
```

**주요 변수**

| 변수 | 기본값 | 영향 범위 |
|------|--------|-----------|
| `OLLAMA_PORT` | `11434` | Docker + build.sh |
| `OLLAMA_KEEP_ALIVE` | `5m` | Docker (모델 메모리 유지) |
| `OLLAMA_MAX_LOADED_MODELS` | `2` | Docker (동시 로드 모델 수) |
| `DEFAULT_MODEL` | `llama3.2:3b` | Docker pull 서비스 |
| `WEBUI_PORT` | `3000` | Docker open-webui |
| `WEBUI_SECRET_KEY` | `changeme-...` | Docker open-webui (변경 필수) |
| `JUPYTER_PORT` | `8888` | Docker jupyter |

---

## 환경 선택 가이드

```
로컬 개발 / 빠른 테스트
  └─► build.sh + Conda 환경

팀 공유 / 서버 배포 / 격리 필요
  └─► Docker Compose

GPU 없는 환경
  └─► profile: cpu

NVIDIA GPU 있는 환경
  └─► profile: nvidia  (nvidia-container-toolkit 사전 설치 필요)

웹 UI로 채팅
  └─► profile: webui  →  http://localhost:3000

노트북 실험
  └─► profile: jupyter  →  http://localhost:8888
      또는 conda activate ollama-env && jupyter lab
```

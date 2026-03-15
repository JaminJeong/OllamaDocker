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
```

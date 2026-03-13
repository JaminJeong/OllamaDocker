# OllmaRunTime

Ollama를 로컬 또는 Docker 환경에서 빠르게 구성하기 위한 Anaconda 환경 및 Docker Compose 템플릿입니다.

---

## 목차

- [파일 구조](#파일-구조)
- [사전 요구사항](#사전-요구사항)
- [빠른 시작](#빠른-시작)
- [Conda 환경 (로컬 설치)](#conda-환경-로컬-설치)
- [Docker Compose 환경](#docker-compose-환경)
- [Python 사용 예제](#python-사용-예제)
- [환경변수 설정](#환경변수-설정)
- [문제 해결](#문제-해결)

---

## 파일 구조

```
OllmaRunTime/
├── README.md              # 이 파일 — 전체 사용 가이드
├── STRUCTURE.md           # 파일별 상세 구조 설명
├── environment.yml        # Conda 환경 정의 (Python 패키지 목록)
├── build.sh               # 설치/빌드 자동화 스크립트
├── docker-compose.yml     # Docker Compose 서비스 정의
├── .env.example           # 환경변수 템플릿
└── notebooks/             # JupyterLab 작업 디렉토리 (Docker 마운트용)
```

---

## 사전 요구사항

### 로컬 (Conda) 환경

| 항목 | 버전 | 비고 |
|------|------|------|
| Anaconda / Miniconda | 최신 | [설치 가이드](https://docs.conda.io/en/latest/miniconda.html) |
| curl | 任意 | Ollama 설치 스크립트 실행에 필요 |
| NVIDIA 드라이버 | ≥ 520 | GPU 사용 시 |

### Docker 환경

| 항목 | 버전 | 비고 |
|------|------|------|
| Docker Engine | ≥ 24.0 | |
| Docker Compose | ≥ 2.20 | |
| NVIDIA Container Toolkit | 최신 | GPU 사용 시 필수 |

---

## 빠른 시작

### 로컬 Conda 환경

```bash
# 1. 저장소 클론
git clone <repo-url> OllmaRunTime
cd OllmaRunTime

# 2. 전체 설치 (Ollama + Conda 환경 + 기본 모델)
./build.sh all

# 3. 환경 활성화
conda activate ollama-env

# 4. 동작 확인
curl http://localhost:11434/api/tags
```

### Docker (CPU)

```bash
cp .env.example .env
docker compose --profile cpu up -d
```

### Docker (NVIDIA GPU)

```bash
cp .env.example .env
docker compose --profile nvidia up -d
```

---

## Conda 환경 (로컬 설치)

### build.sh 명령어

```bash
./build.sh [명령어]
```

| 명령어 | 설명 |
|--------|------|
| `all` | 전체 설치 (기본값) |
| `install` | Ollama 바이너리만 설치 |
| `conda` | Conda 환경만 생성/업데이트 |
| `start` | Ollama 서비스만 시작 |
| `models` | 기본 모델만 Pull |

### Conda 환경 직접 관리

```bash
# 환경 생성
conda env create -f environment.yml

# 환경 업데이트 (패키지 변경 후)
conda env update -n ollama-env -f environment.yml --prune

# 환경 활성화
conda activate ollama-env

# 환경 삭제
conda env remove -n ollama-env

# 환경 내보내기 (현재 상태 스냅샷)
conda env export -n ollama-env > environment.lock.yml
```

### Ollama 서비스 관리

```bash
# 서비스 시작 (백그라운드)
OLLAMA_HOST=0.0.0.0:11434 nohup ollama serve > /tmp/ollama.log 2>&1 &

# 서비스 로그 확인
tail -f /tmp/ollama.log

# 서비스 중지
pkill ollama

# 모델 목록 확인
ollama list

# 모델 Pull
ollama pull llama3.2:3b
ollama pull nomic-embed-text

# 대화형 실행
ollama run llama3.2:3b
```

### JupyterLab 실행

```bash
conda activate ollama-env
jupyter lab
# 브라우저: http://localhost:8888
```

---

## Docker Compose 환경

### 프로파일(Profile) 구성

`docker-compose.yml`은 프로파일 기반으로 서비스를 선택적으로 실행합니다.

| 프로파일 | 서비스 | 포트 | 설명 |
|----------|--------|------|------|
| `cpu` | `ollama-cpu` | 11434 | CPU 전용 Ollama |
| `nvidia` | `ollama-nvidia` | 11434 | NVIDIA GPU Ollama |
| `webui` | `open-webui` | 3000 | 웹 채팅 인터페이스 |
| `jupyter` | `jupyter` | 8888 | JupyterLab 노트북 |
| `pull` | `ollama-pull` | — | 모델 자동 Pull (일회성) |

### 실행 조합 예시

```bash
# CPU + 웹 UI
docker compose --profile cpu --profile webui up -d

# GPU + 웹 UI
docker compose --profile nvidia --profile webui up -d

# CPU + JupyterLab
docker compose --profile cpu --profile jupyter up -d

# 모든 서비스 (GPU 환경)
docker compose --profile nvidia --profile webui --profile jupyter up -d
```

### 서비스 관리

```bash
# 상태 확인
docker compose ps

# 로그 확인
docker compose logs -f ollama-cpu
docker compose logs -f open-webui

# 서비스 중지
docker compose --profile cpu --profile webui down

# 볼륨 포함 완전 삭제
docker compose --profile cpu --profile webui down -v

# 모델 Pull (init container 실행)
docker compose --profile cpu --profile pull up ollama-pull
```

### 컨테이너 내 Ollama 명령 실행

```bash
# 모델 Pull
docker exec ollama-cpu ollama pull llama3.2:3b

# 모델 목록 확인
docker exec ollama-cpu ollama list

# 대화형 실행
docker exec -it ollama-cpu ollama run llama3.2:3b
```

### NVIDIA Container Toolkit 설치 (GPU 사용 시)

```bash
# Ubuntu/Debian
curl -fsSL https://nvidia.github.io/libnvidia-container/gpgkey \
  | sudo gpg --dearmor -o /usr/share/keyrings/nvidia-container-toolkit-keyring.gpg

curl -s -L https://nvidia.github.io/libnvidia-container/stable/deb/nvidia-container-toolkit.list \
  | sed 's#deb https://#deb [signed-by=/usr/share/keyrings/nvidia-container-toolkit-keyring.gpg] https://#g' \
  | sudo tee /etc/apt/sources.list.d/nvidia-container-toolkit.list

sudo apt-get update && sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

---

## Python 사용 예제

### 기본 채팅

```python
import ollama

response = ollama.chat(
    model="llama3.2:3b",
    messages=[{"role": "user", "content": "안녕하세요!"}]
)
print(response["message"]["content"])
```

### 스트리밍 응답

```python
import ollama

stream = ollama.chat(
    model="llama3.2:3b",
    messages=[{"role": "user", "content": "Python의 장점을 설명해줘"}],
    stream=True
)
for chunk in stream:
    print(chunk["message"]["content"], end="", flush=True)
```

### 임베딩 생성

```python
import ollama

result = ollama.embeddings(
    model="nomic-embed-text",
    prompt="Ollama는 로컬 LLM 실행 도구입니다."
)
print(result["embedding"][:5])  # 벡터 앞 5개 출력
```

### LangChain 연동

```python
from langchain_ollama import ChatOllama

llm = ChatOllama(model="llama3.2:3b", base_url="http://localhost:11434")
response = llm.invoke("LangChain이란 무엇인가요?")
print(response.content)
```

### LlamaIndex 연동

```python
from llama_index.llms.ollama import Ollama
from llama_index.core import Settings

Settings.llm = Ollama(model="llama3.2:3b", base_url="http://localhost:11434")
response = Settings.llm.complete("LlamaIndex 소개")
print(response)
```

---

## 환경변수 설정

`.env.example`을 복사하여 `.env`로 사용합니다.

```bash
cp .env.example .env
```

| 변수 | 기본값 | 설명 |
|------|--------|------|
| `OLLAMA_PORT` | `11434` | Ollama API 포트 |
| `OLLAMA_KEEP_ALIVE` | `5m` | 모델 메모리 유지 시간 |
| `OLLAMA_MAX_LOADED_MODELS` | `2` | 동시 로드 최대 모델 수 |
| `OLLAMA_NUM_PARALLEL` | `2` | 병렬 요청 수 |
| `DEFAULT_MODEL` | `llama3.2:3b` | 자동 Pull 기본 모델 |
| `EMBED_MODEL` | `nomic-embed-text` | 임베딩 모델 |
| `WEBUI_PORT` | `3000` | Open-WebUI 포트 |
| `WEBUI_SECRET_KEY` | `changeme-...` | WebUI 비밀 키 (변경 필수) |
| `JUPYTER_PORT` | `8888` | JupyterLab 포트 |

---

## 문제 해결

### Ollama 서비스가 응답하지 않을 때

```bash
# 로그 확인
tail -f /tmp/ollama.log          # 로컬
docker compose logs -f ollama-cpu # Docker

# 포트 사용 여부 확인
ss -tlnp | grep 11434

# 프로세스 확인
pgrep -a ollama
```

### Conda 환경 생성 실패

```bash
# conda 버전 확인
conda --version

# 채널 업데이트 후 재시도
conda update conda
conda env create -f environment.yml
```

### GPU가 인식되지 않을 때 (Docker)

```bash
# NVIDIA 드라이버 확인
nvidia-smi

# container-toolkit 확인
nvidia-ctk --version

# Docker GPU 접근 테스트
docker run --rm --gpus all nvidia/cuda:12.0-base nvidia-smi
```

### 모델 Pull 속도가 느릴 때

```bash
# 현재 다운로드 진행 상황 확인
ollama pull llama3.2:3b

# 이미 받은 모델 확인
ollama list
```

---

## 관련 링크

- [Ollama 공식 사이트](https://ollama.com)
- [Ollama 모델 라이브러리](https://ollama.com/library)
- [Open-WebUI GitHub](https://github.com/open-webui/open-webui)
- [LangChain-Ollama 문서](https://python.langchain.com/docs/integrations/providers/ollama/)
- [LlamaIndex-Ollama 문서](https://docs.llamaindex.ai/en/stable/examples/llm/ollama/)

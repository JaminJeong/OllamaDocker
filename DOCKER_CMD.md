# Docker Compose 명령어 사용법

> `docker-compose.yml` 기준으로 사용하는 명령어를 정리한 문서입니다.

---

## 📋 프로파일(Profile) 구성

| Profile    | 설명                                      |
|-----------|-------------------------------------------|
| `cpu`     | CPU 전용 Ollama 서버                       |
| `nvidia`  | NVIDIA GPU Ollama 서버                     |
| `webui`   | Open-WebUI 웹 인터페이스 (브라우저 UI)      |
| `jupyter` | JupyterLab (Python 노트북 환경)            |
| `pull`    | 모델 자동 Pull (초기화 컨테이너)             |

---

## 🚀 시작 (up)

### CPU 모드로 실행
```bash
docker compose --profile cpu up -d
```

### NVIDIA GPU 모드로 실행
```bash
docker compose --profile nvidia up -d
```

### CPU + Open-WebUI 같이 실행
```bash
docker compose --profile cpu --profile webui up -d
```

### NVIDIA GPU + Open-WebUI 같이 실행
```bash
docker compose --profile nvidia --profile webui up -d
```

### JupyterLab 추가 실행
```bash
docker compose --profile cpu --profile jupyter up -d
# 또는 GPU
docker compose --profile nvidia --profile jupyter up -d
```

### 모델 자동 Pull (초기 설정 시)
```bash
docker compose --profile pull up
```
> `DEFAULT_MODEL`, `EMBED_MODEL` 환경변수에 지정된 모델을 자동으로 다운로드합니다.

---

## 🛑 중지 (down)

### 컨테이너 중지 및 제거
```bash
docker compose down
```

### 볼륨까지 함께 삭제 (데이터 초기화)
```bash
docker compose down -v
```

---

## 🔄 재시작

### 전체 재시작
```bash
docker compose --profile cpu restart
```

### 특정 서비스만 재시작
```bash
docker compose restart ollama-cpu
docker compose restart open-webui
docker compose restart ollama-jupyter
```

---

## 📋 상태 확인

### 실행 중인 컨테이너 목록
```bash
docker compose ps
```

### 헬스체크 포함 상세 상태
```bash
docker compose ps --format "table {{.Name}}\t{{.Status}}\t{{.Ports}}"
```

---

## 📄 로그 확인

### 전체 로그
```bash
docker compose logs
```

### 실시간 로그 스트리밍
```bash
docker compose logs -f
```

### 특정 서비스 로그
```bash
docker compose logs -f ollama-cpu
docker compose logs -f ollama-nvidia
docker compose logs -f open-webui
docker compose logs -f ollama-jupyter
```

### 최근 N줄만 출력
```bash
docker compose logs --tail=100 ollama-cpu
```

---

## 🖥️ 컨테이너 내부 접속

### Ollama 컨테이너 접속 (CPU)
```bash
docker exec -it ollama-cpu bash
```

### Ollama 컨테이너 접속 (NVIDIA)
```bash
docker exec -it ollama-nvidia bash
```

### Open-WebUI 컨테이너 접속
```bash
docker exec -it open-webui bash
```

---

## ▶️ 일회성 실행 (run)

`docker compose run`은 서비스를 **일회성으로** 실행할 때 사용합니다.  
`up`과 달리 컨테이너 종료 후 자동으로 삭제되며, 포트는 기본적으로 바인딩되지 않습니다.

### `up` vs `run` 차이점

| 항목              | `up`                      | `run`                        |
|-----------------|--------------------------|------------------------------|
| 목적             | 서비스 상시 실행            | 일회성 작업 실행               |
| 포트 바인딩       | 자동 바인딩                | 기본 바인딩 안 함 (`-p` 필요)  |
| 컨테이너 유지     | 종료 후 유지               | 종료 후 삭제 (`--rm` 시)       |
| 의존 서비스 시작  | 자동 시작                  | 자동 시작 (무시 가능)           |

---

### 기본 사용법

```bash
docker compose run [옵션] <서비스명> [명령어]
```

---

### Ollama 명령어 일회성 실행

```bash
# CPU 서비스로 일회성 모델 pull
docker compose --profile cpu run --rm ollama-cpu ollama pull llama3.2:3b

# GPU 서비스로 일회성 모델 pull
docker compose --profile nvidia run --rm ollama-nvidia ollama pull llama3.2:3b

# 설치된 모델 목록 확인
docker compose --profile cpu run --rm ollama-cpu ollama list

# 특정 모델로 대화 시작 (인터랙티브)
docker compose --profile cpu run --rm -it ollama-cpu ollama run llama3.2:3b
```

---

### 쉘 접속 (디버깅용)

```bash
# Ollama 컨테이너 bash 접속
docker compose --profile cpu run --rm -it ollama-cpu bash

# Open-WebUI 컨테이너 bash 접속
docker compose --profile webui run --rm -it open-webui bash

# JupyterLab 컨테이너 bash 접속
docker compose --profile jupyter run --rm -it jupyter bash
```

---

### 환경변수 지정하여 실행

```bash
# 환경변수를 직접 전달
docker compose --profile cpu run --rm -e OLLAMA_NUM_PARALLEL=4 ollama-cpu ollama list

# .env 파일과 다른 값을 임시로 덮어쓰기
docker compose --profile cpu run --rm -e OLLAMA_KEEP_ALIVE=30m ollama-cpu bash
```

---

### 포트 바인딩 포함 실행

> `run`은 기본적으로 포트를 바인딩하지 않으므로 필요 시 `-p` 옵션을 사용합니다.

```bash
docker compose --profile cpu run --rm -p 11434:11434 ollama-cpu
```

---

### 의존 서비스 시작 없이 실행

```bash
# --no-deps: 의존 서비스(depends_on)를 시작하지 않음
docker compose --profile webui run --rm --no-deps open-webui bash
```

---

### 주요 옵션 정리

| 옵션              | 설명                                      |
|-----------------|-------------------------------------------|
| `--rm`          | 컨테이너 종료 후 자동 삭제                  |
| `-it`           | 인터랙티브 터미널 모드                      |
| `-d`            | 백그라운드 실행                            |
| `-e KEY=VALUE`  | 환경변수 전달                              |
| `-p HOST:CONT`  | 포트 바인딩                               |
| `--no-deps`     | 의존 서비스 시작 안 함                      |
| `--entrypoint`  | 기본 entrypoint 덮어쓰기                   |
| `-u USER`       | 실행 사용자 지정                           |
| `-w /path`      | 작업 디렉터리 지정                         |

---

## 🤖 Ollama 모델 관리

### 컨테이너 내에서 모델 Pull
```bash
docker exec -it ollama-cpu ollama pull llama3.2:3b
docker exec -it ollama-cpu ollama pull nomic-embed-text
```

### 설치된 모델 목록 확인
```bash
docker exec -it ollama-cpu ollama list
```

### 모델 실행 (대화 테스트)
```bash
docker exec -it ollama-cpu ollama run llama3.2:3b
```

### 모델 삭제
```bash
docker exec -it ollama-cpu ollama rm llama3.2:3b
```

---

## 🌐 서비스 접속 URL

| 서비스        | 기본 URL                    | 환경변수               |
|-------------|----------------------------|----------------------|
| Ollama API  | http://localhost:11434      | `OLLAMA_PORT=11434`  |
| Open-WebUI  | http://localhost:3000       | `WEBUI_PORT=3000`    |
| JupyterLab  | http://localhost:8888       | `JUPYTER_PORT=8888`  |

---

## ⚙️ 환경변수 설정

`.env.example`을 복사하여 `.env` 파일을 생성하고 원하는 값으로 수정합니다.

```bash
cp .env.example .env
```

| 변수명                    | 기본값                 | 설명                          |
|--------------------------|----------------------|-------------------------------|
| `OLLAMA_PORT`            | `11434`              | Ollama API 포트                |
| `OLLAMA_KEEP_ALIVE`      | `5m`                 | 모델 메모리 유지 시간             |
| `OLLAMA_MAX_LOADED_MODELS` | `2`               | 동시에 메모리에 올릴 최대 모델 수  |
| `OLLAMA_NUM_PARALLEL`    | `2`                  | 병렬 요청 처리 수                |
| `DEFAULT_MODEL`          | `llama3.2:3b`        | 자동 Pull 기본 모델             |
| `EMBED_MODEL`            | `nomic-embed-text`   | 자동 Pull 임베딩 모델           |
| `WEBUI_PORT`             | `3000`               | Open-WebUI 포트               |
| `WEBUI_SECRET_KEY`       | `changeme-secret-key`| Open-WebUI 시크릿 키           |
| `JUPYTER_PORT`           | `8888`               | JupyterLab 포트               |

---

## 💡 사전 조건

- **NVIDIA GPU 사용 시**: `nvidia-container-toolkit` 설치 필요
  ```bash
  # Ubuntu 기준
  sudo apt-get install -y nvidia-container-toolkit
  sudo systemctl restart docker
  ```

- **GPU 동작 확인**:
  ```bash
  docker run --rm --gpus all nvidia/cuda:11.8.0-base-ubuntu22.04 nvidia-smi
  ```

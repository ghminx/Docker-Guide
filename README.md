# Docker를 활용한 PyTorch 프로젝트 배포 가이드

이 문서는 PyTorch 기반 프로젝트를 Docker 컨테이너로 개발하고 배포하는 전체 과정을 설명합니다.

---

## 1. 프로젝트 개요

### 목표:
- PyTorch 모델 학습 및 예측 기능과 Streamlit을 이용한 웹 인터페이스 구축

### 주요 파일:
- `train.py`, `predict.py`, `app.py` (각각 학습, 예측, 웹 앱 실행 코드)
- `model.pth` (학습된 모델 가중치)

### 로컬 작업 폴더:
- 예: `C:\project`

---

## 2. 개발 환경 설정

### 2.1 베이스 이미지 선택
Docker Hub에서 PyTorch 공식 이미지를 사용합니다.

#### Dockerfile 예시:
```dockerfile
FROM pytorch/pytorch:latest
```
> **참고:**
> - `latest` 태그는 Docker Hub에서 최신 버전을 의미하지만, 한 번 빌드한 이미지는 그 시점의 상태를 고정합니다.
> - 배포 시에는 특정 버전을 명시하는 것이 안정적입니다. 예: `FROM pytorch/pytorch:1.12.1-cuda11.3`

### 2.2 로컬 작업 폴더 생성
- Windows에서 예: `C:\project`
- 이 폴더에 소스 코드와 관련 파일들을 저장합니다.

---

## 3. 컨테이너와 로컬 폴더 마운트하여 개발

### 3.1 컨테이너 실행 및 마운트
로컬 폴더를 컨테이너 내부의 작업 디렉토리로 마운트하여 실시간으로 코드를 수정하고 테스트할 수 있습니다.

#### 예시 명령어:
```bash
docker run -it --rm --gpus all -v C:/test:/app pytorch/pytorch:latest bash

```
> **설명:**
> - `-v C:/project:/app`는 로컬 `C:\project` 폴더를 컨테이너 내부의 `/app` 디렉토리에 연결합니다.
> - `--gpus all` 은 docker container 내부에서 gpu를 사용할수있게 합니다
> - `--rm` 은 container가 종료되면 자동으로 삭제합니다
> - 이 상태에서 컨테이너 내에서 `train.py`, `predict.py`, `app.py` 등을 수정하여 작업할 수 있습니다.

---

## 4. 배포 준비

### 4.1 `requirements.txt` 생성
프로젝트에서 추가로 설치한 라이브러리 목록을 저장하여, 동일한 환경을 재현할 수 있도록 합니다.

#### 명령어 (컨테이너 또는 로컬에서 실행):
```bash
pip freeze > requirements.txt
```
#### 예시 내용:
```
torch
torchvision
torchaudio
streamlit
numpy
```

### 4.2 Dockerfile 작성
`C:\project` 폴더 내에 `Dockerfile`을 생성합니다.

#### Dockerfile 예시:
```dockerfile
# PyTorch 공식 이미지를 기반으로 사용 (필요한 경우 특정 버전 지정)
FROM pytorch/pytorch:latest

# 작업 디렉토리 설정
WORKDIR /app

# 로컬의 모든 파일을 컨테이너 내부로 복사
COPY . /app

# 필요한 패키지 설치 (requirements.txt에 기록된 버전대로 설치)
RUN pip install --no-cache-dir -r requirements.txt

# Streamlit 앱 실행 (컨테이너가 시작되면 웹 앱이 실행됨)
CMD ["streamlit", "run", "app.py", "--server.port=8501", "--server.address=0.0.0.0"]
```

### 4.3 `.dockerignore` 파일 (선택 사항)
불필요한 파일이나 디렉토리를 이미지에 포함시키지 않도록 `.dockerignore` 파일을 생성합니다.

#### 예시 내용:
```
__pycache__/
*.pyc
*.pyo
.DS_Store
venv/
```

---

## 5. Docker 이미지 빌드 및 실행

### 5.1 이미지 빌드
#### 명령어 (C:\project 폴더 내에서 실행):
```bash
docker build -t my_pytorch_webapp .
```
> **설명:**
> - `-t my_pytorch_webapp`은 새 이미지를 `my_pytorch_webapp`이라는 이름으로 태그합니다.
> - `.`은 현재 디렉토리의 Dockerfile을 사용함을 의미합니다.

### 5.2 컨테이너 실행
#### 명령어:
```bash
docker run -d -p 8501:8501 my_pytorch_webapp
```
> **설명:**
> - `-d`는 컨테이너를 백그라운드에서 실행합니다.
> - `-p 8501:8501`은 컨테이너 내부의 포트 8501을 로컬의 포트 8501과 매핑하여, 웹 브라우저에서 접근할 수 있도록 합니다.

#### 접속 확인:
웹 브라우저에서 [http://localhost:8501](http://localhost:8501) 에 접속하여 Streamlit 웹 앱을 확인합니다.

---

## 6. 배포 (Docker Hub 및 클라우드)

### 6.1 Docker Hub 업로드
#### 이미지 태그 변경 후 푸시:
```bash
docker tag my_pytorch_webapp your_dockerhub_username/my_pytorch_webapp
docker push your_dockerhub_username/my_pytorch_webapp
```
#### 이후 다른 환경에서 다음 명령어로 실행:
```bash
docker pull your_dockerhub_username/my_pytorch_webapp
docker run -d -p 8501:8501 your_dockerhub_username/my_pytorch_webapp
```

### 6.2 클라우드 배포 (선택 사항)
AWS EC2, Google Cloud Run, Azure Container Apps 등에서 Docker 컨테이너를 실행하여 서비스를 배포할 수 있습니다.

---

## 7. 전체 프로세스 요약

1. **베이스 이미지 선택:**  
   - Docker Hub의 `pytorch/pytorch` 이미지를 사용 (예: `FROM pytorch/pytorch:latest`)

2. **로컬 작업 폴더 생성:**  
   - 예: `C:\project`

3. **개발:**  
   - `docker run -it --rm -v C:/project:/app pytorch/pytorch:latest bash`
   - 컨테이너 내부에서 `train.py`, `predict.py`, `app.py` 등 작업

4. **배포 준비:**  
   - 로컬에서 `pip freeze > requirements.txt`로 패키지 목록 생성
   - `Dockerfile` 작성 (베이스 이미지, 작업 디렉토리 설정, 파일 복사, 패키지 설치, 실행 명령어 포함)

5. **이미지 빌드 및 실행:**  
   - `docker build -t my_pytorch_webapp .`
   - `docker run -d -p 8501:8501 my_pytorch_webapp`

6. **배포:**  
   - Docker Hub에 이미지 업로드하거나 클라우드 플랫폼에서 실행


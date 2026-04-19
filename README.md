# 🐳 Kali Linux에서 Kubernetes(Minikube) 설치 완전 가이드

> **대상 환경**: Kali GNU/Linux 2026.1 Rolling (커널 6.18, x86_64)  
> **사용 버전**: kubectl v1.35.4 · minikube v1.38.1 · Docker 28.5.2  
> **난이도**: 입문자 친화적 (모든 명령어 복사-붙여넣기 가능)

---

## 📋 목차

1. [사전 요구사항 확인](#1-사전-요구사항-확인)
2. [Docker 컨테이너 런타임 설정](#2-docker-컨테이너-런타임-설정)
3. [kubectl 설치](#3-kubectl-설치)
4. [Minikube 설치](#4-minikube-설치)
5. [클러스터 시작 및 검증](#5-클러스터-시작-및-검증)
6. [샘플 앱 배포 및 노출](#6-샘플-앱-배포-및-노출)
7. [기본 kubectl 명령어 레퍼런스](#7-기본-kubectl-명령어-레퍼런스)
8. [클러스터 종료 및 삭제](#8-클러스터-종료-및-삭제)
9. [문제 해결 (Troubleshooting)](#9-문제-해결-troubleshooting)

---

## 1. 사전 요구사항 확인

### 최소 시스템 요구사항

| 항목 | 최소 요구사항 | 확인 명령어 |
|------|------------|------------|
| CPU | 2코어 이상 | `nproc` |
| RAM | 2GB 이상 | `free -h` |
| 디스크 | 20GB 이상 | `df -h /` |
| OS | 64비트 Linux | `uname -m` |
| 가상화 지원 | KVM 또는 Docker | `grep -c vmx /proc/cpuinfo` |

### 현재 시스템 정보 한 번에 확인

```bash
echo "=== 시스템 정보 ===" && \
  uname -a && \
  echo "" && \
  echo "=== OS 버전 ===" && \
  cat /etc/os-release | grep -E "PRETTY|VERSION" && \
  echo "" && \
  echo "=== CPU / RAM ===" && \
  echo "CPU 코어: $(nproc)" && \
  free -h | head -2 && \
  echo "" && \
  echo "=== 디스크 ===" && \
  df -h / | tail -1
```

> [!NOTE]
> Kali 2026.1 환경에서는 CPU 4코어, RAM 5.6GB로 충분합니다.

---

## 2. Docker 컨테이너 런타임 설정

Minikube는 Docker를 컨테이너 런타임으로 사용합니다.

### 2-1. Docker 설치 확인

```bash
docker --version
# 예상 출력: Docker version 28.5.2+dfsg3, build ...
```

Docker가 이미 설치된 경우 2-2 단계로 바로 이동하세요.

### 2-2. Docker 미설치 시 (신규 설치)

```bash
# 기존 충돌 패키지 제거
sudo apt remove -y docker docker-engine docker.io containerd runc 2>/dev/null

# 저장소 키 및 소스 추가
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg \
  | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/debian bookworm stable" \
  | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Docker 설치
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

### 2-3. Docker 서비스 시작 및 현재 사용자 추가

```bash
# Docker 서비스 시작 및 부팅 시 자동 시작 설정
sudo systemctl enable --now docker

# sudo 없이 Docker 사용하도록 현재 사용자를 docker 그룹에 추가
sudo usermod -aG docker $USER

# 그룹 변경 즉시 적용 (재로그인 없이)
newgrp docker
```

### 2-4. Docker 정상 동작 확인

```bash
docker run --rm hello-world
```

아래와 같은 메시지가 출력되면 정상입니다:
```
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

---

## 3. kubectl 설치

`kubectl`은 Kubernetes 클러스터를 제어하는 CLI 도구입니다.

### 3-1. 바이너리 다운로드 및 설치

```bash
# 최신 안정 버전 확인
KUBECTL_VER=$(curl -sSL https://dl.k8s.io/release/stable.txt)
echo "설치할 버전: $KUBECTL_VER"

# 바이너리 다운로드
curl -LO "https://dl.k8s.io/release/${KUBECTL_VER}/bin/linux/amd64/kubectl"

# 체크섬 검증 (선택, 권장)
curl -LO "https://dl.k8s.io/release/${KUBECTL_VER}/bin/linux/amd64/kubectl.sha256"
echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check
# 출력: kubectl: OK

# 실행 권한 부여 및 시스템 경로 이동
chmod +x kubectl
sudo mv kubectl /usr/local/bin/kubectl
sudo chown root:root /usr/local/bin/kubectl
```

### 3-2. 설치 확인

```bash
kubectl version --client
```

```
Client Version: v1.35.4
Kustomize Version: v5.x.x
```

### 3-3. 자동완성 설정 (선택)

```bash
# Bash 자동완성
echo 'source <(kubectl completion bash)' >> ~/.bashrc
echo 'alias k=kubectl' >> ~/.bashrc
echo 'complete -o default -F __start_kubectl k' >> ~/.bashrc
source ~/.bashrc
```

---

## 4. Minikube 설치

`minikube`는 로컬 단일 노드 Kubernetes 클러스터를 실행하는 도구입니다.

### 4-1. 바이너리 다운로드 및 설치

```bash
# 최신 버전 다운로드 (현재: v1.38.1)
curl -LO https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64

# 실행 권한 부여 및 시스템 경로 이동
chmod +x minikube-linux-amd64
sudo mv minikube-linux-amd64 /usr/local/bin/minikube
sudo chown root:root /usr/local/bin/minikube
```

### 4-2. 설치 확인

```bash
minikube version
```

```
minikube version: v1.38.1
commit: xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### 4-3. 선택 사항: conntrack 설치

일부 네트워크 기능에 필요한 패키지입니다:

```bash
sudo apt install -y conntrack socat
```

---

## 5. 클러스터 시작 및 검증

### 5-1. Minikube 클러스터 시작

```bash
# Docker 드라이버로 클러스터 시작 (권장)
minikube start --driver=docker --cpus=2 --memory=2048

# root 사용자인 경우 (Kali에서 root로 실행 시)
minikube start --driver=docker --cpus=2 --memory=2048 --force
```

> [!IMPORTANT]
> `--force` 플래그는 root 사용자 경고를 무시합니다. Kali에서 root로 작업 중이라면 필요합니다.

성공 출력 예시:
```
😄  minikube v1.38.1 on Kali 2026.1
✨  Using the docker driver based on user choice
📌  Using Docker driver with root privileges
👍  Starting "minikube" primary control-plane node in "minikube" cluster
🚜  Pulling base image v0.0.47 ...
💾  Downloading Kubernetes v1.35.x preload ...
🔥  Creating docker container (CPUs=2, Memory=2048MB) ...
🐳  Preparing Kubernetes v1.35.x on Docker 28.5.2 ...
🔎  Verifying Kubernetes components...
🌟  Enabled addons: storage-provisioner, default-storageclass
🏄  Done! kubectl is now configured to use "minikube" cluster
```

### 5-2. 클러스터 상태 확인

```bash
# Minikube 상태 확인
minikube status
```

```
minikube
type: Control Plane
host: Running
kubelet: Running
apiserver: Running
kubeconfig: Configured
```

```bash
# 노드 확인
kubectl get nodes
```

```
NAME       STATUS   ROLES           AGE   VERSION
minikube   Ready    control-plane   2m    v1.35.4
```

```bash
# 클러스터 전체 시스템 파드 확인
kubectl get pods -n kube-system
```

```
NAME                               READY   STATUS    RESTARTS   AGE
coredns-xxx                        1/1     Running   0          2m
etcd-minikube                      1/1     Running   0          2m
kube-apiserver-minikube            1/1     Running   0          2m
kube-controller-manager-minikube   1/1     Running   0          2m
kube-proxy-xxx                     1/1     Running   0          2m
kube-scheduler-minikube            1/1     Running   0          2m
storage-provisioner                1/1     Running   0          2m
```

모든 파드가 `Running` 상태면 클러스터가 정상 작동 중입니다. ✅

---

## 6. 샘플 앱 배포 및 노출

nginx 웹 서버를 배포하여 클러스터 기능을 검증합니다.

### 6-1. nginx 디플로이먼트 생성

```bash
kubectl create deployment nginx-demo --image=nginx:latest --replicas=2
```

### 6-2. 배포 상태 확인

```bash
# 배포 확인
kubectl get deployments

# 파드 확인 (-w 플래그로 실시간 모니터링)
kubectl get pods -w
```

```
NAME                          READY   STATUS    RESTARTS   AGE
nginx-demo-xxxxxxxxx-xxxxx   1/1     Running   0          30s
nginx-demo-xxxxxxxxx-yyyyy   1/1     Running   0          30s
```

`Running` 상태가 되면 `Ctrl+C`로 모니터링 종료.

### 6-3. 서비스로 노출 (NodePort)

```bash
kubectl expose deployment nginx-demo --type=NodePort --port=80
```

### 6-4. 서비스 URL 확인 및 접속

```bash
# 서비스 목록 확인
kubectl get services

# minikube가 자동으로 서비스 URL 제공
minikube service nginx-demo --url
```

```
http://192.168.49.2:xxxxx
```

```bash
# URL로 접속 테스트 (curl)
curl $(minikube service nginx-demo --url)
```

```html
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
...
```

nginx 기본 페이지가 반환되면 ✅ 클러스터가 완전히 작동 중입니다!

### 6-5. 브라우저에서 열기 (GUI 환경)

```bash
minikube service nginx-demo
```

---

## 7. 기본 kubectl 명령어 레퍼런스

### 파드 관련

```bash
# 파드 목록
kubectl get pods
kubectl get pods -A              # 모든 네임스페이스

# 파드 상세 정보
kubectl describe pod <파드이름>

# 파드 로그 확인
kubectl logs <파드이름>
kubectl logs -f <파드이름>       # 실시간 스트리밍

# 파드 내부 쉘 접속
kubectl exec -it <파드이름> -- /bin/bash
```

### 디플로이먼트 관련

```bash
# 디플로이먼트 목록
kubectl get deployments

# 레플리카 수 조정 (스케일링)
kubectl scale deployment nginx-demo --replicas=3

# 롤링 업데이트
kubectl set image deployment/nginx-demo nginx=nginx:1.27

# 업데이트 상태 확인
kubectl rollout status deployment/nginx-demo

# 이전 버전으로 롤백
kubectl rollout undo deployment/nginx-demo
```

### 서비스 관련

```bash
kubectl get services
kubectl describe service nginx-demo
kubectl delete service nginx-demo
```

### 네임스페이스

```bash
kubectl get namespaces
kubectl create namespace my-app
kubectl get pods -n my-app
```

### 리소스 조회

```bash
# 파드, 서비스, 디플로이먼트 한 번에
kubectl get all

# 특정 네임스페이스
kubectl get all -n kube-system
```

---

## 8. 클러스터 종료 및 삭제

```bash
# 샘플 앱 정리
kubectl delete service nginx-demo
kubectl delete deployment nginx-demo

# 클러스터 일시 중지 (데이터 보존)
minikube stop

# 클러스터 재시작
minikube start

# 클러스터 완전 삭제 (초기화)
minikube delete
```

---

## 9. 문제 해결 (Troubleshooting)

### ❶ minikube start 시 Docker 권한 오류

```
Error: ... permission denied while trying to connect to Docker daemon
```

**해결:**
```bash
sudo usermod -aG docker $USER
newgrp docker
minikube start --driver=docker
```

### ❷ root 사용자에서 실행 거부

```
Error: ... must be run as non-root user
```

**해결:**
```bash
minikube start --driver=docker --force
```

### ❸ 파드가 Pending 상태에서 멈춤

```bash
kubectl describe pod <파드이름>
# Events 섹션에서 원인 확인
```

보통 리소스 부족이 원인입니다. `--memory` 값을 늘려보세요:
```bash
minikube delete && minikube start --driver=docker --memory=3072
```

### ❹ minikube 애드온 활성화

```bash
# 유용한 애드온 목록
minikube addons list

# Dashboard 활성화 (웹 UI)
minikube addons enable dashboard
minikube dashboard

# Metrics Server 활성화 (kubectl top 명령어)
minikube addons enable metrics-server
kubectl top nodes
```

### ❺ 클러스터 로그 확인

```bash
minikube logs
kubectl cluster-info dump
```

---

## ✅ 설치 완료 체크리스트

- [ ] Docker 설치 및 서비스 실행 확인: `docker run hello-world`
- [ ] kubectl 설치 확인: `kubectl version --client`
- [ ] minikube 설치 확인: `minikube version`
- [ ] 클러스터 시작: `minikube start --driver=docker`
- [ ] 노드 Ready 상태 확인: `kubectl get nodes`
- [ ] 샘플 앱 배포 및 접속 확인: `curl $(minikube service nginx-demo --url)`

---

*가이드 작성일: 2026-04-18 | 환경: Kali GNU/Linux 2026.1 Rolling | kubectl v1.35.4 | minikube v1.38.1*

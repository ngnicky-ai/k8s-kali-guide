# ☸️ Kubernetes 활용 예제 모음

> **실습 환경**: Kali Linux 2026.1 | minikube v1.38.1 | Kubernetes v1.35.1 | kubectl v1.35.4  
> **실습일**: 2026-04-19

---

## 예제 1: Deployment & 파드 스케일링

**개념**: Deployment로 앱을 배포하고, `scale` 명령으로 파드 수를 동적으로 조정합니다.

### 명령어

```bash
# Deployment 생성 (초기 레플리카 2개)
kubectl create deployment nginx-demo --image=nginx:1.25 --replicas=2

# NodePort 서비스로 외부 노출
kubectl expose deployment nginx-demo --type=NodePort --port=80

# 상태 확인
kubectl get deployments nginx-demo
kubectl get pods -l app=nginx-demo
kubectl get service nginx-demo

# 파드 4개로 스케일 아웃
kubectl scale deployment nginx-demo --replicas=4
kubectl get pods -l app=nginx-demo
```

### 실행 결과

```
NAME         READY   UP-TO-DATE   AVAILABLE   AGE
nginx-demo   4/4     4            4           30s

NAME                          READY   STATUS    RESTARTS   AGE
nginx-demo-5fd88d56df-4jjrk   1/1     Running   0          30s
nginx-demo-5fd88d56df-fllk5   1/1     Running   0          22s
nginx-demo-5fd88d56df-h9zxk   1/1     Running   0          22s
nginx-demo-5fd88d56df-qrgdc   1/1     Running   0          30s

NAME         TYPE       CLUSTER-IP    EXTERNAL-IP   PORT(S)        AGE
nginx-demo   NodePort   10.99.91.26   <none>        80:31487/TCP   30s
```

> 💡 `kubectl scale`은 무중단으로 파드 수를 조정합니다. NodePort는 `<노드 IP>:<포트>`로 접속합니다.

---

## 예제 2: 롤링 업데이트 & 롤백

**개념**: 서비스 중단 없이 이미지를 업데이트하고, 문제 발생 시 이전 버전으로 롤백합니다.

### 명령어

```bash
# 롤링 업데이트: nginx:1.25 → nginx:1.27
kubectl set image deployment/nginx-demo nginx=nginx:1.27

# 업데이트 진행 상태 모니터링
kubectl rollout status deployment/nginx-demo

# 업데이트 히스토리 확인
kubectl rollout history deployment/nginx-demo

# 이전 버전으로 롤백
kubectl rollout undo deployment/nginx-demo
kubectl rollout status deployment/nginx-demo
```

### 실행 결과

```
# 업데이트 진행
Waiting for deployment "nginx-demo" rollout to finish: 2 out of 4 new replicas have been updated...
Waiting for deployment "nginx-demo" rollout to finish: 3 out of 4 new replicas have been updated...
deployment "nginx-demo" successfully rolled out

# 히스토리
REVISION  CHANGE-CAUSE
1         <none>
2         <none>   ← nginx:1.27 업데이트

# 롤백 후 이미지
nginx:1.25
```

> 💡 `--record` 플래그를 사용하면 CHANGE-CAUSE에 명령어가 기록됩니다.  
> 특정 리비전으로 돌아갈 때: `kubectl rollout undo deployment/nginx-demo --to-revision=1`

---

## 예제 3: ConfigMap & Secret 환경변수 주입

**개념**: 설정값과 민감 정보를 애플리케이션 코드 외부에서 관리하고 파드에 주입합니다.

### YAML 매니페스트

```yaml
# ConfigMap: 일반 설정값
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_ENV: "production"
  APP_PORT: "8080"
  LOG_LEVEL: "info"
---
# Secret: 민감 정보 (base64 저장)
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:
  DB_PASSWORD: "s3cur3P@ss!"
  API_KEY: "my-api-key-12345"
---
# Pod: ConfigMap & Secret을 환경변수로 주입
apiVersion: v1
kind: Pod
metadata:
  name: config-demo
spec:
  containers:
  - name: demo
    image: busybox:1.36
    command: ["sh", "-c", "echo ENV=$APP_ENV PORT=$APP_PORT LOG=$LOG_LEVEL PASS=$DB_PASSWORD && sleep 3600"]
    envFrom:
    - configMapRef:
        name: app-config      # ConfigMap 전체 주입
    env:
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secret    # Secret에서 특정 키만 주입
          key: DB_PASSWORD
```

### 적용 및 확인

```bash
kubectl apply -f config-demo.yaml
kubectl logs config-demo
```

### 실행 결과

```
ENV=production PORT=8080 LOG=info PASS=s3cur3P@ss!
```

> 💡 Secret은 `kubectl get secret app-secret -o jsonpath='{.data.DB_PASSWORD}' | base64 -d`로 값을 확인할 수 있습니다.

---

## 예제 4: 리소스 요청 & 제한 (Requests & Limits)

**개념**: CPU/메모리 자원을 명시적으로 설정해 클러스터 리소스 과점유를 방지합니다.

### YAML 매니페스트

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-limited
spec:
  containers:
  - name: stress
    image: nginx:1.25
    resources:
      requests:           # 스케줄러가 노드 배정 시 기준
        memory: "64Mi"
        cpu: "100m"       # 100m = 0.1 CPU 코어
      limits:             # 초과 시 스로틀링(CPU) or OOMKill(메모리)
        memory: "128Mi"
        cpu: "200m"
```

### 적용 및 확인

```bash
kubectl apply -f resource-limited.yaml
kubectl describe pod resource-limited | grep -E "Requests:|Limits:|cpu:|memory:"
```

### 실행 결과

```
    Limits:
      cpu:     200m
      memory:  128Mi
    Requests:
      cpu:        100m
      memory:     64Mi
```

> 💡 `kubectl top pods` (metrics-server 활성화 후)로 실제 사용량을 확인할 수 있습니다.  
> `minikube addons enable metrics-server`

---

## 예제 5: Liveness & Readiness Probe

**개념**: Kubernetes가 파드 상태를 주기적으로 검사하여 비정상 파드를 재시작하거나 트래픽에서 제외합니다.

### YAML 매니페스트

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: probe-demo
spec:
  containers:
  - name: nginx
    image: nginx:1.25
    ports:
    - containerPort: 80
    livenessProbe:       # 파드가 살아있는지 확인 → 실패 시 재시작
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 5    # 시작 후 5초 후 첫 체크
      periodSeconds: 10          # 10초마다 체크
      failureThreshold: 3        # 3번 연속 실패 시 재시작
    readinessProbe:      # 트래픽 수신 가능한지 확인 → 실패 시 서비스에서 제외
      httpGet:
        path: /
        port: 80
      initialDelaySeconds: 3
      periodSeconds: 5
```

### 적용 및 확인

```bash
kubectl apply -f probe-demo.yaml
kubectl describe pod probe-demo | grep -A4 "Liveness\|Readiness"
```

### 실행 결과

```
    Liveness:   http-get http://:80/ delay=5s timeout=1s period=10s #success=1 #failure=3
    Readiness:  http-get http://:80/ delay=3s timeout=1s period=5s #success=1 #failure=3
```

| Probe 타입 | 실패 시 동작 | 용도 |
|-----------|------------|------|
| Liveness | 컨테이너 재시작 | 데드락, 무한 루프 등 복구 |
| Readiness | 서비스 엔드포인트에서 제거 | 초기화 중 트래픽 차단 |
| Startup | Liveness 비활성화 | 느린 시작 앱 보호 |

---

## 예제 6: Namespace로 환경 분리

**개념**: 네임스페이스로 개발/운영 환경을 논리적으로 분리하여 자원과 권한을 격리합니다.

### 명령어

```bash
# 네임스페이스 생성
kubectl create namespace dev
kubectl create namespace prod

# 각 네임스페이스에 별도 배포
kubectl create deployment nginx-dev  --image=nginx:1.25 --replicas=1 -n dev
kubectl create deployment nginx-prod --image=nginx:1.25 --replicas=2 -n prod

# 네임스페이스별 파드 확인
kubectl get pods -n dev
kubectl get pods -n prod

# 전체 네임스페이스 배포 현황
kubectl get deployments -A | grep nginx
```

### 실행 결과

```
=== dev ===
NAME                        READY   STATUS    RESTARTS   AGE
nginx-dev-f8595479f-kfj6c   1/1     Running   0          10s

=== prod ===
NAME                         READY   STATUS    RESTARTS   AGE
nginx-prod-85cf5584c-q5dxm   1/1     Running   0          10s
nginx-prod-85cf5584c-xgpfh   1/1     Running   0          10s

=== 전체 배포 목록 ===
NAMESPACE   NAME         READY   UP-TO-DATE   AVAILABLE
default     nginx-demo   4/4     4            4
dev         nginx-dev    1/1     1            1
prod        nginx-prod   2/2     2            2
```

> 💡 `kubectl config set-context --current --namespace=dev`로 기본 네임스페이스를 변경할 수 있습니다.

---

## 예제 7: PersistentVolume & PersistentVolumeClaim

**개념**: 파드가 삭제되어도 데이터가 유지되는 영구 스토리지를 파드에 마운트합니다.

### YAML 매니페스트

```yaml
# PVC: 스토리지 요청
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: demo-pvc
spec:
  accessModes:
    - ReadWriteOnce    # 단일 노드 읽기/쓰기
  resources:
    requests:
      storage: 256Mi
---
# Pod: PVC를 /data 경로에 마운트
apiVersion: v1
kind: Pod
metadata:
  name: pvc-demo
spec:
  containers:
  - name: writer
    image: busybox:1.36
    command: ["sh", "-c", "echo 'Hello from PVC!' > /data/hello.txt && cat /data/hello.txt && sleep 3600"]
    volumeMounts:
    - name: my-storage
      mountPath: /data
  volumes:
  - name: my-storage
    persistentVolumeClaim:
      claimName: demo-pvc
```

### 적용 및 확인

```bash
kubectl apply -f pvc-demo.yaml
kubectl get pvc demo-pvc
kubectl logs pvc-demo
```

### 실행 결과

```
NAME       STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
demo-pvc   Bound    pvc-aa9e4828-624d-4005-a71e-8005b4a9e1b1  256Mi      RWO            standard       10s

# 파드 로그
Hello from PVC!
```

| AccessMode | 설명 |
|-----------|------|
| ReadWriteOnce (RWO) | 단일 노드 읽기/쓰기 |
| ReadOnlyMany (ROX) | 여러 노드 읽기 전용 |
| ReadWriteMany (RWX) | 여러 노드 읽기/쓰기 |

---

## 예제 8: CronJob (정기 작업 스케줄링)

**개념**: cron 표현식으로 정기적인 배치 작업을 Kubernetes 클러스터에서 자동 실행합니다.

### YAML 매니페스트

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello-cron
spec:
  schedule: "*/1 * * * *"    # 1분마다 실행 (cron 표현식)
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: hello
            image: busybox:1.36
            command: ["sh", "-c", "echo Job ran at: $(date)"]
          restartPolicy: OnFailure
```

### 적용 및 확인

```bash
kubectl apply -f hello-cron.yaml

# CronJob 상태 확인
kubectl get cronjob hello-cron

# 실행된 Job 목록
kubectl get jobs

# Job 파드 로그 확인
kubectl logs $(kubectl get pods | grep hello-cron | head -1 | awk '{print $1}')
```

### 실행 결과

```
# CronJob 상태
NAME         SCHEDULE      TIMEZONE   SUSPEND   ACTIVE   LAST SCHEDULE   AGE
hello-cron   */1 * * * *   <none>     False     0        39s             50s

# Job 실행 결과
NAME                  STATUS     COMPLETIONS   DURATION   AGE
hello-cron-29609427   Complete   1/1           3s         39s

# 파드 로그
Job ran at: Sun Apr 19 02:27:00 UTC 2026
```

### Cron 표현식 참고

```
┌───── 분 (0-59)
│ ┌───── 시 (0-23)
│ │ ┌───── 일 (1-31)
│ │ │ ┌───── 월 (1-12)
│ │ │ │ ┌───── 요일 (0-7, 0=일요일)
│ │ │ │ │
* * * * *

예시:
0 9 * * 1-5   → 평일 오전 9시
0 0 1 * *     → 매월 1일 자정
*/5 * * * *   → 5분마다
```

---

## 📊 실습 요약

| # | 예제 | 핵심 리소스 | 결과 |
|---|------|-----------|------|
| 1 | Deployment & 스케일링 | Deployment, Service | 4파드 Running ✅ |
| 2 | 롤링 업데이트 & 롤백 | Deployment | nginx 1.25↔1.27 무중단 교체 ✅ |
| 3 | ConfigMap & Secret | ConfigMap, Secret, Pod | ENV 주입 확인 ✅ |
| 4 | 리소스 제한 | Pod (resources) | CPU/메모리 제한 설정 ✅ |
| 5 | Liveness & Readiness Probe | Pod (probes) | HTTP 헬스체크 동작 ✅ |
| 6 | Namespace 분리 | Namespace, Deployment | dev/prod 환경 독립 ✅ |
| 7 | PersistentVolumeClaim | PVC, Pod | 영구 스토리지 파일 R/W ✅ |
| 8 | CronJob | CronJob, Job | 1분 주기 자동 실행 ✅ |

---

## 🔧 현재 실행 중인 리소스 정리 명령어

```bash
# 예제 리소스 전체 삭제 (클러스터는 유지)
kubectl delete deployment nginx-demo
kubectl delete service nginx-demo
kubectl delete pod config-demo resource-limited probe-demo pvc-demo
kubectl delete configmap app-config
kubectl delete secret app-secret
kubectl delete cronjob hello-cron
kubectl delete pvc demo-pvc
kubectl delete namespace dev prod
```

---

*실습 환경: Kali Linux 2026.1 | minikube v1.38.1 | K8s v1.35.1 | 실습일: 2026-04-19*

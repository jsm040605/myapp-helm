# myapp-helm

ASBG 컨테이너 과정 1주차 실습

## 1. 프로젝트 구조

```
myapp-helm/
├── app.py                  # Flask WAS (/, /healthz 엔드포인트)
├── Dockerfile
├── requirements.txt
├── values-dev.yaml         # 개발 환경용 values override
└── charts/
    └── myapp/
        ├── Chart.yaml
        ├── values.yaml
        └── templates/
            ├── deployment.yaml
            ├── service.yaml
            └── ingress.yaml
```

## 2. WAS 개요

- Python Flask 기반 서버 (`app.py`)
- `/` : 간단한 헬로 메시지 응답
- `/healthz` : 헬스체크용 엔드포인트, Deployment의 `readinessProbe` / `livenessProbe`에서 사용
- 컨테이너 포트: `8080`

## 3. 사전 준비 (minikube)

```bash
minikube start --cpus=2 --memory=4096
minikube addons enable ingress
kubectl -n ingress-nginx get pods -w   # controller가 Running 될 때까지 대기
kubectl get ingressclass               # nginx IngressClass 확인
```

## 4. 이미지 빌드
```
#로컬에서 빌드 후 로드
docker build -t myapp:v1 .
minikube image load myapp:v1
```

레지스트리 조회를 막기 위해 `values.yaml`의 `image.pullPolicy`는 `IfNotPresent`로 고정되어 있습니다.

## 5. 설치 방법

설치 방법
```
# 1. Deployment + Service 설치
helm install myapp ./charts/myapp
kubectl get deployment,pod,svc
kubectl get endpoints myapp

# 2. Ingress 추가 반영
helm upgrade myapp ./charts/myapp -f values-dev.yaml
kubectl get ingress
curl -H "Host: myapp.local" http://$(minikube ip)/healthz

# 3. 전체 삭제 후 재설치 (재현성 검증)
helm uninstall myapp
helm install myapp ./charts/myapp -f values-dev.yaml
kubectl get deployment,pod,svc,ingress

# 업그레이드 / 롤백
helm upgrade myapp ./charts/myapp --set replicaCount=5
helm history myapp
helm rollback myapp 1

# 삭제
helm uninstall myapp

### 업그레이드 / 롤백
helm upgrade myapp ./charts/myapp --set replicaCount=5
helm history myapp
helm rollback myapp 1

### 삭제
helm uninstall myapp
```

## 6. 무엇을 하나의 차트로 묶었는가

`charts/myapp` 차트 하나에 **Deployment, Service, Ingress** 세 가지 리소스를 함께 묶었습니다.

**근거**

- 이 셋은 서로 독립적으로 존재할 이유가 없는, 하나의 애플리케이션을 외부에 노출하기 위한 최소 단위입니다. Deployment만 있으면 Pod는 뜨지만 클러스터 내부에서도 안정적으로 접근할 방법이 없고, Service까지 있어도 클러스터 외부에서는 접근할 수 없습니다. Ingress까지 있어야 비로소 "배포됨 + 접근 가능함"이 완성됩니다.
- 세 리소스는 릴리스 생명주기가 동일합니다. `helm install`로 함께 생성되고, `helm upgrade`로 함께 갱신되고, `helm uninstall`로 함께 삭제되어야 하는 대상입니다. 따로 배포하면 "Pod는 떠 있는데 라우팅 규칙이 없는" 등 중간 상태가 생길 위험이 있습니다.
- `Release.Name`을 기준으로 세 리소스의 이름과 라벨(`app: {{ .Release.Name }}`)이 서로 연결되어 있어, 하나의 차트/릴리스로 관리하는 것이 자연스럽습니다.

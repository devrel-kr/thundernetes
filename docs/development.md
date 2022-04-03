---
layout: default
title: 개발
nav_order: 12
---

# 개발

이 문서에는 Thundernetes 소스 코드 작업을 위한 개발 노트와 팁이 포함되어 있습니다.

## 새로운 Thundernetes 버전 릴리스하기

2개 PR을 필요로합니다.

- 이 저장소의 루트에있는 `.versions` 파일을 새 버전으로 업데이트했는지 확인하십시오.
- `make clean` 을 실행하여 이전 빌드의 캐시 된 아티팩트가 삭제되었는지 확인하십시오.
- 푸시 및 머지
- 새로운 이미지를 만들기 위해 수동으로 GitHub Actions 워크플로 [여기](https://github.com/PlayFab/thundernetes/actions/workflows/publish.yml)를 실행합니다.
- main 브랜치에서 최신 변경 사항을 가져옵니다.
- `make create-install-files` 를 실행하여 운영자 설치 파일을 생성합니다.
- [netcore-sample YAML 파일](https://github.com/PlayFab/thundernetes/samples/netcore)에 대한 이미지 변경합니다.
- 푸시 및 머지

## 메트릭

- Prometheus 및 Prometheus operator를 사용하는 경우 `config/default/kustomization.yaml` 파일에서 `# [PROMETHEUS]`로 모든 섹션의 주석 처리를 제거합니다. 자세한 내용은 [여기](https://book.kubebuilder.io/reference/metrics.html)를 참고합니다.
- 메트릭 서버에 대한 인증을 활성화하려면 ``config/default/kustomization.yaml` 파일의 이 줄에서 주석을 제거합니다: `- manager_auth_proxy_patch.yaml`

## macOS에서 엔드 투 엔드 테스트 실행하기

먼저, 엔드투엔드 테스트는 `envsubst` 유틸리티가 필요하며, `brew install gettext && brew link --force gettext`를 실행 가능한 Homebrew가 설치되어 있다고 가정합니다. 
Go를 설치했다고 가정하며, 이때 `go install sigs.k8s.io/kind@latest`로 kind를 설치해야 합니다. Kind는 `$(go env GOPATH)/bin` 디렉터리에 설치됩니다. 이때 `cp $(go env GOPATH)/bin/kind ./operator/testbin/bin/kind` 같은 명령이 있는 `<projectRoot>/operator/testbin/bin/` 폴더로 kind를 이동시켜야 합니다. `make builddockerlocal createkindcluster e2elocal`로 엔드 투 엔드 테스트를 실행할 수 있습니다.

## 다양한 스트립트

### 테스팅을 위한 인증서 생성하기

```
openssl genrsa 2048 > private.pem
openssl req -x509 -days 1000 -new -key private.pem -out public.pem
kubectl create namespace thundernetes-system
kubectl create secret tls tls-secret -n thundernetes-system --cert=public.pem --key=private.pem
```

### 게임 서버 할당하기

#### TLS auth와 작업하기

```bash
IP=$(kubectl get svc -n thundernetes-system thundernetes-controller-manager -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl --key ~/private.pem --cert ~/public.pem --insecure -H 'Content-Type: application/json' -d '{"buildID":"85ffe8da-c82f-4035-86c5-9d2b5f42d6f5","sessionID":"85ffe8da-c82f-4035-86c5-9d2b5f42d6f5"}' https://${IP}:5000/api/v1/allocate
```

#### TLS auth 없이 (작업하기)

```bash
IP=$(kubectl get svc -n thundernetes-system thundernetes-controller-manager -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
curl -H 'Content-Type: application/json' -d '{"buildID":"85ffe8da-c82f-4035-86c5-9d2b5f42d6f5","sessionID":"85ffe8da-c82f-4035-86c5-9d2b5f42d6f5"}' http://${IP}:5000/api/v1/allocate
```

### 50개 할당 작업

#### TLS auth 없이

```bash
IP=$(kubectl get svc -n thundernetes-system thundernetes-controller-manager -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
for i in {1..50}; do SESSION_ID=$(uuidgen); curl -H 'Content-Type: application/json' -d '{"buildID":"85ffe8da-c82f-4035-86c5-9d2b5f42d6f6","sessionID":"'${SESSION_ID}'"}' http://${IP}:5000/api/v1/allocate; done
```

#### With TLS auth

```bash
IP=$(kubectl get svc -n thundernetes-system thundernetes-controller-manager -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
for i in {1..50}; do SESSION_ID=$(uuidgen); curl --key ~/private.pem --cert ~/public.pem --insecure -H 'Content-Type: application/json' -d '{"buildID":"85ffe8da-c82f-4035-86c5-9d2b5f42d6f6","sessionID":"'${SESSION_ID}'"}' https://${IP}:5000/api/v1/allocate; done
```

## 엔드 투 엔드 테스트를 로컬에서 실행하기

```bash
make clean deletekindcluster builddockerlocal createkindcluster e2elocal
```

## 컨트롤러를 로컬에서 실행하기

```bash
cd operator
THUNDERNETES_INIT_CONTAINER_IMAGE=ghcr.io/playfab/thundernetes-initcontainer:0.2.0 go run main.go
```

## [고급] 해당 저장소를 클론하여 thundernetes 설치

로컬 컴퓨터에 이 리포지토리를 `git clone` 해야 합니다. 이것을 수행하는 즉시 다음과 같은 명령을 실행해서 Thundernetes를 설치할 수 있습니다.

```bash
export TAG=0.0.2.0
IMG=ghcr.io/playfab/thundernetes-operator:${TAG} \
  IMAGE_NAME_INIT_CONTAINER=ghcr.io/playfab/thundernetes-initcontainer \
  IMAGE_NAME_SIDECAR=ghcr.io/playfab/thundernetes-sidecar-netcore \
  API_SERVICE_SECURITY=none \
   make -C pkg/operator install deploy
```

이렇게 하면 API 서비스에 대한 보안 없이 thundernetes가 설치된다는 것에 유의합니다. API 서비스를 위한 보안을 활성화하려는 경우 API 서비스를 위한 인증서와 키를 제공할 수 있어야 합니다.

OpenSSL을 사용해서 자체 서명 인증서와 키를 생성할 수 있습니다(프로덕션에는 권장되지 않음).

```bash
openssl genrsa 2048 > private.pem
openssl req -x509 -days 1000 -new -key private.pem -out public.pem
```

Operator와 같은 네임스페이스에서 Kubernetes Secret으로 인증서와 키를 설치합니다.

```bash
kubectl create namespace thundernetes-system
kubectl create secret tls tls-secret -n thundernetes-system --cert=public.pem --key=private.pem
```

이때 API 서비스를 위한 TLS 인증을 활성화하는 operator를 설치해야 합니다.

```bash
export TAG=0.3.0
IMG=ghcr.io/playfab/thundernetes-operator:${TAG} \
  IMAGE_NAME_INIT_CONTAINER=docker.io/dgkanatsios/thundernetes-initcontainer \
  IMAGE_NAME_SIDECAR=docker.io/dgkanatsios/thundernetes-sidecar-netcore \
  API_SERVICE_SECURITY=usetls \
   make -C pkg/operator install deploy
```

이것을 수행하는 즉시 `kubectl -n thundernetes-system get pods`를 실행해서 operator 파드가 실행되고 있는지 확인할 수 있습니다. 데모 게임 서버를 실행하기 위해 다음 명령을 사용할 수 있습니다.

```bash
kubectl apply -f pkg/operator/config/samples/netcore.yaml
```

이렇게 하면 두 개의 standingBy와 4개의 최대 게임 서버가 있는 GameServerBuild가 생성됩니다.
잠시 후에 게임 서버가 표시됩니다.

```bash
kubectl get gameservers # or kubectl get gs
```

```bash
NAME                           HEALTH    STATE        PUBLICIP        PORTS      SESSIONID
gameserverbuild-sample-apbjz   Healthy   StandingBy   52.183.89.4     80:24558
gameserverbuild-sample-gqhrm   Healthy   StandingBy   52.183.88.255   80:10319
```

그리고 GameServerBuild는 다음과 같습니다:

```bash
kubectl get gameserverbuild # or kubectl get gsb
```

```bash
NAME                     ACTIVE   STANDBY   CRASHES   HEALTH
gameserverbuild-sample   0        2         0         Healthy
```

GameServerBuild의 값을 변경하여 StandingBy의 수와 최대값을 편집할 수 있습니다.

```bash
kubectl edit gsb gameserverbuild-sample
```

또한 할당 API를 사용할 수 있습니다. 공용 IP를 얻기 위해 다음 명령을 사용할 수 있습니다.

```bash
kubectl get svc -n thundernetes-system thundernetes-controller-manager
```

```bash
NAME                              TYPE           CLUSTER-IP    EXTERNAL-IP    PORT(S)          AGE
thundernetes-controller-manager   LoadBalancer   10.0.62.144   20.83.72.255   5000:32371/TCP   39m
```

External-Ip 필드는 할당 API를 호출하는 데 사용할 수 있는 로드 밸런서의 공용 IP입니다.

보안 기능 없이 API 서비스를 구성했을 경우에는 다음과 같습니다.

```bash
IP=...
curl -H 'Content-Type: application/json' -d '{"buildID":"85ffe8da-c82f-4035-86c5-9d2b5f42d6f5","sessionID":"85ffe8da-c82f-4035-86c5-9d2b5f42d6f5"}' http://${IP}:5000/api/v1/allocate
```

TLS 인증을 사용하는 경우 다음과 같습니다.

```bash
IP=...
curl --key ~/private.pem --cert ~/public.pem --insecure -H 'Content-Type: application/json' -d '{"buildID":"85ffe8da-c82f-4035-86c5-9d2b5f42d6f5","sessionID":"85ffe8da-c82f-4035-86c5-9d2b5f42d6f5"}' https://${IP}:5000/api/v1/allocate
```

그러면, 게임 서버가 성공적으로 할당되었음을 확인할 수 있습니다. 

```bash
kubectl get gameservers
```

```bash
NAME                           HEALTH    STATE        PUBLICIP        PORTS      SESSIONID
gameserverbuild-sample-apbjz   Healthy   StandingBy   52.183.89.4     80:24558
gameserverbuild-sample-bmich   Healthy   Active       20.94.219.110   80:38208   85ffe8da-c82f-4035-86c5-9d2b5f42d6f5
gameserverbuild-sample-gqhrm   Healthy   StandingBy   52.183.88.255   80:10319
```

할당된 서버에서 데모 게임 서버 HTTP 엔드포인트를 호출할 수 있습니다.

```bash
curl 20.94.219.110:38208/hello
```

`Hello from <containerName>`과 같은 응답을 얻게 됩니다.

게임 서버를 깔끔하게 종료하는 엔드포인트 호출을 시도합니다.

```bash
curl 20.94.219.110:38208/hello/terminate
```

게임 서버가 종료되었음을 확인할 수 있습니다. GameServerBuild는 두 개의 standingBy로 구성되기 때문에 다른 서버는 생성되지 않습니다.

```bash
kubectl get gameservers
```

```bash
NAME                           HEALTH    STATE        PUBLICIP        PORTS      SESSIONID
gameserverbuild-sample-apbjz   Healthy   StandingBy   52.183.89.4     80:24558
gameserverbuild-sample-gqhrm   Healthy   StandingBy   52.183.88.255   80:10319
```

## kubebuilder 메모

다음과 같은 명령을 사용하는 [kubebuilder](https://github.com/kubernetes-sigs/kubebuilder)를 사용해서 프로젝트가 부트스트랩되었습니다.

```bash
kubebuilder init --domain playfab.com --repo github.com/playfab/thundernetes/pkg/operator
kubebuilder create api --group mps --version v1alpha1 --kind GameServer
kubebuilder create api --group mps --version v1alpha1 --plural gameserverbuilds --kind GameServerBuild 
kubebuilder create api --group mps --version v1alpha1 --plural gameserverdetails --kind GameServerDetail 
```

## env 변수 샘플

```bash
PUBLIC_IPV4_ADDRESS=20.184.250.154
PF_REGION=WestUs
PF_VM_ID=xcloudwus4u4yz5dlozul:WestUs:6b5973a5-a3a5-431a-8378-eff819dc0c25:tvmps_efa402aacd4f682230cfd91bd3dc0ddfae68c312f2b6905577cb7d9424681930_d
PF_SHARED_CONTENT_FOLDER=/gsdkdata/GameSharedContent
PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
PF_SERVER_INSTANCE_NUMBER=2
PWD=/app
PF_BUILD_ID=88a958b9-14fb-4ad9-85ca-5cc13207232e
GSDK_CONFIG_FILE=/gsdkata/Config/gsdkConfig.json
SHLVL=1
HOME=/root
CERTIFICATE_FOLDER=/gsdkdata/GameCertificates
PF_SERVER_LOG_DIRECTORY=/gsdkdata/GameLogs/
PF_TITLE_ID=1E03
_=/usr/bin/env
```

## Docker compose

이 리포지토리 루트의 docker-compose.yml 파일은 사이드카 개발을 촉진하기 위해 생성되었습니다.

## 변경 사항을 클러스터에서 테스트하기

변경 사항을 Kubernetes 클러스터에 thundernetes로 테스트하기 위해서는, 다음 단계를 사용할 수 있습니다.

- 프로젝트의 루트에 있는 Makefile에는 개발 중에 사용하는 컨테이너 레지스트리를 가리키는 변수 `NS`가 포함되어 있습니다. 따라서 환경에서 변수를 설정하거나 (`export NS=<your-container-registry>`) `make` 를 호출하기 전에 변수를 설정해야 합니다 (예: `NS=<your-container-registry> make build push`).
- 컨테이너 레지스트리에 로그인합니다 (`docker login`)
- `make clean build push` 를 실행하여 컨테이너 이미지를 빌드하고 컨테이너 레지스트리로 푸시합니다.
- `create-install-files-dev` 를 실행하여 클러스터에 대한 설치 파일을 만듭니다.
- 생성 된 설치 파일의 `installfilesdev` 폴더를 체크 아웃하십시오. 이 파일은 .gitignore 에 포함되어 있으므로 커밋되지 않습니다.
- 필요에 따라 변경 사항을 테스트합니다.
- 단일 명령어: `NS=docker.io/<repo>/ make clean build push create-install-files-dev`
 
---
layout: default
title: Quickstart
nav_order: 2
---

# 빠른 시작

Azure Kubernetes Service (AKS)와 [kind](https://kind.sigs.k8s.io/) 최신 버전으로 Thundernetes 테스트를 하였지만 이론적으로는 노드(게임 서버를 클러스터 외부에 노출하고자 원하는 무언가입니다) 당 하나의 IP가 있는 모든 Kubernetes 클러스터에 설치할 수 있습니다. Thundernetes를 설치하려는 곳에 관련 섹션을 읽으세요.

> _**참고**_: Windows를 사용하는 경우, [Windows Subsystem for Linux](https://docs.microsoft.com/windows/wsl/install)를 사용하여 아래 나열된 CLI 명령어를 실행할 것을 권장합니다.

## 노드 당 공용 IP로 Azure Kubernetes Service 클러스터 만들기

노드 당 하나의 공용 IP가 있는 Azure Kubernetes Service (AKS) 클러스터를 생성하기 위해 다음 [Azure CLI](https://docs.microsoft.com/cli/azure/) 명령어를 사용할 수 있습니다.

```bash
az login # Azure Cloud shell을 사용하는 경우에는 필요로 하지 않습니다
# 해당 값들을 선호하는 값으로 변경해야 합니다
AKS_RESOURCE_GROUP=thundernetes # AKS가 설치될 리소스 그룹 이름
AKS_NAME=thundernetes # AKS 클러스터 이름
AKS_LOCATION=westus2 # AKS 데이터센터 지역(리전)
AKS_VERSION=1.22.4 # 리전에서 지원하는 Kubernetes 버전으로 변경합니다

# 리소스 그룹 생성
az group create --name $AKS_RESOURCE_GROUP --location $AKS_LOCATION
# 노드 당 공용 IP를 갖는 기능을 활성화하여 새로운 AKS 클러스터를 생성합니다
az aks create --resource-group $AKS_RESOURCE_GROUP --name $AKS_NAME --ssh-key-value ~/.ssh/id_rsa.pub --kubernetes-version $AKS_VERSION --enable-node-public-ip
# 클러스터에 대한 자격 증명(credentials)을 얻어 별도 파일로 저장합니다
az aks get-credentials --resource-group $AKS_RESOURCE_GROUP --name $AKS_NAME --file ~/.kube/config-thundernetes
# 클러스터가 로드되어 실행 중인지 확인합니다
export KUBECONFIG=~/.kube/config-thundernetes
kubectl cluster-info
# Network Security Group (NSG)를 수정하여 10000-12000번 포트를 인터넷에 엽니다
# 다음 섹션에 있는 안내를 확인합니다
```

### 포트 번호 10000-12000번을 인터넷에 노출

Thundernetes는 VM이 공용 IP (따라서 게임 서버에 접근 가능)를 가지고 있고 인터넷으로부터 포트 범위 10000-12000에서 네트워크 트래픽을 수락할 수 있어야 합니다.

> _**NOTE**_: 포트 범위를 구성하실 수 있습니다. [여기](howtos/configureportrange.md)에서 자세히 확인하세요. 

포트를 열기 위해서는 *AKS 클러스터가 생성된 이후* 다음 단계를 수행해야 합니다.

* [Azure 포털](https://portal.azure.com)에 로그인합니다.
* AKS 리소스를 보관하고 있는 리소스 그룹을 찾습니다. `MC_resourceGroupName_AKSName_location`와 같은 이름일 것입니다. 다른 방법으로는 `az resource show --namespace Microsoft.ContainerService --resource-type managedClusters -g $AKS_RESOURCE_GROUP -n $AKS_NAME -o json | jq .properties.nodeResourceGroup` 를 셸에 입력하여 찾아봅니다.
* `aks-agentpool-********-nsg`와 같은 이름을 가진 네트워크 보안 그룹 개체를 찾습니다.
* **인바운드 보안 규칙**을 선택합니다.
* Select **Add (추가)**를 선택하여 프로토콜 (게임에 따라 TCP 또는 UDP 중에서 선택할 수도 있음)은 **Any(모두)** 및 대상 포트 범위를 **10000-12000** 로 하는 새로운 규칙을 생성합니다. 적절한 규칙 이름을 선택하고 그 외 모두는 기본 값으로 남겨둡니다.

또는 `$RESOURCE_GROUP_WITH_AKS_RESOURCES`와 `$NSG_NAME` 변수를 적절한 값으로 설정한 후 다음 명령을 사용할 수 있습니다.

```bash
az network nsg rule create \
  --resource-group $RESOURCE_GROUP_WITH_AKS_RESOURCES \
  --nsg-name $NSG_NAME \
  --name AKSThundernetesGameServerRule \
  --access Allow \
  --protocol "*" \
  --direction Inbound \
  --priority 1000 \
  --source-port-range "*" \
  --destination-port-range 10000-12000
```

마지막으로 Kubernetes 클러스터를 관리하기 위해 kubectl([안내](https://kubernetes.io/ko/docs/tasks/tools/#kubectl))을 설치하는 것을 잊지 마세요.

## kind를 통해 로컬에서 Kubernetes 설치하기 

Kubernetes를 로컬에서 실행하기 위해 [kind](https://kind.sigs.k8s.io/), [k3d](https://k3d.io/) 또는 [minikube](https://kubernetes.io/docs/getting-started-guides/minikube/) 등 다양한 옵션을 사용할 수 있습니다. 이 가이드에서는 [kind](https://kind.sigs.k8s.io/)를 사용할 것입니다.

* Kind는 Docker가 필요하기 때문에 Docker가 실행 중인지 확인하세요. [Windows용 Docker는 여기](https://docs.docker.com/desktop/windows/install/)에서 찾을 수 있습니다.
* [여기](https://kind.sigs.k8s.io/docs/user/quick-start/#installation) 안내를 이용해서 kind를 설치합니다.
* 다음 내용을 사용해서 "kind-config.yaml" 파일을 생성하여 클러스터를 구성합니다. 

전달하려는 포트에 대해 특별한 주의가 필요합니다(아래 나열된 "containerPort"). 무엇보다도, API 서버가 사용하는 포트이기 때문에 포트 5000을 노출해야 합니다. 게임 서버 할당에 이 포트를 사용할 것입니다.
그 후에 트래픽을 전송하여 게임 서버를 테스트하기 위해 포트를 선택적으로 지정할 수 있습니다. Thundernetes는 게임 서버를 위해 10000-12000 범위에서 포트를 동적으로 할당합니다. 이 범위에서의 포트 할당은 순차적입니다. 예를 들어, 각각 하나의 포트를 가지고 있는 두 개의 게임 서버를 사용하는 경우, 첫 번째 게임 서버 포트는 포트 10000으로 매핑되고 두 번째는 포트 10001로 매핑됩니다. GameServerBuild의 규모를 축소한 다음 다시 확대하는 경우 같은 포트를 얻지 못할 것임을 감안하세요. 따라서 kind 구성에서 사용할 포트에 특히 주의해야 합니다.

이 내용을 `kind-config.yaml`이라는 파일에 저장합니다.

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
- role: control-plane
  extraPortMappings:
  - containerPort: 5000
    hostPort: 5000
    listenAddress: "0.0.0.0"
    protocol: tcp
  - containerPort: 10000
    hostPort: 10000
    listenAddress: "0.0.0.0"
    protocol: tcp
  - containerPort: 10001
    hostPort: 10001
    listenAddress: "0.0.0.0"
    protocol: tcp
```

* `kind create cluster --config /path/to/kind-config.yaml`을 실행합니다.
* Kubernetes 클러스터를 관리하기 위해 kubectl([안내](https://kubernetes.io/ko/docs/tasks/tools/#kubectl))을 설치합니다.
* 설치에 성공하면 `kubectl cluster-info`를 실행하여 클러스터가 실행 중인지 확인합니다. 다음과 같은 것을 얻게 됩니다.

```bash
Kubernetes control plane is running at https://127.0.0.1:34253
CoreDNS is running at https://127.0.0.1:34253/api/v1/namespaces/kube-system/services/kube-dns:dns/proxy
```

## 설치 스크립트를 통한 Thundernetes 설치

일단 Kubernetes 클러스터가 실행 중이면 Thundernetes를 설치하기 위해 다음 명령을 실행할 수 있습니다. 이렇게 하면 할당 API 서비스를 위한 TLS 인증 *없이* Thundernetes가 설치되며 이는 테스트 환경에서만 사용해야 합니다.

```bash
kubectl apply -f https://raw.githubusercontent.com/PlayFab/thundernetes/main/installfiles/operator.yaml
```

### TLS 인증을 사용하는 Thundernetes 설치

Thundernetes API 서비스를 보호하기 위해 사용될 인증서를 생성/구성해야 합니다. 프로덕션 환경에서는 (잘 알려진 CA로 서명된) 적절하게 구성된 인증서를 추천합니다.

테스트를 위해 자체 서명 인증서를 생성하고 할당 API 서비스를 보호하는 데 사용할 수 있습니다. OpenSSL을 사용해서 자체 서명 인증서와 키를 생성할 수 있습니다(물론 이 시나리오는 프로덕션에서는 권장하지 않습니다).

```bash
openssl genrsa 2048 > private.pem
openssl req -x509 -days 1000 -new -key private.pem -out public.pem
```

일단 인증서를 갖게 되면 [Kubernetes 시크릿](https://kubernetes.io/ko/docs/concepts/configuration/secret/)으로 등록해야 합니다. 컨트롤러와 같은 네임스페이스에 *있어야 하며* `tls-secret`이라고 부릅니다. 기본 네임스페이스 `thundernetes-system`에 설치할 것입니다.

```bash
kubectl create namespace thundernetes-system
kubectl create secret tls tls-secret -n thundernetes-system --cert=/path/to/public.pem --key=/path/to/private.pem
```

그러면 다음 스크립트를 실행해서 API 서비스를 위한 TLS 보안이 있는 Thundernetes를 설치할 수 있습니다.

```bash
kubectl apply -f https://raw.githubusercontent.com/PlayFab/thundernetes/main/installfiles/operator_with_security.yaml
```

두 개의 설치 파일(operator.yaml과 operator_with_security.yaml)은 컨트롤러 컨테이너로 전달되는 API_SERVICE_SECURITY 환경 변수를 제외하고 동일합니다.

이제는 Thundernetes에서 게임 서버를 실행할 준비가 된 것입니다. 샘플 게임 서버 중 하나를 실행하려면 계속 읽으세요. 그렇지 않고 자체 게임 서버를 실행하려면 [이 문서](developertool.md)로 이동하세요.

## 샘플 게임 서버 실행

Thundernetes는 [GSDK](https://github.com/PlayFab/gsdk)와 통합되어 있는 두 개의 샘플 게임 서버 프로젝트와 함께 제공됩니다. 이들 중 하나를 이용해서 Thundernetes 설치를 검증할 수 있습니다.

### .NET Core Fake 게임 서버

[여기](https://github.com/playfab/thundernetes/samples/netcore)에 위치한 이 샘플은 GSDK를 구현하는 단순한 .NET Core 웹 API 앱입니다. 다음 명령을 실행하여 Kubernetes 클러스터에 설치할 수 있습니다.

```bash
kubectl apply -f https://raw.githubusercontent.com/PlayFab/thundernetes/main/samples/netcore/sample.yaml
```

> _**참고**_: GameServerBuild를 위해 지정해야 하는 필드에 대해 읽으려면 [이 문서](gameserverbuild.md)를 확인하세요.

실행 중인 게임 서버를 확인하기 위해 `kubectl get gs`를 사용해 보면, 다음과 유사한 내용을 볼 수 있습니다.

```bash
dgkanatsios@desktopdigkanat:thundernetes$ kubectl get gs
NAME                                   HEALTH    STATE        PUBLICIP      PORTS      SESSIONID
gameserverbuild-sample-netcore-ayxzh   Healthy   StandingBy   52.183.89.4   80:10001
gameserverbuild-sample-netcore-mveex   Healthy   StandingBy   52.183.89.4   80:10000
```

그리고 GameServerBuild의 상태를 확인하기 위해 `kubectl get gsb` 를 사용해 봅니다.

```bash
dgkanatsios@desktopdigkanat:thundernetes$ kubectl get gsb
NAME                             STANDBY   ACTIVE   CRASHES   HEALTH
gameserverbuild-sample-netcore   2/2       0        0         Healthy
```

> _**참고**_: `gs`와 `gsb`는 단지 각각 GameServer와 GameServerBuild의 약어에 해당합니다. `kubectl get gameserver` 또는 `kubectl get gameserverbuild`를 대신 입력할 수 있습니다.

GameServerBuild의 크기를 조정하기 위해 `kubectl edit gsb gameserverbuild-sample-netcore`를 수행하고 max/standingByy 수를 편집할 수 있습니다.

#### 게임 서버 할당

GameServer를 할당하면 상태가 "StandingBy"에서 "Active"으로 전환되고 "ReadyForPlayers" GSDK 호출이 차단 해제가 됩니다. [이 문서](gameserverlifecycle.md)를 읽고 GameServer에 대한 수명 주기에 관해 알아보세요.

Azure Kubernetes Service에서 실행하고 있는 경우 다음 명령을 사용해서 게임 서버를 할당할 수 있습니다.

```bash
# 트래픽을 thundernetes API 서비스로 라우팅하는 데 사용되는 외부 로드 밸런서의 IP를 가져옵니다.
IP=$(kubectl get svc -n thundernetes-system thundernetes-controller-manager -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
# 할당 호출을 수행합니다. buildID가 빌드를 생성할 때 사용했던 것과 같은 것인지 확인합니다.
# sessionID는 게임 서버 세션을 추적하기 위해 사용할 수 있는 고유한 식별자 (GUID)입니다.
curl -H 'Content-Type: application/json' -d '{"buildID":"85ffe8da-c82f-4035-86c5-9d2b5f42d6f6","sessionID":"ac1b7082-d811-47a7-89ae-fe1a9c48a6da"}' http://${IP}:5000/api/v1/allocate
```

보시다시피, 할당 호출에 대한 인수는 두 개입니다.

* buildID: GameServerBuild에서 구성된 buildID와 같아야 합니다.
* sessionID: 게임 서버 세션을 식별하기 위해 사용할 수 있는 GUID. 할당한 각 게임 서버에 고유해야 합니다. 사용 중인 sessionID를 사용해서 할당을 시도하는 경우 호출이 기존 게임 서버의 세부 정보를 반환합니다.  

> _**참고**_: 이 호출은 PlayFab 멀티 플레이어 서버의 [RequestMultiplayerServer](https://docs.microsoft.com/rest/api/playfab/multiplayer/multiplayer-server/request-multiplayer-server)를 호출하는 것과 같습니다. 

할당 호출의 결과는 JSON 형식으로 된 서버의 IP/포트입니다.

```bash
{"IPV4Address":"52.183.89.4","Ports":"80:10000","SessionID":"ac1b7082-d811-47a7-89ae-fe1a9c48a6da"}
```

이제 IP/포트를 사용해서 할당된 게임 서버에 연결할 수 있습니다. 임의로 준비한(fake) 게임 서버는 컨테이너의 호스트 이름을 반환하는 `/Hello` 엔드포인트를 노출합니다.

```bash
dgkanatsios@desktopdigkanat:thundernetes$ curl 52.183.89.4:10000/Hello
Hello from fake GameServer with hostname gameserverbuild-sample-netcore-mveex
```

동시에, 게임 서버를 다시 확인할 수 있습니다. 원본 요청이 두 개의 standingBy와 4개의 최대 서버에 대한 것이었기 때문에 지금 두 개의 StandingBy 한 개의 Active를 볼 수 있습니다. 또한 Active GameServer에 대한 SessionID도 볼 수 있습니다.

```bash
dgkanatsios@desktopdigkanat:thundernetes$ kubectl get gs
NAME                                   HEALTH    STATE        PUBLICIP      PORTS      SESSIONID
gameserverbuild-sample-netcore-ayxzh   Healthy   StandingBy   52.183.89.4   80:10001
gameserverbuild-sample-netcore-mveex   Healthy   Active       52.183.89.4   80:10000   ac1b7082-d811-47a7-89ae-fe1a9c48a6da
gameserverbuild-sample-netcore-pxrqx   Healthy   StandingBy   52.183.89.4   80:10002
```

#### 게임 서버의 수명 주기

게임 서버 프로세스가 계속 실행 중일 때까지, 게임 서버는 Active 상태로 남아 있습니다. 게임 서버 프로세스가 일단 종료되면 GameServer에 대한 Custom Resource가 삭제될 것입니다. 이를 통해 게임 서버 파드가 삭제되고 대신 새로운 파드가 해당 위치(GameServerBuild 최대로 지정한 범위를 초과하지 않는 범위 내에서)에 생성됩니다. GameServer 수명 주기에 대한 보다 자세한 정보는 [여기](gameserverlifecycle.md)에서 확인하세요.

### Openarena

[여기](https://github.com/playfab/thundernetes/samples/openarena)에 위치한 이 샘플은 인기있는 오픈 소스 FPS 게임인 [OpenArena](http://www.openarena.ws/smfnews.php)를 기반으로 합니다. 다음 스크립트를 사용하여 설치할 수 있습니다.

```bash
kubectl apply -f https://raw.githubusercontent.com/PlayFab/thundernetes/main/samples/openarena/sample.yaml
```

임의로 준비한(fake)게임 서버에서와 동일한 명령을 사용해서 게임 서버를 할당할 수 있습니다 활성화된 서버에 연결하기 위해서는 [여기](http://openarena.ws/download.php?view.4)에서 OpenArena 클라이언트를 다운로드해야 합니다.

## 대기(standingBy) 및 최대 서버 수 수정

max/standingBy 수를 수정하기 위해 `kubectl edit gsb <name-of-your-gameserverbuild>` 명령어를 사용할 수 있습니다. active+standingBy 수가 최대값보다 클 수는 없습니다.

## Thundernetes 제거하기

각 GameServer에는 종료자(finalizer)가 있으므로 GameServer 인스턴스를 제거하기 전에 컨트롤러를 제거하면 GameServer 인스턴스를 삭제하려고 할 때 멈추는 상황이 발생할 것입니다.

```bash
kubectl delete gsb --all -A # 이렇게 하면 모든 네임스페이스에서 모든 GameServerBuild가 삭제되고, 결국 모든 GameServer가 삭제될 것입니다
kubectl get gs -A # 모든 네임스페이스에서 GameServer가 없는지 확인합니다
kubectl delete ns thundernetes-system # 모든 thundernetes 리소스와 함께 네임스페이스를 삭제합니다
# RBAC 리소스를 삭제합니다. 서비스 계정 및 역할 바인딩을 위한 네임스페이스를 추가해야 할 수 있습니다.
kubectl delete clusterrole thundernetes-proxy-role thundernetes-metrics-reader thundernetes-manager-role thundernetes-gameserver-editor-role
kubectl delete serviceaccount thundernetes-gameserver-editor
kubectl delete clusterrolebinding thundernetes-manager-rolebinding thundernetes-proxy-rolebinding
kubectl delete rolebinding thundernetes-gameserver-editor-rolebinding
```

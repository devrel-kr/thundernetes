---
layout: default
title: 아키텍처
nav_order: 8
---

# 아키텍처

이 문서는 Thundernetes 아키텍처와 해당 구현과 관련한 다양한 디자인 노트에 대해 설명합니다.

![Architecture diagram](diagram.png)

## 목표

최종 목표는 게임 개발자가 온프레미스 또는 퍼블릭 클라우드 프로바이더에 있는 Kubernetes 클러스터에서 게임 서버 SDK([gsdk](http://github.com/playfab/gsdk))를 사용하는 Linux 게임 서버를 호스팅할 수 있는 유틸리티를 만드는 것입니다. 이 유틸리티는 게임 서버 자동 크기 조정 및 게임 서버 할당을 활성화합니다. "allocation(할당)"이라는 용어는 미리 워밍된 게임 서버(이 상태를 "StandingBy"라고 함)가 "Active(활성)" 상태로 전환되어 플레이어(게임 클라이언트)가 연결할 수 있음을 의미합니다. 잠재적인 규모 축소 시, Thundernetes는 액티브 게임 서버나 액티브 게임 서버가 실행 중인 노드를 무너뜨리지 않습니다.

[Operator](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)를 생성하여 Kubernetes를 확장합니다. 이는 사용자 지정 [컨트롤러](https://kubernetes.io/docs/concepts/architecture/controller/)와 몇 개의 [사용자 지정 리소스(CRD)](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/) 생성을 동반합니다. [controller-runtime](https://github.com/kubernetes-sigs/controller-runtime)를 사용하는 Operator 파일을 스캐폴드(scaffold)하기 위해 오픈 소스 도구 [kubebuilder](https://github.com/kubernetes-sigs/kubebuilder)를 사용합니다.

특히, 각각 해당하는 CRD를 대표하는 프로젝트에 대한 세 개의 핵심 엔터티가 있습니다.

- **GameServerBuild** ([YAML](http://github.com/playfab/thundernetes/tree/main/pkg/operator/config/crd/bases/mps.playfab.com_gameserverbuilds.yaml), [Go](http://github.com/playfab/thundernetes/tree/main/pkg/operator/api/v1alpha1/gameserverbuild_types.go)): 이는 같은 파드 템플릿을 실행할 GameServers 컬렉션을 나타내며 빌드 내에서 스케일 인/아웃할 수 있습니다(즉, 인스턴스를 추가하거나 제거). 같은 GameServerBuild의 구성원인 GameServers는 실행 환경에서 같은 세부 정보를 공유합니다(예: 이들 모두는 같은 멀티플레이어 맵 또는 같은 유형의 게임을 시작할 수 있음). 따라서 게임의 "캡처 더 플래그(Capture the flag)" 모드를 위한 GameServerBuild 하나와 "컨퀘스트(Conquest)" 모드를 위한 또 다른 GameServerBuild를 가질 수 있습니다. 또는, 맵 "X"에서 플레이하고 있는 플레이어를 위한 GameServerBuild 하나와 맵 "Y”에서 플레이하고 있는 플레이어를 위한 또 다른 GameServerBuild가 있습니다. 하지만 각 GameServer가 BuildMedata(GameServerBuild에 구성됨)를 통해 또는 환경 변수를 통해 작동하는 방식을 수정할 수 있습니다.
- **GameServer** ([YAML](http://github.com/playfab/thundernetes/tree/main/pkg/operator/config/crd/bases/mps.playfab.com_gameservers.yaml), [Go](http://github.com/playfab/thundernetes/tree/main/pkg/operator/api/v1alpha1/gameserver_types.go)): 이는 멀티플레이어 게임 서버 자체를 나타냅니다. 각 DedicatedGameServer는 실행 가능한 게임 서버를 포함하고 있는 컨테이너 이미지를 실행할 단일의 해당 하위 [파드](https://kubernetes.io/docs/concepts/workloads/pods/pod/)를 가지고 있습니다.
- **GameServerDetail** ([YAML](http://github.com/playfab/thundernetes/tree/main/pkg/operator/config/crd/bases/mps.playfab.com_gameserverdetails.yaml), [Go](http://github.com/playfab/thundernetes/tree/main/pkg/operator/api/v1alpha1/gameserverdetail_types.go)): 이는 GameServer에 대한 세부 사항을 대표합니다. InitialPlayers, ConnectedPlayerCount 및 ConnectedPlayer 이름/ID와 같은 정보를 포함합니다. Thundernetes는 GameServer당 하나의 GameServerDetail 인스턴스를 생성합니다. 이러한 커스텀 리소스를 생성하는 이유는 GameServer에 해당 정보를 오버로드하고 싶지 않기 때문입니다. 이러한 방식으로 메모리를 덜 소비하고 성능을 높일 수 있습니다.

## GSDK integration

클러스터의 모든 노드(또는 사용자가 NodeSelectors로 DaemonSet을 구성하는 경우)에서 실행되는 파드를 생성하는 [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)를 만들었습니다. DaemonSet 파드에서 실행 중인 프로세스를 NodeAgent라고 합니다. NodeAgent는 실행되는 노드의 게임 서버 파드로부터 모든 GSDK 호출을 수신하는 웹 서버를 설정하고 그에 따라 게임 서버 상태를 수정합니다. 간단히 말하자면, 모든 게임 서버는 동일한 노드의 NodeAgent 프로세스로 heartbeats(GSDK를 통해)를 처리합니다. NodeAgent는 또한 이러한 GameServer 객체의 상태를 (Kubernetes watch를 통해) "감시"하여 변경시 알림을 받는 일을 담당합니다. 이는 게임 서버가 할당된 시간(게임 상태가 Active로 전환됨)을 추적하는 데 특히 유용합니다.

GSDK는 [오픈 소스](https://github.com/PlayFab/gsdk)이며, 가장 인기있는 게임 프로그래밍 언어 및 환경을 지원합니다. GSDK 문서 [여기](https://docs.microsoft.com/gaming/playfab/features/multiplayer/servers/integrating-game-servers-with-gsdk)를 확인하고, 바로 사용할 수 있는 샘플은 [여기](https://github.com/PlayFab/MpsSamples)에서 찾을 수 있습니다. GSDK는 Azure PlayFab MPS(멀티플레이어 서버)에서 실행되는 게임 서버에서도 사용되므로 Thundernetes에서 MPS 서비스로(또는 그 반대의 경우) 매우 원활하게 마이그레이션할 수 있습니다. 게임 서버 프로세스에서 GSDK 통합을 확인하려면 [LocalMultiplayerAgent](https://github.com/PlayFab/MpsAgent) 도구를 사용하고 자세한 내용은 [여기](howtos/runlocalmultiplayeragent.md)를 확인하는 것이 좋습니다.

### initcontainer

GSDK 라이브러리는 파일(GSDKConfig.json)에서 구성을 읽어야 합니다. 이 파일을 만들고 GameServer 파드에서 읽을 수 있도록 하기 위해 GameServer 컨테이너와 볼륨 마운트를 공유하고 읽을 구성 파일을 만드는 경량 Kubernetes [Init Container](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/)를 만들었습니다. 초기화 컨테이너와 GameServer 컨테이너 모두 동일한 파드의 일부입니다.

## 엔드 투 엔드 (e2e) 테스팅

엔드 투 엔드 테스팅 시나리오를 위한 [kind](https://kind.sigs.k8s.io/)와 Kubernetes [client-go](https://github.com/kubernetes/client-go) 라이브러리를 사용합니다. Kind는 그 안에서 게임 서버를 설정 및 할당하고 다양한 시나리오를 테스트하는 Kubernetes 클러스터를 동적으로 설정합니다. 세부 정보는 [해당](../cmd/e2e) 폴더를 확인하세요.

## 메트릭

Thundernetes는 게임 서버 관리에 관한 다양한 메트릭을 [Prometheus](https://prometheus.io) 형식으로 노출합니다. 이를 보려면 다음을 통해 트래픽을 컨트롤러의 포트 8080으로 전달할 수 있습니다.


```bash
kubectl port-forward -n thundernetes-system deployments/thundernetes-controller-manager 8080:8080
```

그 다음, 메트릭을 보기 위해 브라우저를 사용해서 `http://localhost:8080/metrics`를 향하게 할 수 있습니다.

현재 [Prometheus](https://prometheus.io)와 [Grafana](https://grafana.org)가 설치되어 있는 경우 [샘플 대시보드](http://github.com/playfab/thundernetes/tree/main/samples/grafana/readme.md)를 활용해서 현재 컨트롤러와 게임서버 파드 메트릭을 시각화할 수 있습니다.

## 포트 할당

Thundernetes는 노드당 공용 IP를 가진 쿠버네티스 클러스터가 필요합니다. [Azure Kubernetes Service - AKS](https://docs.microsoft.com/azure/aks/intro-kubernetes)와 [kind](https://kind.sigs.k8s.io/)를 사용하는 로컬 클러스터에서 광범위하게 테스트했습니다. 또한 Thundernetes가 게임 네트워크 트래픽을 수신하고 게임 서버 파드로 전달할 수 있도록 기본적으로 Kubernetes 노드에 설정하는 포트이므로 클러스터에 포트 10000-12000이 열려 있어야 합니다. 

> _**NOTE**_: 공용 IP 없이 쿠버네티스 클러스터를 사용할 수도 있습니다. 그러나 게임 서버에 액세스하려면 고유한 네트워크 아키텍처를 구성해야 합니다. 예를 들어 클라우드 프로바이더에서 제공하는 로드 밸런서를 사용하는 경우 로드 밸런서의 공용 엔드포인트에서 Kubernetes 클러스터의 내부 엔드포인트로의 경로를 구성해야 합니다. HTTP 기반 게임 서버(예: WebGL)에 대한 [traefik](https://github.com/traefik/traefik) 인그레스 컨트롤러를 사용하는 예제 컨트롤러에 대해 [여기](https://github.com/dgkanatsios/thundernetescontrib/tree/main/traefikingress)를 확인합니다.

또한 Thundernetes는 외부 연결을 위해 클러스터에서 10000-12000 범위의 포트를 열어야 합니다(즉, Azure Kubernetes Service의 경우 이 포트 범위는 해당 네트워크 보안 그룹에서 들어오는 트래픽을 허용해야 함). 이 포트 범위는 구성 가능하며 자세한 내용은 [여기](howtos/configureportrange.md)를 확인합니다.

각 GameServerBuild에는 들어오는 클라이언트 연결을 위해 각 게임 서버가 수신 대기하는 포트에 대한 정보가 들어 있는 *portsToExpose* 필드가 포함되어 있습니다. GameServer Pod가 생성되면 *portsToExpose* 필드의 각 포트에는 Thundernetes 컨트롤러의 PortRegistry 메커니즘을 통해 (기본값) 범위 10000-12000(외부 포트라고 함)의 포트가 할당됩니다. 게임 클라이언트는 이 외부 포트로 트래픽을 보낼 수 있으며 이는 게임 서버 컨테이너 포트로 전달됩니다. GameServer 세션이 종료되면 포트는 사용 가능한 포트 풀로 다시 반환되며 나중에 다시 사용될 수 있습니다.

> _**NOTE**_: PortRegistry에서 할당하는 각 포트는 포드 정의의 HostPort 필드에 할당됩니다. 클러스터의 노드에 공용 IP가 있다는 사실은 클러스터 외부에서 이 포트에 액세스할 수 있도록 합니다.
> _**NOTE**_: Thundernetes는 GameServerBuild 파드 정의에서 요청된 경우 GameServer Pods에 대한 `hostNetwork` 네트워킹을 지원합니다.

## GameServer 할당

[게임 서버 수명 주기](gameserverlifecycle.md) 문서에 먼저 익숙해져 있는지 확인하십시오. 플레이어가 연결할 수 있도록 GameServer를 할당할 때(Active 상태로 전환), Thundernetes는 다음 두 가지 작업을 수행해야 합니다.

- StandindBy 상태에서 요청된 GameServerBuild에 대한 게임 서버 인스턴스를 찾아 활성 상태로 업데이트합니다.
- 해당 게임 서버 파드 (특히, 해당 파드에 대한 GSDK 호출을 처리하는 노드 에이전트)에 게임 서버 상태가 현재 활성 상태임을 알립니다. NodeAgent는 이 정보를 GameServer 컨테이너에 전달합니다. 이 작업을 수행하는 방법은 다음과 같습니다: 각 GameServer 프로세스/컨테이너는 NodeAgent에 정기적으로 하트비트(JSON HTTP 요청 전송)를 보냅니다. NodeAgent가 게임 서버 상태가 Active로 전환되었다는 알림을 받으면 상태 변경에 대한 정보를 GameServer 컨테이너로 다시 보냅니다.

두 번째 단계를 수행할 수 있는 방법에는 두 가지가 있습니다.

- NodeAgent에서 Kubernetes의 API 서버로 [Kubernetes watch](https://kubernetes.io/docs/reference/using-api/api-concepts/#efficient-detection-of-changes)를 가지고 게임 서버가 업데이트될 때 알림을 받게 됩니다. 이 접근 방식은 게임 서버 파드에 대한 RBAC 규칙을 구성할 수 있으므로 보안 관점에서 잘 작동합니다.
- 컨트롤러의 할당 API 서비스가 할당 요청을 NodeAgent에 전달하도록 합니다. 이 작업은 NodeAgent가 클러스터 내에 HTTP 서버를 노출하도록 함으로써 수행됩니다. 물론 이것은 클러스터의 컨테이너에서 실행되는 프로세스를 신뢰한다고 가정합니다.

NodeAgent와 통신하기 위해 우리는 결국 첫 번째 접근 방식을 선택했습니다. 두 번째 접근 방식은 처음에는 사용되었지만 보안 문제로 인해 포기되었습니다.

또한 사용자가 게임 서버를 할당하면 InitialPlayers(있는 경우)를 저장하는 GameServerDetail 사용자 지정 리소스의 인스턴스를 만듭니다. GameServerDetail은 게임 서버 CR과 1:1 관계가 있으며 동일한 이름을 공유합니다. 또한 GameServer는 GameServerDetail 인스턴스의 소유자이므로 GameServer가 삭제되면 둘 다 삭제됩니다. GameServerDetail은 또한 UpdateConnectedPlayers GSDK 메서드로 업데이트할 수 있는 ConnectedPlayersCount 및 ConnectedPlayers 이름/ID를 추적합니다.

여기서 언급 할 가치가 있는 부분으로, NodeAgent 프로세스가 0.1 (버전)까지는 GameServer 포드 내부에 존재하는 사이드카 컨테이너라는 사실입니다. 그러나 버전 0.2에서는 클러스터의 모든 노드에서 실행되는 NodeAgent 파드로 전환했습니다. 이것은 사이드카 컨테이너의 필요성을 피하기 위해 수행되었으며 `hostNetwork` 기능을 사용할 수있게되었습니다.

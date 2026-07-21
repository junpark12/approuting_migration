# AKS 운영 클러스터 Best Practice 체크리스트 (High/Medium 우선순위)

이 문서는 [The AKS Checklist](https://www.the-aks-checklist.com/) ([GitHub: lgmorand/aks-checklist](https://github.com/lgmorand/aks-checklist))의 공개 항목(High/Medium 우선순위, 총 124개)에, 체크리스트 사이트에는 없는 최신 AKS 기능(Deployment Safeguards, Image Cleaner, Node Autoprovisioning, VPA, KEDA, ACR Task 자동 리빌드 등 9개) 기반 항목을 추가하여 재구성한 것이다.

> ⚠️ **목적**: 특정 마이그레이션(App Routing 등)에 한정되지 않은, **현재 운영 중인 AKS 클러스터가 일반적인 프로덕션 모범사례를 준수하고 있는지** 점검하기 위한 범용 체크리스트다.

## 사용 방법
- 항목은 **우선순위(High → Medium) 순으로 정렬**했다 (카테고리 순서가 아님). 각 항목 하단의 뱃지로 원본 카테고리와 태그를 표시한다.
- 각 항목은 `제목 / 설명 / 확인 방법(az cli 우선, 필요 시 kubectl) / 권장 방식으로 변경하는 명령어` 형식이다.
- 모든 `az cli` 명령어는 **실제 `az <command> --help` 실행 결과로 플래그를 검증**했다 (2026-07 기준 azure-cli 2.86.0). `[Preview]` 표시가 있는 기능은 GA 이전이므로 프로덕션 적용 전 별도 검토가 필요하다.
- kubectl/YAML로만 확인·변경 가능한 애플리케이션 레벨 항목(리소스 request/limit, probe 등)은 az cli 대신 kubectl 명령/YAML 예시를 제공한다.
- CLI만으로 판별 불가능한 프로세스/조직 차원의 항목(예: CI/CD 도입, DR 훈련 수행 여부)은 그 사실을 명시했다.

## 원본 항목 분포 (카테고리별, High+Medium 기준)
| 카테고리 | 항목 수 |
|---|---|
| Operations | 23 |
| Networking | 19 |
| Application | 17 |
| Cluster Security | 15 |
| Windows | 11 |
| BC/DR | 8 |
| Storage | 8 |
| Container | 7 |
| Identity | 7 |
| Resource Management | 6 |
| Cluster Multi | 3 |
| **소계** | **124** |
| **추가 항목** (체크리스트 사이트에 없는 최신 기능) | **9** |
| **총계** | **133** |

---

## [High] 노드 이미지 업그레이드를 정기적으로 수행하거나 AKS 자동 업그레이드를 사용하세요

**설명**: 노드 이미지에는 OS 패치와 컨테이너 런타임 보안 수정이 함께 포함되므로, 이를 오래 미루면 취약점과 운영 편차가 빠르게 누적됩니다. 운영 클러스터에서는 주간 단위 점검 또는 Node OS 자동 업그레이드 채널을 사용해 패치 적용을 상시화하는 편이 안전합니다.

**확인 방법 (az cli)**:
```bash
az aks show -g $RESOURCE_GROUP -n $CLUSTER --query "autoUpgradeProfile.nodeOsUpgradeChannel" -o tsv
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.osImage}{"\t"}{.status.nodeInfo.containerRuntimeVersion}{"\n"}{end}'
```

**권장 방식으로 변경**:
```bash
az aks update -g $RESOURCE_GROUP -n $CLUSTER --node-os-upgrade-channel NodeImage
```

`OPERATIONS` `ALL` `SECURITY` `WINDOWS`
---

## [High] 로그 수집/집계 도구를 프로비저닝하세요

**설명**: 중앙 로그 수집 체계가 없으면 노드, 시스템 파드, 애플리케이션에서 무슨 일이 일어났는지 사건 후 재구성하기가 매우 어렵습니다. 최소한 클러스터 전체 로그를 한 곳으로 모으고 보존해야 운영·장애 대응·감사 요구사항을 동시에 충족할 수 있습니다.

**확인 방법 (az cli)**:
```bash
az aks show -g $RESOURCE_GROUP -n $CLUSTER --query "addonProfiles.omsagent.enabled" -o tsv
kubectl get ds -n kube-system
```

**권장 방식으로 변경**:
```bash
az aks update -g $RESOURCE_GROUP -n $CLUSTER --enable-azure-monitor-logs --workspace-resource-id $WORKSPACE_ID
```

`OPERATIONS` `ALL` `MONITORING`
---

## [High] Container Insights(또는 Prometheus 등)로 클러스터 메트릭을 모니터링하세요

**설명**: CPU/메모리 같은 기본 텔레메트리만 봐서는 포화 징후, 워크로드별 병목, 사용자 정의 메트릭 이상을 충분히 파악하기 어렵습니다. 운영 클러스터는 메트릭 수집, 저장, 시각화(Grafana 등)까지 이어지는 경로를 갖춰야 용량 계획과 장애 예측이 가능합니다.

**확인 방법 (az cli)**:
```bash
AKS_ID=$(az aks show -g $RESOURCE_GROUP -n $CLUSTER --query id -o tsv)
az aks show -g $RESOURCE_GROUP -n $CLUSTER --query "azureMonitorProfile.metrics.enabled" -o tsv
az monitor metrics list-definitions --resource "$AKS_ID" -o table
```

**권장 방식으로 변경**:
```bash
az aks update -g $RESOURCE_GROUP -n $CLUSTER --enable-azure-monitor-metrics --azure-monitor-workspace-resource-id $AMW_ID --grafana-resource-id $GRAFANA_ID
```

`OPERATIONS` `ALL` `MONITORING`
---

## [High] GitOps로 워크로드를 배포하세요

**설명**: GitOps는 선언형 구성을 Git에 고정해 변경 이력, 승인 절차, 롤백 근거를 모두 남길 수 있게 해줍니다. 운영자가 클러스터에서 직접 수작업 변경을 하는 빈도를 줄여 드리프트를 방지하고, 재현 가능한 배포 체계를 만들 수 있습니다.

**확인 방법 (az cli)**:
```bash
az k8s-configuration flux list -g $RESOURCE_GROUP -c $CLUSTER --cluster-type managedClusters -o table
```

**권장 방식으로 변경**:
```bash
az k8s-configuration flux create -g $RESOURCE_GROUP -c $CLUSTER --cluster-type managedClusters -n cluster-flux -u $REPO_URL --branch main --scope cluster --namespace flux-system
```

`OPERATIONS` `ALL`
---

## [High] CI/CD로 워크로드를 배포하세요

**설명**: 운영 배포를 사람의 수동 kubectl 실행에 의존하면 승인 누락, 재현 불가, 환경별 편차가 쉽게 발생합니다. 빌드·테스트·배포를 파이프라인으로 묶으면 변경 품질을 일정하게 유지하고, 배포 이력과 실패 지점을 추적하기 쉬워집니다.

**확인 방법 (az cli)**:
```bash
# GitHub Actions / Azure DevOps / Jenkins 등의 파이프라인 정의를 저장소에서 확인해야 합니다.
# az CLI만으로 AKS 배포의 CI/CD 도입 여부를 정확히 판별할 수 없습니다.
```

**권장 방식으로 변경**:
```bash
# 배포 파이프라인에서 이미지 빌드, 보안 검사, 매니페스트 검증, kubectl/helm/flux 배포를 자동화해야 합니다.
# 저장소와 빌드 시스템 구성 변경이 필요하며, 단일 az CLI 명령으로 대체할 수 없습니다.
```

`OPERATIONS` `ALL`
---

## [High] 마스터 로그(API 서버 로그)를 Azure Monitor 또는 선호하는 로그 관리 솔루션으로 전송하세요

**설명**: 컨트롤 플레인 로그가 없으면 인증 실패, 권한 거부, 스케줄링 이상, API 서버 오류 같은 핵심 운영 신호를 사후에 복원할 방법이 거의 없습니다. 특히 장애 분석이나 Microsoft 지원 요청 시에도 제어 영역 로그가 있어야 원인 파악 속도가 크게 빨라집니다.

**확인 방법 (az cli)**:
```bash
AKS_ID=$(az aks show -g $RESOURCE_GROUP -n $CLUSTER --query id -o tsv)
az monitor diagnostic-settings list --resource "$AKS_ID" -o json
```

**권장 방식으로 변경**:
```bash
AKS_ID=$(az aks show -g $RESOURCE_GROUP -n $CLUSTER --query id -o tsv)
az monitor diagnostic-settings create --resource "$AKS_ID" -n aks-control-plane-logs --workspace $WORKSPACE_ID --export-to-resource-specific true --logs '[{"category":"kube-apiserver","enabled":true},{"category":"kube-audit","enabled":true},{"category":"kube-audit-admin","enabled":true},{"category":"guard","enabled":true},{"category":"cluster-autoscaler","enabled":true}]'
```

`OPERATIONS` `ALL` `MONITORING`
---

## [High] 너무 크지도 작지도 않은 적절한 노드 크기를 선택하세요

**설명**: 노드가 지나치게 크면 예약 단위가 커져 빈 리소스가 많아지고, 지나치게 작으면 시스템 파드와 애플리케이션 파드가 자주 자원 경쟁을 일으킵니다. 실제 사용률과 워크로드 특성에 맞춰 노드 크기를 조정해야 비용, 성능, 가용성을 균형 있게 맞출 수 있습니다.

**확인 방법 (az cli)**:
```bash
az aks nodepool list -g $RESOURCE_GROUP --cluster-name $CLUSTER --query "[].{pool:name,vmSize:vmSize,count:count,mode:mode}" -o table
kubectl top nodes
```

**권장 방식으로 변경**:
```bash
az aks nodepool add -g $RESOURCE_GROUP --cluster-name $CLUSTER -n $NEW_NODEPOOL --node-vm-size $TARGET_VM_SIZE --node-count 3 --mode User
```

`OPERATIONS` `ALL` `COST` `FINOPS` `MONITORING`
---

## [High] Container Insights(또는 Telegraf/ElasticSearch 등)로 클러스터 로그를 저장하고 분석하세요

**설명**: 로그를 단순 저장만 하고 분석 체계를 만들지 않으면 장애 패턴, 반복 오류, 성능 저하 전조를 찾아내기 어렵습니다. 운영 환경에서는 수집된 로그를 검색·상관분석·보존 정책과 함께 관리해 사후 분석과 선제 대응 모두에 활용해야 합니다.

**확인 방법 (az cli)**:
```bash
az monitor log-analytics workspace table list -g $WORKSPACE_RG --workspace-name $WORKSPACE --query "[?name=='ContainerLogV2' || name=='KubePodInventory' || name=='KubeNodeInventory'].name" -o table
kubectl get ds ama-logs -n kube-system
```

**권장 방식으로 변경**:
```bash
az aks update -g $RESOURCE_GROUP -n $CLUSTER --enable-azure-monitor-logs --workspace-resource-id $WORKSPACE_ID
```

`OPERATIONS` `ALL` `MONITORING`
---

## [High] 가장 중요한 메트릭에 대한 경고를 구성하세요

**설명**: 대시보드를 사람이 계속 보고 있을 수는 없기 때문에, 임계 메트릭에는 반드시 능동적인 경고가 필요합니다. 알림이 없으면 장애를 사용자 신고로 먼저 알게 되거나, 포화·오류율 증가 같은 징후를 놓쳐 복구 시간이 길어집니다.

**확인 방법 (az cli)**:
```bash
AKS_ID=$(az aks show -g $RESOURCE_GROUP -n $CLUSTER --query id -o tsv)
az monitor metrics alert list -g $RESOURCE_GROUP -o table
az monitor metrics list-definitions --resource "$AKS_ID" -o table
```

**권장 방식으로 변경**:
```bash
AKS_ID=$(az aks show -g $RESOURCE_GROUP -n $CLUSTER --query id -o tsv)
az monitor action-group create -g $RESOURCE_GROUP -n aks-ops-ag --short-name aksops --action email ops $ALERT_EMAIL
az monitor metrics alert create -g $RESOURCE_GROUP -n aks-critical-metric-alert --scopes "$AKS_ID" --condition "avg METRIC_NAME > 80" --window-size 5m --evaluation-frequency 1m --severity 2 --action aks-ops-ag
```

`OPERATIONS` `ALL` `MONITORING` `SECURITY`
---

## [High] AKS 클러스터의 Resource Health 알림을 구독하세요

**설명**: 플랫폼 측 장애나 Azure 리소스 상태 이상은 클러스터 내부 지표만으로는 즉시 보이지 않을 수 있습니다. Resource Health 알림을 연결해 두면 Azure 측 문제를 더 빨리 인지하고, 운영팀과 이해관계자에게 즉시 전파할 수 있습니다.

**확인 방법 (az cli)**:
```bash
AKS_ID=$(az aks show -g $RESOURCE_GROUP -n $CLUSTER --query id -o tsv)
az monitor activity-log alert list -g $RESOURCE_GROUP -o table
```

**권장 방식으로 변경**:
```bash
AKS_ID=$(az aks show -g $RESOURCE_GROUP -n $CLUSTER --query id -o tsv)
az monitor action-group create -g $RESOURCE_GROUP -n aks-rh-ag --short-name aksrh --action email ops $ALERT_EMAIL
az monitor activity-log alert create -g $RESOURCE_GROUP -n aks-resource-health --scope "$AKS_ID" --condition category=ResourceHealth --action-group aks-rh-ag
```

`OPERATIONS` `ALL` `MONITORING` `SECURITY`
---

## [High] 노드 리소스 그룹(인프라 RG)에서 운영자 변경이 발생하지 않도록 거버넌스를 마련하세요

**설명**: AKS가 관리하는 노드 리소스 그룹에 수동 변경이 들어가면 클러스터 드리프트와 지원 불가 상태를 동시에 초래할 수 있습니다. 운영자는 이 영역을 '직접 수정 금지' 구역으로 다루고, RBAC와 프로세스로 변경 권한을 엄격히 제한해야 합니다.

**확인 방법 (az cli)**:
```bash
NODE_RG=$(az aks show -g $RESOURCE_GROUP -n $CLUSTER --query nodeResourceGroup -o tsv)
az role assignment list --scope "$(az group show -n "$NODE_RG" --query id -o tsv)" --include-inherited -o table
```

**권장 방식으로 변경**:
```bash
NODE_RG=$(az aks show -g $RESOURCE_GROUP -n $CLUSTER --query nodeResourceGroup -o tsv)
az role assignment create --assignee-object-id $AAD_GROUP_OBJECT_ID --assignee-principal-type Group --role Reader --scope "$(az group show -n "$NODE_RG" --query id -o tsv)"
```

`OPERATIONS` `ALL` `MONITORING`
---

## [High] 클러스터에 디버깅 도구를 설치하세요

**설명**: 장애가 발생한 뒤에 디버깅 환경을 급히 만들려고 하면 원인 분석 시간이 길어지고, 보안 검토가 안 된 임시 도구가 들어오기 쉽습니다. 네트워크·DNS·패킷·프로세스 점검용 툴박스를 미리 준비해 두면 MTTR을 크게 줄일 수 있습니다.

**확인 방법 (az cli)**:
```bash
az aks command invoke -g $RESOURCE_GROUP -n $CLUSTER -c "kubectl get pods -A | egrep 'netshoot|busybox|toolbox|inspektor-gadget'"
```

**권장 방식으로 변경**:
```bash
az aks command invoke -g $RESOURCE_GROUP -n $CLUSTER -c "kubectl create namespace ops-tools || true; kubectl -n ops-tools run netshoot --image=nicolaka/netshoot --restart=Never -- sleep 3600"
```

`OPERATIONS` `ALL` `MONITORING`
---

## [High] 클러스터를 물리적으로 격리

**설명**: 규제, 신뢰 경계, 데이터 주권, 강한 성능 격리가 필요한 워크로드는 전용 클러스터가 필요할 수 있습니다. 다만 모든 팀이나 애플리케이션마다 별도 클러스터를 만들면 업그레이드, 네트워크, 보안 운영 비용이 급격히 늘어나므로 실제 물리 분리 기준을 명확히 정의해 사용하는 것이 중요합니다.

**확인 방법 (az cli)**:
```bash
az aks list -o table
```

**권장 방식으로 변경**:
```bash
# 규제/신뢰 경계가 강한 워크로드만 전용 클러스터로 분리합니다.
az aks create -g $RESOURCE_GROUP -n $CLUSTER-dedicated --enable-private-cluster --network-plugin azure --network-plugin-mode overlay
```

`CLUSTER MULTI` `ALL`
---

## [High] Azure CNI 사용 시 최대 Pod 수를 고려해 서브넷 크기 산정

**설명**: Azure CNI는 노드 수뿐 아니라 노드당 최대 Pod 수까지 IP 소비량에 직접 반영되므로 서브넷을 작게 잡으면 스케일아웃과 업그레이드가 막힙니다. 특히 오토스케일러, 버퍼 노드, 신규 노드풀 추가까지 감안해 여유 IP를 충분히 남겨 두는지 확인해야 합니다.

**확인 방법 (az cli)**:
```bash
az aks show -g $RESOURCE_GROUP -n $CLUSTER --query "{plugin:networkProfile.networkPlugin,pools:agentPoolProfiles[].{pool:name,nodeCount:count,maxPods:maxPods,subnet:vnetSubnetId}}" -o json
az network vnet subnet show --ids $AKS_SUBNET_ID --query "{prefix:addressPrefix,prefixes:addressPrefixes}" -o json
```

**권장 방식으로 변경**:
```bash
# 기존 서브넷이 부족하면 더 큰 서브넷에 새 노드풀을 만들고 워크로드를 이전합니다.
az aks nodepool add -g $RESOURCE_GROUP --cluster-name $CLUSTER -n np2 --vnet-subnet-id $NEW_NODE_SUBNET_ID --max-pods 50
```

`NETWORKING` `ALL` `NETWORK`
---

## [High] 불필요한 인터넷 공개 Load Balancer 사용 금지

**설명**: 내부 전용 서비스까지 공인 Load Balancer로 노출하면 공격 표면이 불필요하게 넓어지고 공인 IP 관리 부담도 증가합니다. 내부 사용자나 사설 네트워크에서만 접근하면 되는 서비스는 Internal Load Balancer 또는 사설 Ingress로 제한하는 것이 기본 원칙입니다.

**확인 방법 (az cli)**:
```bash
az aks get-credentials -g $RESOURCE_GROUP -n $CLUSTER
kubectl get svc -A -o jsonpath='{range .items[?(@.spec.type=="LoadBalancer")]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.metadata.annotations.service\.beta\.kubernetes\.io/azure-load-balancer-internal}{"\t"}{.status.loadBalancer.ingress[*].ip}{"\n"}{end}'
NODE_RG=$(az aks show -g $RESOURCE_GROUP -n $CLUSTER --query nodeResourceGroup -o tsv)
az network public-ip list -g "$NODE_RG" --query "[].{name:name,ip:ipAddress}" -o table
```

**권장 방식으로 변경**:
```bash
kubectl annotate svc -n $NAMESPACE $SERVICE service.beta.kubernetes.io/azure-load-balancer-internal="true" --overwrite
```

`NETWORKING` `ALL` `SECURITY` `NETWORK`
---

## [High] 요구사항이 있으면 Private Cluster 사용

**설명**: API 서버를 인터넷에 노출하지 않으면 관리 평면 공격 표면이 크게 줄고, 접근 경로를 사설 네트워크와 승인된 관리 지점으로 한정할 수 있습니다. 보안·규제 요구가 강한 프로덕션 환경에서는 Private Cluster가 기본 검토 항목이어야 합니다.

**확인 방법 (az cli)**:
```bash
az aks show -g $RESOURCE_GROUP -n $CLUSTER --query "{privateCluster:apiServerAccessProfile.enablePrivateCluster,privateDNSZone:apiServerAccessProfile.privateDNSZone}" -o json
```

**권장 방식으로 변경**:
```bash
az aks update -g $RESOURCE_GROUP -n $CLUSTER --enable-private-cluster
```

`NETWORKING` `ALL` `SECURITY` `NETWORK`
---

## [High] Azure CNI 사용 시 노드당 최대 Pod 수 점검 (기본값 30)

**설명**: maxPods 값은 노드당 IP 예약량과 스케줄링 밀도를 동시에 좌우합니다. 기본값 30이 항상 맞는 것은 아니므로, 워크로드 특성과 서브넷 여유를 함께 고려해 너무 낮아 병목이 생기거나 너무 높아 IP가 낭비되지 않는지 검토해야 합니다.

**확인 방법 (az cli)**:
```bash
az aks show -g $RESOURCE_GROUP -n $CLUSTER --query "agentPoolProfiles[].{pool:name,maxPods:maxPods}" -o table
```

**권장 방식으로 변경**:
```bash
# 기존 노드풀 값 변경이 어려우면 적정 max-pods로 새 노드풀을 추가 후 이전합니다.
az aks nodepool add -g $RESOURCE_GROUP --cluster-name $CLUSTER -n npmax --max-pods 50
```

`NETWORKING` `ALL` `IPAM` `NETWORK`
---

## [High] 적절한 Liveness Probe 구현

**설명**: 장시간 실행되는 애플리케이션은 데드락, 스레드 고갈, 외부 의존성 고착 같은 상태에 빠질 수 있습니다. liveness probe는 이런 파드를 자동으로 재시작하게 해 복구 시간을 줄이지만, 값이 너무 공격적이면 정상 파드도 재시작 루프에 들어갈 수 있으므로 실제 장애 패턴에 맞춰 조정해야 합니다.

**확인 방법**:
```bash
kubectl get deploy -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\n"}{range .spec.template.spec.containers[*]}{"  - "}{.name}{": liveness="}{.livenessProbe}{"\n"}{end}{end}'
```

**권장 방식으로 변경**:
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 8080
  initialDelaySeconds: 15
  periodSeconds: 10
  timeoutSeconds: 2
  failureThreshold: 3
```

`APPLICATION` `ALL`
---

## [High] 비밀값은 Azure Key Vault에 저장하고 Docker 이미지에 포함하지 않기

**설명**: 비밀번호나 연결 문자열을 이미지에 bake-in 하거나 일반 환경 변수로만 주입하면 유출 시 회수와 회전이 매우 어렵습니다. Azure Key Vault와 CSI Driver를 사용하면 비밀값 수명주기, 접근 제어, 감사 로그를 중앙화하면서 애플리케이션 배포와 비밀 배포를 분리할 수 있습니다.

**확인 방법**:
```bash
kubectl get secretproviderclass -A
kubectl get deploy -A -o yaml | grep -nE 'secrets-store.csi.k8s.io|secretKeyRef|envFrom:'
```

**권장 방식으로 변경**:
```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: app-secrets
  namespace: prod
spec:
  provider: azure
  parameters:
    keyvaultName: <KEYVAULT_NAME>
    tenantId: <TENANT_ID>
    objects: |
      array:
        - |
          objectName: db-password
          objectType: secret
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: prod
spec:
  template:
    spec:
      serviceAccountName: api-sa
      containers:
      - name: api
        volumeMounts:
        - name: kv-secrets
          mountPath: /mnt/secrets-store
          readOnly: true
      volumes:
      - name: kv-secrets
        csi:
          driver: secrets-store.csi.k8s.io
          readOnly: true
          volumeAttributes:
            secretProviderClass: app-secrets
```

`APPLICATION` `ALL` `SECURITY`
---

## [High] 컨테이너 requests와 limits 설정

**설명**: requests가 없으면 스케줄러가 정확히 배치 결정을 내리기 어렵고, limits가 없으면 한 워크로드가 노드 자원을 독점해 다른 서비스까지 불안정하게 만들 수 있습니다. 운영 환경에서는 최소/최대 자원을 명시해 QoS를 분명히 하고, HPA/VPA가 의미 있는 지표를 보도록 해야 합니다.

**확인 방법**:
```bash
kubectl get deploy -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\n"}{range .spec.template.spec.containers[*]}{"  - "}{.name}{": requests="}{.resources.requests}{", limits="}{.resources.limits}{"\n"}{end}{end}'
```

**권장 방식으로 변경**:
```yaml
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

`APPLICATION` `ALL` `RESOURCES` `MULTI-TENANCY`
---

## [High] Dockerfile 스캐닝으로 이미지 보안 모범 사례 준수

**설명**: 취약한 베이스 이미지, latest 태그, root 사용자, 불필요한 패키지 설치 같은 문제는 Dockerfile 단계에서 이미 시작됩니다. Dockerfile을 빌드 전에 정적 점검하면 운영에 올라가기 전에 보안 부채를 조기에 제거할 수 있습니다.

**확인 방법**:
```bash
find . -name 'Dockerfile*' -print
grep -RInE 'FROM .+:latest|USER root|ADD |curl .*\| *sh' .
```

**권장 방식으로 변경**:
```yaml
name: dockerfile-lint
on: [pull_request]
jobs:
  hadolint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hadolint/hadolint-action@v3.1.0
        with:
          dockerfile: Dockerfile
```

`APPLICATION` `ALL` `SECURITY`
---

## [High] 빌드 시 Docker 이미지 정적 분석 수행

**설명**: 이미지가 레지스트리에 푸시된 뒤에 취약점을 찾는 것보다, 빌드 단계에서 바로 실패시키는 편이 수정 비용과 배포 위험이 훨씬 낮습니다. 운영 기준에서는 OS 패키지와 애플리케이션 의존성을 함께 스캔하고 결과를 빌드 아티팩트로 남겨야 합니다.

**확인 방법**:
```bash
az security pricing show --name Containers --output json
grep -RInE 'trivy|grype|twistcli|anchore' .github/workflows azure-pipelines.yml .
```

**권장 방식으로 변경**:
```yaml
- name: Build image
  run: docker build -t <ACR_NAME>.azurecr.io/api:${GITHUB_SHA} .
- name: Scan image before push
  uses: aquasecurity/trivy-action@0.28.0
  with:
    image-ref: <ACR_NAME>.azurecr.io/api:${GITHUB_SHA}
    severity: HIGH,CRITICAL
    exit-code: '1'
    ignore-unfixed: true
```

`APPLICATION` `ALL` `SECURITY`
---

## [High] Docker 이미지 빌드 컴플라이언스 정책 적용

**설명**: 루트 사용자 실행, 과도한 capability, 금지 포트 노출, 시크릿 파일 포함 여부는 단순 CVE 스캔만으로는 놓치기 쉽습니다. 컴플라이언스 검사를 빌드 파이프라인에 넣어야 보안 기준에 맞지 않는 이미지를 레지스트리와 클러스터 앞단에서 차단할 수 있습니다.

**확인 방법**:
```bash
grep -RInE 'hadolint|dockle|conftest|checkov|opa' .github/workflows azure-pipelines.yml .
```

**권장 방식으로 변경**:
```yaml
- name: Lint Dockerfile
  uses: hadolint/hadolint-action@v3.1.0
  with:
    dockerfile: Dockerfile
- name: Enforce image compliance
  run: |
    dockle --exit-code 1 --exit-level warn <ACR_NAME>.azurecr.io/api:${GITHUB_SHA}
```

`APPLICATION` `ALL` `SECURITY`
---

## [High] 컨테이너 이미지 취약점 스캔

**설명**: 실제로 배포되는 아티팩트는 소스 코드가 아니라 이미지이므로, 레지스트리 기준의 취약점 스캔이 없으면 운영 반입 전 마지막 방어선이 비어 있게 됩니다. 초기 스캔뿐 아니라 새 CVE 공개 시 재평가가 가능해야 오래 저장된 이미지의 드리프트도 잡을 수 있습니다.

**확인 방법**:
```bash
az security pricing show --name Containers --output json
az acr task list --registry <ACR_NAME> --output table
```

**권장 방식으로 변경**:
```bash
az security pricing create --name Containers --tier standard --extensions name=ContainerRegistriesVulnerabilityAssessments isEnabled=True
trivy image --severity HIGH,CRITICAL --exit-code 1 <ACR_NAME>.azurecr.io/<REPOSITORY>:<TAG>
```

`CONTAINER` `ALL` `SECURITY`
---

## [High] 애플리케이션 런타임 보안 적용

**설명**: 빌드 시점 스캔만으로는 실제 실행 중 발생하는 비정상 프로세스, 권한 상승, 악성 네트워크 연결, 파일 시스템 변조를 막을 수 없습니다. 운영 클러스터는 런타임 탐지와 경보 체계를 추가해 침해 후 행동을 조기에 포착해야 합니다.

**확인 방법**:
```bash
az security pricing show --name Containers --output json
kubectl get ds -A | grep -Ei 'falco|defender'
```

**권장 방식으로 변경**:
```bash
az security pricing create --name Containers --tier standard
helm upgrade --install falco falcosecurity/falco --namespace falco --create-namespace
```

`CONTAINER` `ALL` `SECURITY`
---

## [High] 문제 발견된 레지스트리 이미지는 격리

**설명**: 이미지가 레지스트리에 저장된 뒤에도 새로운 CVE나 공급망 문제가 뒤늦게 발견될 수 있습니다. 격리 정책이 없으면 이미 승인된 태그가 그대로 계속 배포되어, 시간이 지날수록 보안 기준이 무너지는 드리프트가 생깁니다.

**확인 방법**:
```bash
ACR_ID=$(az acr show --name <ACR_NAME> --query id --output tsv)
az resource show --ids "$ACR_ID" --query properties.policies.quarantinePolicy --output json
az acr repository show-tags --name <ACR_NAME> --repository <REPOSITORY> --detail --output table
```

**권장 방식으로 변경**:
```bash
ACR_ID=$(az acr show --name <ACR_NAME> --query id --output tsv)
az resource update --ids "$ACR_ID" --set properties.policies.quarantinePolicy.status=enabled
```

`CONTAINER` `ALL` `SECURITY`
---

## [High] Docker Registry에 RBAC 적용

**설명**: 레지스트리에 광범위한 push/delete 권한을 주면 이미지 변조, 실수 삭제, 감사 추적 불가 같은 문제가 발생합니다. ACR은 AcrPull, AcrPush 같은 내장 역할을 제공하므로 사용자와 워크로드별로 최소 권한을 분리해 할당해야 합니다.

**확인 방법**:
```bash
ACR_ID=$(az acr show --name <ACR_NAME> --query id --output tsv)
az role assignment list --scope "$ACR_ID" --output table
```

**권장 방식으로 변경**:
```bash
ACR_ID=$(az acr show --name <ACR_NAME> --query id --output tsv)
az role assignment create --assignee-object-id <PRINCIPAL_OBJECT_ID> --assignee-principal-type ServicePrincipal --role AcrPull --scope "$ACR_ID"
```

`CONTAINER` `ALL` `SECURITY`
---

## [High] Docker Registry 네트워크 분리

**설명**: ACR이 퍼블릭 엔드포인트로 그대로 노출되면 인터넷 경유 접근, 토큰 오남용, 데이터 유출 경로가 늘어납니다. Private Link와 공용 네트워크 차단을 적용하면 레지스트리 접근 경로를 VNet 내부와 Microsoft 백본으로 제한할 수 있습니다.

**확인 방법**:
```bash
ACR_ID=$(az acr show --name <ACR_NAME> --query id --output tsv)
az acr show --name <ACR_NAME> --output json
az network private-endpoint-connection list --id "$ACR_ID" --output table
```

**권장 방식으로 변경**:
```bash
ACR_ID=$(az acr show --name <ACR_NAME> --query id --output tsv)
az network private-endpoint create --name acr-pe --resource-group <RG> --vnet-name <VNET_NAME> --subnet <SUBNET_NAME> --private-connection-resource-id "$ACR_ID" --group-id registry --connection-name acr-registry-conn
az acr update --name <ACR_NAME> --public-network-enabled false
```

`CONTAINER` `ALL` `SECURITY` `NETWORK`
---

## [High] Kubernetes 버전을 최신 지원 범위로 유지

**설명**: 지원 종료된 Kubernetes 버전은 보안 패치와 버그 수정이 끊기므로 취약점 대응 시간이 급격히 늘어납니다. 운영 클러스터는 현재 버전과 업그레이드 가능 버전을 함께 확인하고, 자동 업그레이드 채널까지 정해 두는 것이 안전합니다.

**확인 방법 (az cli)**:
```bash
az aks show --resource-group <RG> --name <AKS_NAME> --query "kubernetesVersion" --output tsv
az aks get-upgrades --resource-group <RG> --name <AKS_NAME> --output table
```

**권장 방식으로 변경**:
```bash
az aks upgrade --resource-group <RG> --name <AKS_NAME> --kubernetes-version <TARGET_VERSION> --yes
az aks update --resource-group <RG> --name <AKS_NAME> --auto-upgrade-channel stable --node-os-upgrade-channel NodeImage
```

`CLUSTER SECURITY` `ALL` `SECURITY` `OPERATIONS`
---

## [High] Azure Key Vault 사용

**설명**: 시크릿을 Kubernetes Secret에만 장기 저장하면 회전 주기 관리와 접근 통제가 느슨해지기 쉽습니다. Key Vault와 Secrets Store CSI Driver, Workload Identity를 조합하면 애플리케이션이 필요한 시점에만 비밀을 읽고 회전도 자동화할 수 있습니다.

**확인 방법 (az cli)**:
```bash
az aks addon show --resource-group <RG> --name <AKS_NAME> --addon azure-keyvault-secrets-provider --output yaml
az aks show --resource-group <RG> --name <AKS_NAME> --query "oidcIssuerProfile" --output yaml
kubectl get secretproviderclass -A
```

**권장 방식으로 변경**:
```bash
az aks update --resource-group <RG> --name <AKS_NAME> --enable-oidc-issuer --enable-workload-identity
az aks addon enable --resource-group <RG> --name <AKS_NAME> --addon azure-keyvault-secrets-provider --enable-secret-rotation --rotation-poll-interval 2m
```

`CLUSTER SECURITY` `ALL` `SECURITY` `OPERATIONS`
---

## [High] 클러스터에서 취약한 이미지 캐시 제거

**설명**: 이미 배포에서 제외된 취약 이미지라도 노드 캐시에 남아 있으면 재사용 위험과 디스크 압박을 동시에 키웁니다. Image Cleaner를 켜면 쓰지 않는 취약 이미지 정리를 정기화해 운영 중 노드 위생 상태를 유지할 수 있습니다.

**확인 방법 (az cli)**:
```bash
az aks show --resource-group <RG> --name <AKS_NAME> --query "securityProfile.imageCleaner" --output yaml
kubectl get pods -n kube-system | grep image-cleaner
```

**권장 방식으로 변경**:
```bash
az aks update --resource-group <RG> --name <AKS_NAME> --enable-image-cleaner --image-cleaner-interval-hours 48
```

`CLUSTER SECURITY` `ALL` `SECURITY` `OPERATIONS`
---

## [High] Microsoft Defender for Containers 사용

**설명**: 운영 AKS는 이미지 취약점만이 아니라 런타임 이상 징후와 Kubernetes 위협 탐지도 함께 봐야 합니다. Defender for Containers를 켜 두면 배포 전후 위험을 하나의 운영 체계로 연결할 수 있어 사고 대응 속도가 좋아집니다.

**확인 방법 (az cli)**:
```bash
az security pricing show --name Containers --output jsonc
az aks show --resource-group <RG> --name <AKS_NAME> --query "securityProfile.defender" --output yaml
```

**권장 방식으로 변경**:
```bash
az security pricing create --name Containers --tier standard --subplan P2
az aks update --resource-group <RG> --name <AKS_NAME> --enable-defender --workspace-resource-id <LAW_RESOURCE_ID>
```

`CLUSTER SECURITY` `ALL` `SECURITY` `OPERATIONS`
---

## [High] Service Principal 사용 시 자격 증명 정기 교체

**설명**: 서비스 프린시펄은 장기 비밀이 남기 쉬워서 운영자가 교체 주기를 놓치면 곧바로 고위험 자산이 됩니다. Managed Identity로 전환하지 못한 상태라면 최소한 분기 단위 회전과 즉시 반영 절차를 자동화해야 합니다.

**확인 방법 (az cli)**:
```bash
az aks show --resource-group <RG> --name <AKS_NAME> --query "servicePrincipalProfile.clientId" --output tsv
```

**권장 방식으로 변경**:
```bash
az ad sp credential reset --id <SERVICE_PRINCIPAL_APP_ID> --append --years 1
az aks update-credentials --resource-group <RG> --name <AKS_NAME> --reset-service-principal --service-principal <SERVICE_PRINCIPAL_APP_ID> --client-secret <NEW_CLIENT_SECRET>
```

`CLUSTER SECURITY` `ALL` `SECRETS`
---

## [High] 이미지 레지스트리는 ACR 같은 프라이빗 레지스트리 사용

**설명**: 퍼블릭 레지스트리 의존도가 높으면 공급망 통제, 네트워크 안정성, 이미지 승인 절차가 모두 느슨해집니다. 운영 클러스터는 프라이빗 레지스트리로 공급 경로를 고정하고 pull 권한도 클러스터 정체성과 분리해 관리해야 합니다.

**확인 방법 (az cli)**:
```bash
az aks check-acr --resource-group <RG> --name <AKS_NAME> --acr <ACR_LOGIN_SERVER>
az acr repository list --name <ACR_NAME> --output table
kubectl get deployments -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"/"}{.metadata.name}{"\t"}{range .spec.template.spec.containers[*]}{.image}{" "}{end}{"\n"}{end}'
```

**권장 방식으로 변경**:
```bash
az aks update --resource-group <RG> --name <AKS_NAME> --attach-acr <ACR_NAME_OR_RESOURCE_ID>
```

`CLUSTER SECURITY` `ALL` `COMPLIANCE`
---

## [High] 앱 분리 요구사항(namespace/nodepool/cluster) 정의

**설명**: 멀티테넌트 운영에서는 어느 수준에서 격리할지 미리 정하지 않으면 네임스페이스만으로 끝낼지, 전용 노드풀이나 전용 클러스터까지 갈지 계속 흔들리게 됩니다. 워크로드 민감도와 운영 책임 경계를 기준으로 분리 단위를 정하고 스케줄링 규칙까지 함께 적용해야 합니다.

**확인 방법 (az cli)**:
```bash
kubectl get ns
kubectl get networkpolicy -A
az aks nodepool list --resource-group <RG> --cluster-name <AKS_NAME> --output table
```

**권장 방식으로 변경**:
```bash
kubectl create namespace tenant-a
az aks nodepool add --resource-group <RG> --cluster-name <AKS_NAME> --name tenanta01 --node-vm-size Standard_D4ds_v5 --node-count 3 --mode User --labels tenant=tenant-a --node-taints tenant=tenant-a:NoSchedule
```

`CLUSTER SECURITY` `ALL` `COMPLIANCE`
---

## [High] Microsoft Entra ID 관리형 통합으로 인증 연동

**설명**: 운영 클러스터 접근을 개별 kubeconfig 파일이나 로컬 계정에 의존하면 계정 회수와 감사 추적이 어려워집니다. Entra ID 관리형 통합을 쓰면 사용자 인증을 중앙화하고, 퇴사·권한 변경 같은 운영 이벤트를 클러스터 접근에 바로 반영할 수 있습니다.

**확인 방법 (az cli)**:
```bash
az graph query -q "where type=='microsoft.containerservice/managedclusters' | extend compliant = isnotnull(properties.aadProfile) | distinct id,compliant" --output table
az aks show --resource-group <RG> --name <AKS_NAME> --query "aadProfile" --output yaml
```

**권장 방식으로 변경**:
```bash
az aks update --resource-group <RG> --name <AKS_NAME> --enable-aad --aad-admin-group-object-ids <ADMIN_GROUP_OBJECT_ID> --aad-tenant-id <TENANT_ID>
```

`IDENTITY` `ALL` `SECURITY`
---

## [High] Microsoft Entra ID RBAC로 권한 부여 연동

**설명**: 인증만 Entra ID로 하고 권한 부여를 별도로 관리하면 누가 어떤 리소스까지 건드릴 수 있는지 추적이 어렵습니다. Azure RBAC를 연결하면 Azure 역할 할당과 클러스터 접근 모델을 맞춰 운영 권한 검토를 단순화할 수 있습니다.

**확인 방법 (az cli)**:
```bash
az aks show --resource-group <RG> --name <AKS_NAME> --query "{managedAad:aadProfile.managed,azureRbac:aadProfile.enableAzureRbac}" --output yaml
az role assignment list --scope <AKS_RESOURCE_ID> --query "[?contains(roleDefinitionName, 'Azure Kubernetes Service')].[principalName,roleDefinitionName]" --output table
```

**권장 방식으로 변경**:
```bash
az aks update --resource-group <RG> --name <AKS_NAME> --enable-aad --enable-azure-rbac --aad-admin-group-object-ids <ADMIN_GROUP_OBJECT_ID> --aad-tenant-id <TENANT_ID>
```

`IDENTITY` `ALL` `SECURITY`
---

## [High] Service Principal 대신 Managed Identity 사용

**설명**: Managed Identity는 비밀값 보관과 회전 부담을 크게 줄여 운영 실수를 줄여 줍니다. 특히 장기 운영 클러스터는 컨트롤 플레인과 kubelet 권한을 서비스 프린시펄 대신 관리형 ID로 가져가는 편이 사고 범위를 줄이기 쉽습니다.

**확인 방법 (az cli)**:
```bash
az graph query -q "where type=='microsoft.containerservice/managedclusters' | extend compliant = (properties.servicePrincipalProfile.clientId=='msi') | distinct id,compliant" --output table
az aks show --resource-group <RG> --name <AKS_NAME> --query "servicePrincipalProfile.clientId" --output tsv
```

**권장 방식으로 변경**:
```bash
az aks update --resource-group <RG> --name <AKS_NAME> --enable-managed-identity
```

`IDENTITY` `ALL` `SECURITY`
---

## [High] 노드풀 스케일아웃용 구독 쿼터 확보

**설명**: 오토스케일러가 정상이어도 구독 쿼터가 부족하면 피크 시점에 노드가 추가되지 않아 장애로 이어질 수 있습니다. 운영 리뷰에서는 현재 사용량과 VM 패밀리별 한도를 함께 보고, 예상 최대치보다 여유 있게 확보하는 것이 핵심입니다.

**확인 방법 (az cli)**:
```bash
az vm list-usage --location <REGION> --output table
az quota show --scope "/subscriptions/<SUBSCRIPTION_ID>/providers/Microsoft.Compute/locations/<REGION>" --resource-name <VM_FAMILY_QUOTA_NAME> --output jsonc
```

**권장 방식으로 변경**:
```bash
az quota update --scope "/subscriptions/<SUBSCRIPTION_ID>/providers/Microsoft.Compute/locations/<REGION>" --resource-name <VM_FAMILY_QUOTA_NAME> --limit-object value=<NEW_LIMIT> --resource-type dedicated
```

`RESOURCE MANAGEMENT` `ALL` `SCALABILITY` `RESOURCES`
---

## [High] 가용성 SLA, RTO, RPO 같은 비기능 요구사항 정의

**설명**: 프로덕션 리뷰에서 먼저 정해야 하는 것은 기술이 아니라 목표 복구수준입니다. SLA·RTO·RPO가 없으면 어떤 백업 주기, 다중 리전 투자, 운영 자동화가 필요한지 판단할 수 없습니다. 현재 클러스터 설정이 이 목표를 충족하는지 숫자로 비교할 수 있어야 운영 의사결정이 쉬워집니다.

**확인 방법**:
```bash
az aks show -g $RG -n $CLUSTER --query "{tier:sku.tier,privateCluster:apiServerAccessProfile.enablePrivateCluster,networkPlugin:networkProfile.networkPlugin}" -o yaml
az dataprotection backup-policy list -g $VAULT_RG -v $VAULT_NAME -o table
```

**권장 방식으로 변경**:
```bash
az aks update -g $RG -n $CLUSTER --tier standard
az dataprotection backup-vault create -g $VAULT_RG -v $VAULT_NAME -l $LOCATION --storage-setting "[{type:'LocallyRedundant',datastore-type:'VaultStore'}]"
```

`BC/DR` `ALL` `RESILIENCY`
---

## [High] DR 테스트를 정기적으로 수행하고 화이트스페이스 배포를 연습

**설명**: 백업이 있어도 실제로 복구가 되는지는 별개의 문제입니다. 빈 환경에 클러스터와 애플리케이션을 다시 세우는 연습을 주기적으로 해 두면 IaC 누락, 이미지 의존성, 권한 문제를 운영 전에 드러낼 수 있습니다. 복구 절차는 문서가 아니라 실행 가능한 배포 파이프라인으로 검증되어야 합니다.

**확인 방법**:
```bash
az aks show -g $RG -n $CLUSTER --query "{name:name,version:kubernetesVersion,nodeResourceGroup:nodeResourceGroup}" -o yaml
kubectl get all -A
```

**권장 방식으로 변경**:
```bash
az aks create -g $DR_RG -n $DR_CLUSTER --tier standard --generate-ssh-keys
az aks nodepool add -g $DR_RG --cluster-name $DR_CLUSTER -n win22dr --os-type Windows --os-sku Windows2022 --node-count 1
```

`BC/DR` `ALL` `RESILIENCY` `DEVOPS`
---

## [High] Azure 리전이 지원하면 가용성 영역 사용

**설명**: 가용성 영역은 단일 데이터센터 장애가 곧바로 서비스 중단으로 이어지는 것을 줄여 줍니다. 특히 시스템 풀, 사용자 풀, 디스크 배치가 영역 전략과 함께 설계되어야 업그레이드·장애 복구 때도 분산 효과를 얻습니다. 영역을 쓰지 않는다면 그 이유와 대체 보호수단이 명확해야 합니다.

**확인 방법**:
```bash
az graph query -q "where type=='microsoft.containerservice/managedclusters' | extend compliant= isnotnull(zones) | distinct id,compliant" -o table
az vm list-skus -l $LOCATION --zone true --size $SYSTEM_VM_SIZE -o table
```

**권장 방식으로 변경**:
```bash
az aks create -g $RG -n $CLUSTER --zones 1 2 3 --generate-ssh-keys
az aks nodepool add -g $RG --cluster-name $CLUSTER -n userpoolz --zones 1 2 3 --node-vm-size Standard_D4s_v5
```

`BC/DR` `ALL` `RESILIENCY`
---

## [High] AKS Standard 티어 사용

**설명**: 프로덕션 클러스터는 비용보다 제어 플레인 가용성 보장이 더 중요합니다. Standard 티어는 더 높은 SLA를 제공하므로 장애 대응 기준이 명확하고, 내부 운영 승인에서도 설명이 쉽습니다. Free 티어를 그대로 두면 플랫폼 장애 시 허용 가능한 다운타임을 스스로 늘리는 셈이 됩니다.

**확인 방법**:
```bash
az graph query -q "where type=='microsoft.containerservice/managedclusters' | extend compliant = (sku.tier=='Paid') | distinct id,compliant" -o table
az aks show -g $RG -n $CLUSTER --query sku.tier -o tsv
```

**권장 방식으로 변경**:
```bash
az aks update -g $RG -n $CLUSTER --tier standard
```

`BC/DR` `ALL` `RESILIENCY`
---

## [High] Azure Backup for AKS로 클러스터 리소스와 영구 볼륨 보호

**설명**: AKS 백업은 YAML만 저장하는 수준을 넘어서 클러스터 리소스와 영구 볼륨 복구 시나리오를 함께 다뤄야 의미가 있습니다. 운영자 입장에서는 백업 성공 여부보다도 실제 복원 대상, 네임스페이스 범위, 스냅샷 정책이 명확한지가 중요합니다. 복구 테스트까지 묶어 관리형 백업 체계를 만드는 것이 가장 안전합니다.

**확인 방법**:
```bash
az dataprotection backup-instance list -g $VAULT_RG -v $VAULT_NAME -o table
az dataprotection backup-policy list -g $VAULT_RG -v $VAULT_NAME -o table
```

**권장 방식으로 변경**:
```bash
AKS_ID=$(az aks show -g $RG -n $CLUSTER --query id -o tsv)
POLICY_ID=$(az dataprotection backup-policy show -g $VAULT_RG -v $VAULT_NAME -n $POLICY_NAME --query id -o tsv)

az dataprotection backup-instance initialize-backupconfig \
  --datasource-type AzureKubernetesService \
  --include-cluster-scope-resources true \
  --snapshot-volumes true \
  --include-all-containers \
  --storage-account-resource-group $STAGING_RG \
  --storage-account-name $STAGING_STORAGE > aks-backup-config.json

az dataprotection backup-instance initialize \
  --datasource-type AzureKubernetesService \
  --datasource-id "$AKS_ID" \
  -l $LOCATION \
  --policy-id "$POLICY_ID" \
  --backup-configuration aks-backup-config.json \
  --friendly-name $CLUSTER \
  --snapshot-resource-group-name $SNAPSHOT_RG > aks-backup-instance.json

az dataprotection backup-instance create -g $VAULT_RG -v $VAULT_NAME --backup-instance aks-backup-instance.json
```

`BC/DR` `ALL` `RESILIENCY` `STORAGE`
---

## [High] 워크로드에 맞는 스토리지 유형 선택

**설명**: 블록 스토리지, 파일 공유, 오브젝트 스토리지는 성능과 공유 모델이 다르기 때문에 하나로 통일하면 오히려 장애가 늘어납니다. DB처럼 지연 시간에 민감한 워크로드는 Azure Disk CSI, 다중 마운트가 필요한 경우는 Azure Files CSI처럼 용도에 맞춰 나눠야 합니다. 스토리지 타입 선택이 잘못되면 애플리케이션 튜닝으로 해결되지 않는 병목이 생깁니다.

**확인 방법**:
```bash
kubectl get storageclass
kubectl get pvc -A
```

**권장 방식으로 변경**:
```bash
cat <<'EOF2' > storageclasses.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-csi-premium
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: azurefile-csi-premium
provisioner: file.csi.azure.com
parameters:
  skuName: Premium_LRS
allowVolumeExpansion: true
mountOptions:
  - mfsymlinks
  - cache=strict
volumeBindingMode: Immediate
EOF2
kubectl apply -f storageclasses.yaml
```

`STORAGE` `ALL`
---

## [High] 클러스터 내부에 상태를 두지 말고 외부(Azure Storage, SQL, Cosmos 등)에 저장

**설명**: Pod나 노드 수명주기와 함께 사라지는 상태를 클러스터 내부에 두면 스케일아웃과 복구가 어려워집니다. 운영 리뷰에서는 StatefulSet이 꼭 필요한지, 단순 파일·세션·큐를 굳이 클러스터 안에 저장하고 있지 않은지 먼저 봐야 합니다. 가능하면 관리형 PaaS로 분리해 데이터 복제와 백업 책임을 플랫폼으로 넘기는 편이 안정적입니다.

**확인 방법**:
```bash
kubectl get statefulset,pvc -A
kubectl get deployment -A
```

**권장 방식으로 변경**:
```bash
kubectl set env deployment/myapp AZURE_STORAGE_ACCOUNT=$STORAGE_ACCOUNT AZURE_STORAGE_CONTAINER=$BLOB_CONTAINER
kubectl rollout restart deployment/myapp
```

`STORAGE` `ALL` `RESILIENCY`
---

## [High] Ephemeral OS 디스크 사용

**설명**: Ephemeral OS 디스크는 노드 재생성 속도가 빠르고 OS 디스크 지연 시간이 낮아 대규모 스케일 이벤트와 업그레이드에 유리합니다. 특히 상태를 외부화한 AKS 운영 모델과 잘 맞아서 노드 교체 부담을 줄일 수 있습니다. 다만 VM SKU와 로컬 디스크 용량 제약을 함께 확인해야 합니다.

**확인 방법**:
```bash
az graph query -q "where type=='microsoft.containerservice/managedclusters' | project id,resourceGroup,name,pools=properties.agentPoolProfiles | mvexpand pools | extend compliant = (pools.osDiskType=='Ephemeral') | project id,name=strcat(name,'-',pools.name), resourceGroup, compliant" -o table
az aks nodepool show -g $RG --cluster-name $CLUSTER -n $NODEPOOL --query "{osDiskType:osDiskType,osDiskSizeGB:osDiskSizeGb}" -o yaml
```

**권장 방식으로 변경**:
```bash
az aks nodepool add -g $RG --cluster-name $CLUSTER -n ephpool --node-vm-size Standard_D4ds_v5 --node-osdisk-type Ephemeral --node-osdisk-size 128
```

`STORAGE` `ALL` `RESILIENCY`
---

## [High] 비-ephemeral 디스크를 쓴다면 많은 Pod/노드 환경에서 더 큰 OS 디스크와 높은 IOPS 선택

**설명**: 노드당 Pod 수가 많아질수록 이미지 캐시, 컨테이너 로그, 임시 파일이 OS 디스크를 빠르게 소모합니다. Ephemeral을 쓸 수 없는 환경에서는 OS 디스크 크기와 VM 계열을 보수적으로 잡아야 디스크 포화로 인한 kubelet 불안정성을 줄일 수 있습니다. 특히 Windows 노드나 로깅이 많은 워크로드에서 체감 차이가 큽니다.

**확인 방법**:
```bash
az aks nodepool show -g $RG --cluster-name $CLUSTER -n $NODEPOOL --query "{osDiskType:osDiskType,osDiskSizeGB:osDiskSizeGb,vmSize:vmSize,maxPods:maxPods}" -o yaml
kubectl get nodes
```

**권장 방식으로 변경**:
```bash
az aks nodepool add -g $RG --cluster-name $CLUSTER -n logsnp --node-vm-size Standard_D8s_v5 --node-osdisk-size 256 --max-pods 110
```

`STORAGE` `ALL` `RESILIENCY`
---

## [High] 베이스 이미지와 노드 OS 버전 매핑

**설명**: Windows 컨테이너는 호스트 OS와 컨테이너 베이스 이미지 버전이 맞지 않으면 실행 자체가 실패할 수 있습니다. 운영 리뷰에서는 노드풀의 Windows SKU와 이미지 태그가 실제로 대응되는지 반드시 확인해야 합니다. 업그레이드 전후에 가장 자주 생기는 문제라서 배포 파이프라인에도 검증 로직을 넣는 편이 좋습니다.

**확인 방법**:
```bash
az aks nodepool show -g $RG --cluster-name $CLUSTER -n $WIN_POOL --query "{osType:osType,osSKU:osSKU,nodeImageVersion:nodeImageVersion}" -o yaml
kubectl get nodes -L kubernetes.azure.com/os-sku,kubernetes.io/os
```

**권장 방식으로 변경**:
```bash
kubectl patch deployment win-web --type merge -p '{"spec":{"template":{"spec":{"nodeSelector":{"kubernetes.io/os":"windows","kubernetes.azure.com/os-sku":"Windows2022"}}}}}'
kubectl set image deployment/win-web app=mcr.microsoft.com/windows/servercore:ltsc2022
```

`WINDOWS` `ALL`
---

## [High] CNI 네트워크 모드 구현

**설명**: Windows 노드풀은 AKS에서 Azure CNI 계열 네트워크 구성이 사실상 전제입니다. 네트워크 모델을 잘못 고르면 Windows 풀 추가 자체가 막히거나 이후 네트워크 정책·IP 계획이 꼬일 수 있습니다. 프로덕션에서는 처음부터 Windows 요구사항을 반영해 클러스터를 설계하는 편이 재작업 비용이 훨씬 적습니다.

**확인 방법**:
```bash
az aks show -g $RG -n $CLUSTER --query "{networkPlugin:networkProfile.networkPlugin,networkPolicy:networkProfile.networkPolicy}" -o yaml
kubectl get nodes -L kubernetes.io/os
```

**권장 방식으로 변경**:
```bash
az aks create -g $RG -n $CLUSTER --network-plugin azure --generate-ssh-keys
az aks nodepool add -g $RG --cluster-name $CLUSTER -n win22 --os-type Windows --os-sku Windows2022
```

`WINDOWS` `ALL`
---

## [High] Windows 컨테이너 패치 수준을 호스트 패치 수준과 동기화

**설명**: Windows 노드 이미지와 컨테이너 베이스 이미지가 서로 다른 패치 수준으로 오래 벌어지면 배포 실패나 런타임 오류가 생기기 쉽습니다. AKS가 새 노드 이미지를 제공해도 실제로 업그레이드하지 않으면 보호 효과가 없습니다. 운영 팀은 노드 이미지 갱신과 애플리케이션 이미지 재빌드를 같은 릴리스 흐름으로 묶는 것이 좋습니다.

**확인 방법**:
```bash
az aks show -g $RG -n $CLUSTER --query autoUpgradeProfile.nodeOSUpgradeChannel -o tsv
az aks nodepool show -g $RG --cluster-name $CLUSTER -n $WIN_POOL --query "{osSKU:osSKU,nodeImageVersion:nodeImageVersion}" -o yaml
kubectl get deployment -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}{"\t"}{range .spec.template.spec.containers[*]}{.image}{" "}{end}{"\n"}{end}'
```

**권장 방식으로 변경**:
```bash
az aks update -g $RG -n $CLUSTER --node-os-upgrade-channel NodeImage
az aks nodepool upgrade -g $RG --cluster-name $CLUSTER -n $WIN_POOL --node-image-only
kubectl set image deployment/win-web -n $NS app=mcr.microsoft.com/windows/servercore:ltsc2022
```

`WINDOWS` `ALL`
---

## [High] 컨테이너 트래픽 보호

**설명**: Windows 워크로드에서 네트워크 정책 지원이 제한될 수 있으므로 인증, TLS, 프록시 계층 같은 보완 통제가 더 중요합니다. 운영 리뷰에서는 서비스가 내부망만 믿고 열려 있지 않은지, 인그레스와 앱 자체가 인증·암호화를 제공하는지 함께 확인해야 합니다. 특히 east-west 트래픽도 최소 권한 원칙으로 설계하는 것이 좋습니다.

**확인 방법**:
```bash
kubectl get svc,ingress -A
kubectl get networkpolicy -A
```

**권장 방식으로 변경**:
```bash
kubectl create secret tls app-tls --cert=certs/tls.crt --key=certs/tls.key -n $NS
kubectl apply -f - <<'EOF2'
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: win-app
  namespace: default
spec:
  tls:
  - hosts:
    - win-app.contoso.com
    secretName: app-tls
  rules:
  - host: win-app.contoso.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: win-app
            port:
              number: 443
EOF2
```

`WINDOWS` `ALL`
---

## [High] Deployment Safeguards로 Kubernetes 모범사례를 정책적으로 강제

**설명**: Deployment Safeguards는 Azure Policy 기반으로 리소스 request/limit 누락, `:latest` 이미지 태그 사용, hostPath 마운트 등 알려진 안티패턴을 배포 시점에 경고(Warning)하거나 차단(Enforce)하는 AKS 내장 기능이다. 체크리스트 사이트(the-aks-checklist.com)에는 개별 안티패턴 항목만 있고, 이를 "정책으로 자동 강제"하는 이 기능 자체는 반영되어 있지 않다. AKS Automatic은 기본 Enforce, AKS Standard는 수동 설정이 필요하다.

**확인 방법 (az cli)**:
```bash
az aks show -g $RESOURCE_GROUP -n $CLUSTER \
  --query "{safeguardsLevel: safeguardsProfile.level, excludedNamespaces: safeguardsProfile.excludedNamespaces}" -o table
```

**권장 방식으로 변경**:
```bash
# Azure Policy 애드온이 선행 활성화되어 있어야 함
az aks enable-addons -g $RESOURCE_GROUP -n $CLUSTER --addons azure-policy

# Warning으로 먼저 적용해 영향 파악 후 Enforcement로 전환 권장
az aks update -g $RESOURCE_GROUP -n $CLUSTER --safeguards-level Warning
az aks update -g $RESOURCE_GROUP -n $CLUSTER --safeguards-level Enforcement --safeguards-excluded-ns kube-system,gatekeeper-system
```

`ADDITIONAL` `SECURITY` `OPERATIONS` `GOVERNANCE`
---

## [High] Image Cleaner(Eraser) 애드온으로 미사용/취약 이미지 자동 정리

**설명**: 노드에 누적된 미사용 컨테이너 이미지는 디스크 공간을 낭비할 뿐 아니라, 알려진 CVE가 있는 오래된 이미지가 노드에 계속 남아 공격 표면을 늘린다. Image Cleaner(오픈소스 Eraser 기반)는 주기적으로 미사용/비참조 이미지를 스캔해 자동 삭제하는 관리형 add-on이며, 체크리스트 사이트에는 "이미지 취약점 스캔" 항목은 있지만 이 자동 정리 add-on 자체는 없다.

**확인 방법 (az cli)**:
```bash
az aks show -g $RESOURCE_GROUP -n $CLUSTER \
  --query "securityProfile.imageCleaner" -o jsonc
```

**권장 방식으로 변경**:
```bash
az aks update -g $RESOURCE_GROUP -n $CLUSTER \
  --enable-image-cleaner --image-cleaner-interval-hours 24
```

`ADDITIONAL` `SECURITY` `CONTAINER` `OPERATIONS`
---

## [High] AKS Automatic 모드 채택 검토 (신규 구축/재구축 시)

**설명**: 체크리스트 사이트의 항목 상당수(시스템 node pool 분리, Cluster Autoscaler, Deployment Safeguards Enforce, 네트워크 정책, private cluster 성격의 하드닝 등)는 AKS Automatic 모드에서 기본값으로 미리 켜져 있다. 신규 클러스터를 구축하거나 대규모 재구축을 계획 중이라면, 개별 항목을 하나씩 수동으로 켜는 AKS Standard 대신 AKS Automatic을 기본값으로 검토할 가치가 있다. 단, 기존 운영 중인 클러스터는 모드 전환이 불가(재생성 필요)하므로 신규/재구축 시나리오에만 해당한다.

**확인 방법 (az cli)**:
```bash
az aks show -g $RESOURCE_GROUP -n $CLUSTER --query "sku" -o jsonc
# sku.name 이 "Automatic" 이면 이미 Automatic 모드
```

**권장 방식으로 변경**:
```bash
# 기존 클러스터의 in-place 전환은 지원하지 않음 — 신규 클러스터 생성 시에만 선택 가능
az aks create -g $RESOURCE_GROUP -n $CLUSTER --sku automatic
```

`ADDITIONAL` `OPERATIONS` `SECURITY` `GOVERNANCE`
---

## [Medium] ACR Task로 베이스 이미지 업데이트 시 컨테이너 이미지 자동 리빌드

**설명**: 애플리케이션 이미지를 한 번 빌드한 뒤 방치하면, 베이스 이미지(OS/런타임)에 보안 패치가 나와도 재빌드가 되지 않아 운영 중인 이미지에 알려진 취약점이 계속 남아있게 됩니다. ACR Task는 Git commit 트리거 외에도 **베이스 이미지 업데이트 트리거**와 **타이머(cron) 트리거**를 지원해서, 베이스 이미지가 갱신되거나 정해진 주기가 되면 ACR이 자동으로 `docker build`를 다시 실행하고 새 이미지를 레지스트리에 push하도록 구성할 수 있습니다. AKS 노드가 최신 패치를 받는 것과 별개로, 컨테이너 이미지 자체도 지속적으로 최신화해야 공급망 전체의 패치 지연을 줄일 수 있습니다.

**확인 방법 (az cli)**:
```bash
# 레지스트리에 구성된 Task 목록 확인 (트리거 유형은 Portal Trigger 열 아이콘 또는 show로 확인)
az acr task list --registry <ACR_NAME> --output table

# 특정 Task의 트리거 구성(베이스 이미지/커밋/타이머) 상세 확인
az acr task show --registry <ACR_NAME> --name <TASK_NAME> --query "{step:step, trigger:trigger}" --output json

# 최근 실행 이력(자동 리빌드가 실제로 동작했는지) 확인
az acr task list-runs --registry <ACR_NAME> --output table
```

**권장 방식으로 변경**:
```bash
# Dockerfile 기반 Task 생성 시 베이스 이미지 업데이트 트리거를 기본 활성화(Default: true)
az acr task create --registry <ACR_NAME> --name <TASK_NAME> \
  --image <REPOSITORY>:{{.Run.ID}} \
  --context <GIT_REPO_URL_OR_OCI_CONTEXT> --file Dockerfile \
  --base-image-trigger-type Runtime

# 기존 Task에 베이스 이미지 트리거만 별도로 켜고 싶다면
az acr task update --registry <ACR_NAME> --name <TASK_NAME> --base-image-trigger-enabled true

# 주기적 리빌드가 추가로 필요하면 cron 기반 타이머 트리거도 함께 구성 가능
az acr task create --registry <ACR_NAME> --name <TASK_NAME> \
  --image <REPOSITORY>:{{.Run.ID}} \
  --context <GIT_REPO_URL_OR_OCI_CONTEXT> --file Dockerfile \
  --schedule "0 0 * * *"
```

`CONTAINER` `ALL` `OPERATIONS` `SECURITY`
---

## [Medium] Bastion 호스트를 통해 노드에 안전하게 연결하세요

**설명**: 운영 노드에 대한 직접 공개 접근은 공격 표면을 크게 넓히고, 비상 시에도 추적하기 어려운 우회 운영을 낳기 쉽습니다. 프라이빗 클러스터와 점프 호스트/Bastion 경유 접근을 기본값으로 두면 운영 접근 경로를 통제하고 감사하기 쉬워집니다.

**확인 방법 (az cli)**:
```bash
az aks show -g $RESOURCE_GROUP -n $CLUSTER --query "{privateCluster:apiServerAccessProfile.enablePrivateCluster,nodeResourceGroup:nodeResourceGroup}" -o yaml
az aks nodepool list -g $RESOURCE_GROUP --cluster-name $CLUSTER --query "[].{pool:name,nodePublicIP:enableNodePublicIP}" -o table
```

**권장 방식으로 변경**:
```bash
az aks update -g $RESOURCE_GROUP -n $CLUSTER --enable-private-cluster --disable-public-fqdn
az aks nodepool update -g $RESOURCE_GROUP --cluster-name $CLUSTER -n $NODEPOOL --ssh-access disabled
# 별도 관리 VNet에 Azure Bastion 또는 점프박스를 배치해 운영 접근을 제한합니다.
```

`OPERATIONS` `ALL` `SECURITY`
---

## [Medium] 클러스터 이슈를 정기적으로 점검하세요

**설명**: 성능 저하, 스케줄링 실패, 리소스 요청/제한 누락 같은 문제는 장애가 나기 전까지 겉으로 드러나지 않는 경우가 많습니다. 운영팀은 이벤트와 비정상 파드를 지속적으로 점검하고, 정적/동적 스캐너를 정기 배치로 돌려 누적 리스크를 조기에 제거해야 합니다.

**확인 방법 (az cli)**:
```bash
az aks command invoke -g $RESOURCE_GROUP -n $CLUSTER -c "kubectl get pods -A; echo '---'; kubectl get events -A --sort-by=.lastTimestamp | tail -n 100"
kubectl get resourcequota -A
```

**권장 방식으로 변경**:
```bash
# kubestriker, kubescape, kubeaudit 같은 스캐너를 CI 또는 CronJob으로 주기 실행해야 합니다.
# az CLI만으로 '정기 점검 체계'를 강제할 수는 없으므로 파이프라인/운영 프로세스 구성이 필요합니다.
```

`OPERATIONS` `ALL`
---

## [Medium] 업그레이드 채널을 설정하세요

**설명**: 클러스터 버전 업그레이드를 완전히 수동으로만 운영하면 누가 언제 어떤 기준으로 업그레이드할지 팀 의존성이 커집니다. 자동 업그레이드 채널을 정해 두면 보안 패치와 지원 버전 유지가 예측 가능해지고, 지원 종료 리스크를 줄일 수 있습니다.

**확인 방법 (az cli)**:
```bash
az aks show -g $RESOURCE_GROUP -n $CLUSTER --query "autoUpgradeProfile.upgradeChannel" -o tsv
```

**권장 방식으로 변경**:
```bash
az aks update -g $RESOURCE_GROUP -n $CLUSTER --auto-upgrade-channel stable
```

`OPERATIONS` `ALL`
---

## [Medium] 클러스터 오토스케일링을 활성화하세요

**설명**: 워크로드 수요는 시간대와 이벤트에 따라 급격히 달라지므로, 노드 수를 고정하면 과소 프로비저닝과 과다 지출을 번갈아 겪게 됩니다. 오토스케일러를 활성화하면 스케줄링 실패를 줄이면서도 유휴 노드를 자동으로 줄여 비용과 안정성을 함께 관리할 수 있습니다.

**확인 방법 (az cli)**:
```bash
az aks nodepool list -g $RESOURCE_GROUP --cluster-name $CLUSTER --query "[].{pool:name,autoscaling:enableAutoScaling,min:minCount,max:maxCount}" -o table
```

**권장 방식으로 변경**:
```bash
az aks nodepool update --enable-cluster-autoscaler --min-count 1 --max-count 5 -g $RESOURCE_GROUP -n $NODEPOOL --cluster-name $CLUSTER
```

`OPERATIONS` `ALL` `RESOURCES` `FINOPS`
---

## [Medium] K8S 운영 도구로 효율을 높이세요

**설명**: 운영 난이도가 높은 클러스터일수록 kubectl만으로는 이벤트 추적, 로그 탐색, 네임스페이스 전환, 실시간 관찰이 비효율적입니다. 팀 표준 도구(예: helm, krew, kubectx, kubens, stern, k9s)를 정해두면 대응 속도와 작업 일관성이 크게 좋아집니다.

**확인 방법 (az cli)**:
```bash
# kubectl, helm, krew 플러그인(kubectx/kubens/stern 등) 사용 표준이 있는지 운영 문서와 관리자 워크스테이션 구성을 확인합니다.
# az CLI만으로 도구 표준화 수준을 판별할 수 없습니다.
```

**권장 방식으로 변경**:
```bash
# 운영 표준에 따라 kubectl, helm, krew, kubectx, kubens, stern, k9s 같은 도구를 팀 표준으로 채택합니다.
# 로컬/운영자 환경 구성 변경이 필요하므로 단일 az CLI 명령으로 강제할 수 없습니다.
```

`OPERATIONS` `ALL`
---

## [Medium] default 네임스페이스를 사용하지 마세요

**설명**: default 네임스페이스에 워크로드를 계속 쌓으면 팀·서비스·환경 간 경계가 사라져 권한, 네트워크 정책, 쿼터를 분리하기 어려워집니다. 운영 클러스터는 애플리케이션별 또는 환경별 네임스페이스를 분리해 책임 범위와 정책 적용 지점을 명확히 해야 합니다.

**확인 방법 (az cli)**:
```bash
kubectl get all -n default
```

**권장 방식으로 변경**:
```bash
kubectl create namespace app-prod
kubectl config set-context --current --namespace=app-prod
```

`OPERATIONS` `ALL`
---

## [Medium] 노드의 CPU 및 메모리 사용률을 모니터링하세요

**설명**: 노드 사용률을 보지 않으면 포화 직전 상태나 과대 할당 상태를 장기간 놓치게 되어 장애와 비용 낭비가 동시에 발생합니다. 운영자는 노드별 CPU·메모리 추세를 지속적으로 보면서 스케일, 노드 크기, 워크로드 배치를 조정해야 합니다.

**확인 방법 (az cli)**:
```bash
az aks show -g $RESOURCE_GROUP -n $CLUSTER --query "addonProfiles.omsagent.enabled" -o tsv
kubectl top nodes
```

**권장 방식으로 변경**:
```bash
az aks update -g $RESOURCE_GROUP -n $CLUSTER --enable-azure-monitor-logs --workspace-resource-id $WORKSPACE_ID
```

`OPERATIONS` `ALL` `MONITORING`
---

## [Medium] Azure Advisor 권장 사항을 정기적으로 확인하세요

**설명**: Azure Advisor는 비용, 성능, 고가용성, 보안 측면에서 놓치기 쉬운 관리 항목을 정리해 보여줍니다. 운영팀이 이를 주기적으로 검토하면 환경이 커질수록 누락되기 쉬운 개선 포인트를 빠르게 찾을 수 있습니다.

**확인 방법 (az cli)**:
```bash
AKS_ID=$(az aks show -g $RESOURCE_GROUP -n $CLUSTER --query id -o tsv)
az advisor recommendation list --ids "$AKS_ID" --refresh -o table
```

**권장 방식으로 변경**:
```bash
# Azure Advisor 권장 사항은 항목별 조치가 다릅니다.
# 예: 비용/성능 권장 사항에 따라 az aks nodepool update, az aks update, 진단 설정 변경 등을 적용합니다.
```

`OPERATIONS` `ALL` `COMPLIANCE` `COST` `FINOPS` `MONITORING`
---

## [Medium] 로깅에 ContainerLogV2 스키마를 활성화하세요

**설명**: ContainerLogV2는 네임스페이스, 파드, 컨테이너 같은 필드를 더 구조적으로 제공해 운영 쿼리와 대시보드 품질을 높여 줍니다. 기존 스키마에 머무르면 필드 활용성과 장기 호환성 측면에서 손해가 커지므로, 운영 로그 분석 체계는 가급적 v2 기준으로 맞추는 것이 좋습니다.

**확인 방법 (az cli)**:
```bash
az monitor log-analytics workspace table list -g $WORKSPACE_RG --workspace-name $WORKSPACE --query "[?name=='ContainerLogV2'].name" -o table
kubectl -n kube-system get configmap container-azm-ms-agentconfig -o yaml
```

**권장 방식으로 변경**:
```bash
# container-azm-ms-agentconfig ConfigMap에서 containerlog_schema_version 을 v2로 설정한 뒤 적용합니다.
kubectl apply -f container-azm-ms-agentconfig.yaml
```

`OPERATIONS` `ALL` `MONITORING`
---

## [Medium] Azure Firewall/NVA 기반 egress 필터링을 사용하지 않는다면 Standard LB의 할당된 SNAT 포트를 모니터링하세요

**설명**: SNAT 포트가 고갈되면 외부 의존 서비스로의 연결 실패, 타임아웃, 간헐적 장애가 발생할 수 있습니다. 특히 별도 egress 제어 계층이 없다면 Standard Load Balancer의 SNAT 사용량을 지속적으로 관찰해 포트 고갈을 조기에 감지해야 합니다.

**확인 방법 (az cli)**:
```bash
NODE_RG=$(az aks show -g $RESOURCE_GROUP -n $CLUSTER --query nodeResourceGroup -o tsv)
LB_ID=$(az network lb list -g "$NODE_RG" --query "[0].id" -o tsv)
az network lb outbound-rule list -g "$NODE_RG" --lb-name $(az network lb list -g "$NODE_RG" --query "[0].name" -o tsv) -o table
az monitor metrics list-definitions --resource "$LB_ID" --query "[?contains(name.value, 'Snat')].name.value" -o table
az monitor metrics list --resource "$LB_ID" --metrics UsedSnatPorts --aggregation Average --interval PT5M -o table
```

**권장 방식으로 변경**:
```bash
NODE_RG=$(az aks show -g $RESOURCE_GROUP -n $CLUSTER --query nodeResourceGroup -o tsv)
LB_ID=$(az network lb list -g "$NODE_RG" --query "[0].id" -o tsv)
az monitor metrics alert create -g $RESOURCE_GROUP -n aks-snat-port-alert --scopes "$LB_ID" --condition "avg UsedSnatPorts > THRESHOLD" --window-size 5m --evaluation-frequency 1m --severity 2
```

`OPERATIONS` `ALL` `MONITORING` `SECURITY`
---

## [Medium] Kubernetes 버전에 Long-Term Support(LTS) 사용을 검토하세요

**설명**: LTS를 선택하면 지원 기간이 길어져 운영팀이 버전 업그레이드를 너무 자주 반복하지 않아도 됩니다. 업그레이드 빈도를 줄이면서도 지원 상태를 유지할 수 있어, 변경 윈도우가 제한적인 프로덕션 환경에 특히 유리합니다.

**확인 방법 (az cli)**:
```bash
az aks show -g $RESOURCE_GROUP -n $CLUSTER --query "supportPlan" -o tsv
az aks get-upgrades -g $RESOURCE_GROUP -n $CLUSTER -o json
```

**권장 방식으로 변경**:
```bash
az aks update -g $RESOURCE_GROUP -n $CLUSTER --k8s-support-plan AKSLongTermSupport
```

`OPERATIONS` `ALL` `SECURITY`
---

## [Medium] 클러스터를 논리적으로 격리

**설명**: 하나의 AKS 클러스터를 여러 팀이 함께 쓰더라도 네임스페이스, RBAC, 리소스 쿼터, 네트워크 정책이 분리되지 않으면 장애와 보안 영향이 빠르게 전파됩니다. 생산 환경에서는 먼저 논리 격리로 멀티테넌시 경계를 만들고, 팀 간 자원 경합과 실수 범위를 네임스페이스 단위로 제한하는 것이 운영 효율에 유리합니다.

**확인 방법 (az cli)**:
```bash
az aks get-credentials -g $RESOURCE_GROUP -n $CLUSTER
kubectl get ns
kubectl get resourcequota -A
kubectl get networkpolicy -A
```

**권장 방식으로 변경**:
```bash
kubectl create namespace $NAMESPACE
kubectl apply -n $NAMESPACE -f - <<'EOF'
apiVersion: v1
kind: ResourceQuota
metadata:
  name: namespace-quota
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF
```

`CLUSTER MULTI` `ALL` `MULTI-TENANCY`
---

## [Medium] AKS에 Azure 태그 사용

**설명**: 태그는 비용 배부, 소유자 식별, 자동화 분기, 운영 책임 소재를 정리하는 가장 기본적인 메타데이터입니다. 운영 중인 프로덕션 클러스터라면 최소한 환경, 서비스명, 소유팀, 비용센터 태그를 일관되게 부여해 인벤토리와 FinOps 분석에 바로 활용할 수 있어야 합니다.

**확인 방법 (az cli)**:
```bash
az aks show -g $RESOURCE_GROUP -n $CLUSTER --query "{id:id,tags:tags,nodeResourceGroup:nodeResourceGroup}" -o json
```

**권장 방식으로 변경**:
```bash
AKS_ID=$(az aks show -g $RESOURCE_GROUP -n $CLUSTER --query id -o tsv)
az tag update --resource-id "$AKS_ID" --operation Merge --tags environment=prod owner=platform costCenter=cc123
```

`CLUSTER MULTI` `ALL` `OPERATIONS` `FINOPS`
---

## [Medium] 요구사항에 맞는 CNI 네트워크 플러그인 선택 (권장: Azure CNI)

**설명**: 네트워크 플러그인은 Pod IP 할당 방식, 온프레미스 연동, 네트워크 정책 엔진, 운영 난이도를 좌우하는 핵심 설계 결정입니다. 기업 환경에서는 Azure CNI를 기본으로 검토하되, IP 고갈을 줄이고 라우팅을 단순화하려면 Overlay 모델까지 함께 고려하는 것이 실무적으로 유리합니다.

**확인 방법 (az cli)**:
```bash
az graph query -q "where type=='microsoft.containerservice/managedclusters' and name=='$CLUSTER' | extend compliant = (properties.networkProfile.networkPlugin=='azure') | distinct id, compliant"
```

**권장 방식으로 변경**:
```bash
# 네트워크 플러그인 변경은 사실상 재생성이므로 신규/대체 클러스터에 적용합니다.
az aks create -g $RESOURCE_GROUP -n $CLUSTER --network-plugin azure --network-plugin-mode overlay --network-dataplane cilium --network-policy cilium
```

`NETWORKING` `ALL` `NETWORK`
---

## [Medium] 웹 애플리케이션은 LoadBalancer 서비스 대신 Ingress Controller로 노출

**설명**: HTTP/HTTPS 워크로드를 서비스별 LoadBalancer로 직접 노출하면 공인 IP, 인증서, 라우팅 규칙이 분산되어 운영 복잡도가 커집니다. Ingress Controller를 사용하면 경로 기반 라우팅, TLS 종료, WAF 연계, 공통 정책 적용을 중앙화할 수 있어 운영 표준화에 유리합니다.

**확인 방법 (az cli)**:
```bash
az aks get-credentials -g $RESOURCE_GROUP -n $CLUSTER
kubectl get ingressclass
kubectl get svc -A -o jsonpath='{range .items[?(@.spec.type=="LoadBalancer")]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.status.loadBalancer.ingress[*].ip}{"\n"}{end}'
```

**권장 방식으로 변경**:
```bash
az aks approuting enable -g $RESOURCE_GROUP -n $CLUSTER --nginx AnnotationControlled
```

`NETWORKING` `ALL` `NETWORK`
---

## [Medium] 외부 노출 애플리케이션은 WAF로 보호

**설명**: 인터넷에 공개된 애플리케이션은 OWASP Top 10, 봇, L7 공격, 비정상 요청에 지속적으로 노출됩니다. 프로덕션에서는 Ingress 앞단에 Application Gateway WAF 또는 동급 서비스를 배치해 탐지뿐 아니라 차단 모드까지 검토해야 합니다.

**확인 방법 (az cli)**:
```bash
az network application-gateway show -g $APPGW_RESOURCE_GROUP -n $APPGW --query "firewallPolicy.id" -o tsv
az network application-gateway waf-policy show -g $APPGW_RESOURCE_GROUP -n $WAF_POLICY --query "{state:policySettings.state,mode:policySettings.mode}" -o json
```

**권장 방식으로 변경**:
```bash
az network application-gateway waf-policy create -g $APPGW_RESOURCE_GROUP -n $WAF_POLICY
az network application-gateway waf-policy policy-setting update -g $APPGW_RESOURCE_GROUP --policy-name $WAF_POLICY --state Enabled --mode Prevention
```

`NETWORKING` `ALL` `NETWORK`
---

## [Medium] 네트워크 정책으로 트래픽 흐름 제어

**설명**: 기본 설정에서는 클러스터 내부 Pod 간 통신이 지나치게 넓게 열려 있어, 침해나 설정 실수가 옆 워크로드로 쉽게 확산될 수 있습니다. 프로덕션에서는 애플리케이션 간 허용 통신만 남기는 방식으로 정책을 정의해 동서 트래픽을 최소 권한으로 줄여야 합니다.

**확인 방법 (az cli)**:
```bash
az graph query -q "where type=='microsoft.containerservice/managedclusters' and name=='$CLUSTER' | extend compliant = isnotnull(properties.networkProfile.networkPolicy) | distinct id, compliant"
az aks show -g $RESOURCE_GROUP -n $CLUSTER --query "{plugin:networkProfile.networkPlugin,policy:networkProfile.networkPolicy}" -o json
```

**권장 방식으로 변경**:
```bash
az aks update -g $RESOURCE_GROUP -n $CLUSTER --network-policy azure
```

`NETWORKING` `ALL` `SECURITY` `NETWORK` `MULTI-TENANCY`
---

## [Medium] 각 네임스페이스에 기본 네트워크 정책 구성

**설명**: 네트워크 정책 기능만 켜 두고 기본 차단 정책이 없으면 새 네임스페이스나 새 서비스는 여전히 과도하게 열려 있을 수 있습니다. 각 네임스페이스에서 먼저 기본 차단 정책을 배포하고, 이후 필요한 인바운드/아웃바운드만 점진적으로 허용하는 운영 패턴이 안전합니다.

**확인 방법 (az cli)**:
```bash
az aks get-credentials -g $RESOURCE_GROUP -n $CLUSTER
kubectl get networkpolicy -n $NAMESPACE
```

**권장 방식으로 변경**:
```bash
kubectl apply -n $NAMESPACE -f - <<'EOF'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF
```

`NETWORKING` `ALL` `SECURITY` `MULTI-TENANCY` `NETWORK`
---

## [Medium] 보안 요구사항이 있으면 AzFW/NVA로 아웃바운드 트래픽 필터링

**설명**: 민감한 워크로드는 인터넷으로 나가는 트래픽도 제어·기록·승인되어야 하며, 기본 Load Balancer egress만으로는 이런 요구를 충족하기 어렵습니다. UDR과 Azure Firewall/NVA를 조합하면 목적지 제한, 감사 로깅, 위협 차단을 중앙에서 강제할 수 있습니다.

**확인 방법 (az cli)**:
```bash
az graph query -q "where type=='microsoft.containerservice/managedclusters' and name=='$CLUSTER' | extend compliant = (properties.networkProfile.outboundType=='userDefinedRouting') | distinct id, compliant"
az network vnet subnet show --ids $AKS_SUBNET_ID --query "routeTable.id" -o tsv
```

**권장 방식으로 변경**:
```bash
az network route-table route create -g $NETWORK_RESOURCE_GROUP --route-table-name $ROUTE_TABLE -n default-egress --address-prefix 0.0.0.0/0 --next-hop-type VirtualAppliance --next-hop-ip-address $FIREWALL_PRIVATE_IP
az network vnet subnet update --ids $AKS_SUBNET_ID --route-table $ROUTE_TABLE_ID
az aks update -g $RESOURCE_GROUP -n $CLUSTER --outbound-type userDefinedRouting
```

`NETWORKING` `ALL` `SECURITY` `NETWORK`
---

## [Medium] Pod의 VMSS IMDS 접근 차단

**설명**: Pod가 노드의 IMDS에 접근하면 노드에 연결된 관리형 ID 토큰을 악용할 여지가 생깁니다. 프로덕션에서는 워크로드 아이덴티티를 우선 사용하고, 노드 메타데이터 접근은 명시적으로 막아 자격 증명 탈취 경로를 줄여야 합니다.

**확인 방법 (az cli)**:
```bash
az aks nodepool show -g $RESOURCE_GROUP --cluster-name $CLUSTER -n $NODEPOOL --query enableImdsRestriction -o tsv
```

**권장 방식으로 변경**:
```bash
az aks nodepool update -g $RESOURCE_GROUP --cluster-name $CLUSTER -n $NODEPOOL --enable-imds-restriction
```

`NETWORKING` `ALL` `SECURITY` `NETWORK`
---

## [Medium] 트래픽 관리 기능 활성화

**설명**: 단일 클러스터나 단일 진입점에만 의존하면 지역 장애, 배포 실패, 순간 과부하에 취약합니다. Traffic Manager 같은 글로벌 트래픽 관리 계층을 두면 다중 클러스터 간 우회와 성능 기반 분산을 구현해 가용성을 높일 수 있습니다.

**확인 방법 (az cli)**:
```bash
az network traffic-manager profile list -g $RESOURCE_GROUP -o table
az network traffic-manager endpoint list -g $RESOURCE_GROUP --profile-name $TM_PROFILE -o table
```

**권장 방식으로 변경**:
```bash
az network traffic-manager profile create -g $RESOURCE_GROUP -n $TM_PROFILE --routing-method Performance --unique-dns-name $TM_DNS_LABEL --ttl 30 --protocol HTTPS --port 443 --path /
az network traffic-manager endpoint create -g $RESOURCE_GROUP --profile-name $TM_PROFILE -n $TM_ENDPOINT --type azureEndpoints --target-resource-id $PUBLIC_IP_ID --endpoint-status Enabled
```

`NETWORKING` `ALL` `NETWORK` `RESILIENCY` `MULTI-TENANCY`
---

## [Medium] 공개 API 엔드포인트를 쓴다면 접근 가능한 IP 대역 제한

**설명**: 공개 API 서버가 어디서나 접근 가능하면 제어 평면이 불필요한 스캔과 공격에 계속 노출됩니다. 운영자가 접속하는 고정 NAT, VPN, Bastion 대역만 허용하도록 줄이면 관리 평면 노출 범위를 크게 줄일 수 있습니다.

**확인 방법 (az cli)**:
```bash
az aks show -g $RESOURCE_GROUP -n $CLUSTER --query "{privateCluster:apiServerAccessProfile.enablePrivateCluster,authorizedIpRanges:apiServerAccessProfile.authorizedIpRanges}" -o json
```

**권장 방식으로 변경**:
```bash
az aks update -g $RESOURCE_GROUP -n $CLUSTER --api-server-authorized-ip-ranges "203.0.113.10/32,198.51.100.0/24"
```

`NETWORKING` `ALL` `SECURITY` `NETWORK`
---

## [Medium] 대규모 아웃바운드 트래픽에는 NAT Gateway를 outboundType으로 사용

**설명**: Load Balancer 기반 아웃바운드는 동시 연결 수와 포트 고갈 관점에서 대규모 워크로드에 한계가 있을 수 있습니다. NAT Gateway를 사용하면 더 많은 SNAT 포트와 예측 가능한 아웃바운드 동작을 확보해 대량 egress 환경에서 안정성이 좋아집니다.

**확인 방법 (az cli)**:
```bash
az aks show -g $RESOURCE_GROUP -n $CLUSTER --query "{outboundType:networkProfile.outboundType,natGatewayProfile:networkProfile.natGatewayProfile}" -o json
```

**권장 방식으로 변경**:
```bash
az aks update -g $RESOURCE_GROUP -n $CLUSTER --outbound-type managedNATGateway --nat-gateway-managed-outbound-ip-count 2
```

`NETWORKING` `ALL` `SCALABILITY` `NETWORK`
---

## [Medium] Azure CNI IP 고갈 방지를 위해 동적 IP 할당 사용

**설명**: 전통적인 Azure CNI는 Pod IP를 미리 넉넉히 예약해야 해서 클러스터가 커질수록 서브넷 소진 위험이 빠르게 커집니다. Pod Subnet과 동적 할당을 사용하면 필요한 만큼만 IP를 가져와 IP 사용 효율과 확장성을 함께 개선할 수 있습니다.

**확인 방법 (az cli)**:
```bash
az aks show -g $RESOURCE_GROUP -n $CLUSTER --query "agentPoolProfiles[].{pool:name,podSubnetId:podSubnetId,podIpAllocationMode:podIpAllocationMode}" -o table
```

**권장 방식으로 변경**:
```bash
az aks nodepool add -g $RESOURCE_GROUP --cluster-name $CLUSTER -n npdyn --vnet-subnet-id $NODE_SUBNET_ID --pod-subnet-id $POD_SUBNET_ID --pod-ip-allocation-mode DynamicIndividual
```

`NETWORKING` `ALL` `SCALABILITY` `NETWORK`
---

## [Medium] 클러스터에서 PaaS 접근 시 Private Endpoint 우선, 차선책으로 Service Endpoint 사용

**설명**: 스토리지, Key Vault, SQL 같은 PaaS를 공용 엔드포인트로 접근하면 네트워크 경계와 데이터 경로 통제가 약해집니다. 가능하면 Private Endpoint로 사설 경로를 만들고, 불가한 경우에만 Service Endpoint를 사용해 클러스터 서브넷에서만 접근되도록 제한하는 것이 안전합니다.

**확인 방법 (az cli)**:
```bash
az graph query -q "Resources | where type =~ 'microsoft.network/privateendpoints' | project name, resourceGroup, subnet=tostring(properties.subnet.id), target=tostring(properties.privateLinkServiceConnections[0].properties.privateLinkServiceId)" --first 100
az network vnet subnet show --ids $AKS_SUBNET_ID --query "serviceEndpoints[].service" -o tsv
```

**권장 방식으로 변경**:
```bash
az network private-endpoint create -g $RESOURCE_GROUP -n $PE_NAME --vnet-name $VNET_NAME --subnet $SUBNET_NAME --private-connection-resource-id $PAAS_RESOURCE_ID --group-id $GROUP_ID --connection-name $PE_CONNECTION
# 차선책: az network vnet subnet update --ids $AKS_SUBNET_ID --service-endpoints Microsoft.Storage
```

`NETWORKING` `ALL` `SECURITY` `NETWORK`
---

## [Medium] 고급 마이크로서비스 통신 관리가 필요하면 Service Mesh 고려

**설명**: 서비스 수가 늘어나면 재시도, 타임아웃, mTLS, 카나리, 세분화된 트래픽 제어를 애플리케이션 코드만으로 일관되게 관리하기 어렵습니다. 서비스 메시를 사용하면 보안과 네트워크 정책을 데이터 플레인에서 표준화해 운영 품질을 높일 수 있습니다.

**확인 방법 (az cli)**:
```bash
az aks show -g $RESOURCE_GROUP -n $CLUSTER --query "serviceMeshProfile.mode" -o tsv
```

**권장 방식으로 변경**:
```bash
az aks mesh enable -g $RESOURCE_GROUP -n $CLUSTER
```

`NETWORKING` `ALL` `SECURITY` `NETWORK`
---

## [Medium] Azure CNI 사용 시 NodePool별로 서로 다른 서브넷 고려

**설명**: 노드풀마다 역할과 보안 요구가 다르면 서브넷도 분리해 NSG, UDR, 주소 계획을 독립적으로 관리하는 편이 좋습니다. 시스템 풀과 사용자 풀, 민감 워크로드 풀을 분리하면 장애 반경과 보안 예외 범위를 작게 유지할 수 있습니다.

**확인 방법 (az cli)**:
```bash
az aks show -g $RESOURCE_GROUP -n $CLUSTER --query "agentPoolProfiles[].{pool:name,subnet:vnetSubnetId}" -o table
```

**권장 방식으로 변경**:
```bash
az aks nodepool add -g $RESOURCE_GROUP --cluster-name $CLUSTER -n npsplit --vnet-subnet-id $NEW_NODE_SUBNET_ID
```

`NETWORKING` `ALL` `SECURITY` `NETWORK`
---

## [Medium] Service CIDR와 Pod/Subnet CIDR의 비중첩 여부 검증

**설명**: Service CIDR이 Pod CIDR이나 AKS 서브넷, 피어링된 VNet 주소와 겹치면 DNS 해석, 서비스 접근, 라우팅에서 원인 파악이 어려운 장애가 발생할 수 있습니다. 초기 설계 때 겹치지 않게 정하는 것이 최선이며, 운영 중 발견되면 재배포 수준의 수정이 필요할 수 있습니다.

**확인 방법 (az cli)**:
```bash
AKS_JSON=$(az aks show -g $RESOURCE_GROUP -n $CLUSTER --query "{serviceCidr:networkProfile.serviceCidr,serviceCidrs:networkProfile.serviceCidrs,podCidr:networkProfile.podCidr,podCidrs:networkProfile.podCidrs}" -o json)
SUBNET_JSON=$(az network vnet subnet show -g $VNET_RESOURCE_GROUP -n $SUBNET_NAME --vnet-name $VNET_NAME --query "{subnet:addressPrefix,subnets:addressPrefixes}" -o json)
export AKS_JSON SUBNET_JSON
python3 - <<'PY'
import ipaddress, json, os
aks = json.loads(os.environ["AKS_JSON"])
sub = json.loads(os.environ["SUBNET_JSON"])
values = []
for key in ("serviceCidr", "podCidr"):
    if aks.get(key):
        values.append((key, aks[key]))
for key in ("serviceCidrs", "podCidrs"):
    if aks.get(key):
        values.extend((key, v) for v in aks[key])
if sub.get("subnet"):
    values.append(("subnet", sub["subnet"]))
if sub.get("subnets"):
    values.extend(("subnet", v) for v in sub["subnets"])
for i, (an, av) in enumerate(values):
    for bn, bv in values[i+1:]:
        print(f"{an}:{av} <-> {bn}:{bv} overlap={ipaddress.ip_network(av, strict=False).overlaps(ipaddress.ip_network(bv, strict=False))}")
PY
```

**권장 방식으로 변경**:
```bash
# 겹치는 CIDR은 새 클러스터/새 주소 계획으로 바로잡는 것이 안전합니다.
az aks create -g $RESOURCE_GROUP -n $CLUSTER --network-plugin azure --network-plugin-mode overlay --pod-cidr 10.240.0.0/16 --service-cidr 10.0.0.0/16 --dns-service-ip 10.0.0.10
```

`NETWORKING` `ALL` `SECURITY` `NETWORK`
---

## [Medium] 적절한 Startup Probe 구현

**설명**: 기동 시간이 긴 애플리케이션은 startup probe가 없으면 liveness probe가 초기 부팅을 장애로 오판해 반복 재시작을 일으킬 수 있습니다. 특히 JVM, 대형 모델 로딩, 마이그레이션 수행 앱은 startup probe로 초기 구간을 분리해야 안정적으로 올라옵니다.

**확인 방법**:
```bash
kubectl get deploy -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\n"}{range .spec.template.spec.containers[*]}{"  - "}{.name}{": startup="}{.startupProbe}{"\n"}{end}{end}'
```

**권장 방식으로 변경**:
```yaml
startupProbe:
  httpGet:
    path: /startup
    port: 8080
  periodSeconds: 10
  timeoutSeconds: 2
  failureThreshold: 30
```

`APPLICATION` `ALL`
---

## [Medium] 적절한 Readiness Probe 구현

**설명**: 프로세스가 떠 있다고 해서 곧바로 트래픽을 받을 준비가 된 것은 아닙니다. readiness probe가 있어야 초기 데이터 로딩, 캐시 워밍, 외부 서비스 연결 복구 중인 파드가 Service 엔드포인트에 잘못 포함되는 일을 막을 수 있습니다.

**확인 방법**:
```bash
kubectl get deploy -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\n"}{range .spec.template.spec.containers[*]}{"  - "}{.name}{": readiness="}{.readinessProbe}{"\n"}{end}{end}'
```

**권장 방식으로 변경**:
```yaml
readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 3
```

`APPLICATION` `ALL`
---

## [Medium] Deployment에 2개 이상의 Replica 운영

**설명**: replica가 1개뿐이면 파드 크래시, 노드 재부팅, 드레인, 업그레이드 시 서비스 중단이 바로 발생합니다. 운영 워크로드는 최소 2개 이상으로 두고, 가능하면 서로 다른 노드나 가용 영역에 분산되도록 함께 설계해야 합니다.

**확인 방법**:
```bash
kubectl get deploy -A -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,REPLICAS:.spec.replicas,READY:.status.readyReplicas'
```

**권장 방식으로 변경**:
```yaml
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
```

`APPLICATION` `ALL`
---

## [Medium] 모든 리소스에 태그와 라벨 적용

**설명**: 일관된 라벨은 운영자가 리소스를 검색하고, 모니터링을 분류하고, 정책과 네트워크 제어를 적용하는 기본 키가 됩니다. 라벨 체계가 없으면 Service selector 불일치, 운영 자동화 누락, 비용/소유권 추적 실패가 자주 발생합니다.

**확인 방법**:
```bash
kubectl get deploy,svc,ingress,configmap,secret -A --show-labels
```

**권장 방식으로 변경**:
```yaml
metadata:
  labels:
    app.kubernetes.io/name: payments-api
    app.kubernetes.io/part-of: commerce-platform
    app.kubernetes.io/component: backend
    app.kubernetes.io/environment: production
    owner: team-payments
```

`APPLICATION` `ALL`
---

## [Medium] Azure Workload Identity 적용

**설명**: Pod 내부에 고정 자격 증명이나 장기 비밀값을 넣는 방식은 유출과 재사용 위험이 큽니다. Azure Workload Identity를 사용하면 서비스 계정과 관리 ID를 연계해 필요한 Azure 리소스에만 최소 권한으로 접근하게 만들 수 있고, 비밀 회전 부담도 줄어듭니다.

**확인 방법**:
```bash
az aks show --resource-group <RG> --name <CLUSTER> --output json
kubectl get sa -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.metadata.annotations.azure\.workload\.identity/client-id}{"\n"}{end}'
```

**권장 방식으로 변경**:
```bash
az aks update --resource-group <RG> --name <CLUSTER> --enable-oidc-issuer --enable-workload-identity
kubectl annotate serviceaccount api-sa -n prod azure.workload.identity/client-id=<USER_ASSIGNED_MANAGED_IDENTITY_CLIENT_ID>
```

`APPLICATION` `ALL` `IDENTITY` `SECURITY`
---

## [Medium] Kubernetes Namespace로 리소스 격리

**설명**: 모든 워크로드를 default namespace에 몰아넣으면 접근 제어, 리소스 할당, 네트워크 분리가 한꺼번에 어려워집니다. 팀, 환경, 업무 경계에 맞는 namespace를 나누고 RBAC·NetworkPolicy·ResourceQuota를 결합해야 운영 사고가 퍼지는 범위를 줄일 수 있습니다.

**확인 방법**:
```bash
kubectl get ns
kubectl get deploy,svc,configmap,secret -A
```

**권장 방식으로 변경**:
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: prod-payments
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  namespace: prod-payments
```

`APPLICATION` `ALL` `SECURITY` `MULTI-TENANCY`
---

## [Medium] Pod와 Container 보안 컨텍스트 명시

**설명**: securityContext를 비워두면 루트 실행, 권한 상승, 쓰기 가능한 루트 파일시스템, 과도한 Linux capability 같은 위험한 기본값을 그대로 허용하기 쉽습니다. 서비스 계정 토큰 자동 마운트까지 함께 줄여 두어야 파드가 침해되더라도 측면 이동과 권한 남용 범위를 줄일 수 있습니다.

**확인 방법**:
```bash
kubectl get deploy -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\n"}{"  automountServiceAccountToken="}{.spec.template.spec.automountServiceAccountToken}{"\n"}{range .spec.template.spec.containers[*]}{"  - "}{.name}{": securityContext="}{.securityContext}{"\n"}{end}{end}'
```

**권장 방식으로 변경**:
```yaml
spec:
  template:
    spec:
      automountServiceAccountToken: false
      securityContext:
        seccompProfile:
          type: RuntimeDefault
      containers:
      - name: api
        securityContext:
          runAsNonRoot: true
          runAsUser: 10001
          allowPrivilegeEscalation: false
          readOnlyRootFilesystem: true
          capabilities:
            drop:
            - ALL
```

`APPLICATION` `ALL` `SECURITY`
---

## [Medium] 매니페스트가 모범 사례를 준수하도록 관리

**설명**: 클러스터 내부의 운영 품질은 결국 YAML과 Helm 값에서 시작됩니다. 배포 전에 서버 사이드 검증과 차이 비교를 수행하고, CI에서 라벨·리소스·보안 설정을 강제해야 운영 환경에서만 드러나는 오류를 크게 줄일 수 있습니다.

**확인 방법**:
```bash
kubectl apply --dry-run=server -f ./manifests
kubectl diff -f ./manifests
```

**권장 방식으로 변경**:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
  labels:
    app.kubernetes.io/name: api
spec:
  replicas: 2
  template:
    metadata:
      labels:
        app.kubernetes.io/name: api
    spec:
      automountServiceAccountToken: false
      containers:
      - name: api
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
        securityContext:
          runAsNonRoot: true
          allowPrivilegeEscalation: false
```

`APPLICATION` `ALL` `RESOURCES` `SECURITY`
---

## [Medium] 취약점을 포함한 Docker 이미지 빌드에 임계값 정책 적용

**설명**: 스캔만 하고 빌드를 계속 통과시키면 결과는 참고 자료에 머물고 실제 보안 수준은 개선되지 않습니다. 운영 환경에서는 High/Critical, 공급업체 수정 가능 여부, 예외 승인 프로세스 같은 기준을 정해 자동으로 차단해야 합니다.

**확인 방법**:
```bash
grep -RInE 'exit-code:.*1|--exit-code 1|--fail-on|severity: HIGH,CRITICAL|threshold' .github/workflows azure-pipelines.yml .
```

**권장 방식으로 변경**:
```yaml
- name: Enforce vulnerability threshold
  uses: aquasecurity/trivy-action@0.28.0
  with:
    image-ref: <ACR_NAME>.azurecr.io/api:${GITHUB_SHA}
    severity: HIGH,CRITICAL
    exit-code: '1'
    ignore-unfixed: true
```

`APPLICATION` `ALL` `SECURITY`
---

## [Medium] ARM 또는 Terraform으로 Azure 리소스 자동화

**설명**: 포털 클릭이나 일회성 CLI 실행으로 만든 리소스는 변경 이력과 코드 리뷰가 남지 않아 드리프트가 빠르게 쌓입니다. AKS, ACR, 네트워크, 모니터링 리소스를 IaC로 관리해야 재현성, 감사 가능성, 재배포 속도를 모두 확보할 수 있습니다.

**확인 방법**:
```bash
find . -type f \( -name '*.tf' -o -name '*.bicep' -o -name 'Chart.yaml' \)
grep -RInE 'az aks create|az acr create|az network vnet create' .github/workflows azure-pipelines.yml .
```

**권장 방식으로 변경**:
```bash
terraform init
terraform plan -out tfplan
terraform apply tfplan
```

`APPLICATION` `ALL` `IAC`
---

## [Medium] Canary 또는 Blue/Green 배포 사용

**설명**: 새 버전을 한 번에 100% 배포하면 설정 실수나 성능 문제의 영향 반경이 즉시 전체 사용자로 번집니다. 점진적 배포 전략을 쓰면 일부 트래픽으로 먼저 검증하고, 문제 발생 시 되돌리기 시간을 짧게 유지할 수 있습니다.

**확인 방법**:
```bash
kubectl get deploy -A -o yaml | grep -nE 'strategy:|maxSurge:|maxUnavailable:'
kubectl get svc -A
```

**권장 방식으로 변경**:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: api
spec:
  replicas: 4
  strategy:
    canary:
      steps:
      - setWeight: 20
      - pause:
          duration: 5m
      - setWeight: 50
      - pause:
          duration: 10m
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
```

`APPLICATION` `ALL` `IAC`
---

## [Medium] 신뢰된 레지스트리의 컨테이너만 배포 허용

**설명**: 누구나 접근 가능한 퍼블릭 레지스트리나 오타가 섞인 이미지 경로는 공급망 공격의 진입점이 되기 쉽습니다. 허용된 ACR 또는 승인된 레지스트리 패턴만 배포되도록 강제하면 우회 배포와 검증 누락을 막을 수 있습니다.

**확인 방법**:
```bash
kubectl get validatingwebhookconfigurations | grep -Ei 'kyverno|gatekeeper|azure-policy'
kubectl get pods -A | grep -Ei 'kyverno|gatekeeper|azure-policy'
```

**권장 방식으로 변경**:
```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: allow-only-approved-registries
spec:
  validationFailureAction: Enforce
  background: true
  rules:
  - name: restrict-image-registries
    match:
      any:
      - resources:
          kinds:
          - Pod
    validate:
      message: Only images from approved registries are allowed.
      pattern:
        spec:
          containers:
          - image: "<ACR_NAME>.azurecr.io/*"
```

`CONTAINER` `ALL` `SECURITY`
---

## [Medium] Distroless 이미지 우선 사용

**설명**: Distroless 이미지는 셸과 불필요한 도구, 패키지를 제거해 공격 표면과 CVE 수를 줄여 줍니다. 운영 디버깅 편의성은 약간 떨어지지만, 별도의 디버그 이미지 전략을 두면 보안 이점이 훨씬 큽니다.

**확인 방법**:
```bash
find . -name 'Dockerfile*' -print
grep -RIn '^FROM ' .
```

**권장 방식으로 변경**:
```bash
cat <<'EOF' > Dockerfile
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src
COPY . .
RUN dotnet publish -c Release -o /out

FROM gcr.io/distroless/base-debian12:nonroot
WORKDIR /app
COPY --from=build /out .
USER nonroot:nonroot
ENTRYPOINT ["/app/MyService"]
EOF
```

`CONTAINER` `ALL` `SECURITY`
---

## [Medium] 호스트 OS로 Azure Linux 사용

**설명**: Azure Linux는 AKS에 맞춰 최적화된 1st-party 호스트 OS라서 패치 일관성, 부팅 속도, 공격 표면 측면에서 유리합니다. 운영 중인 노드풀이 Ubuntu 중심이라면 신규 Azure Linux 노드풀을 추가한 뒤 워크로드를 점진적으로 옮기는 방식이 가장 안전합니다.

**확인 방법 (az cli)**:
```bash
az aks nodepool list --resource-group <RG> --cluster-name <AKS_NAME> --query "[].{pool:name,mode:mode,osSKU:osSKU}" --output table
```

**권장 방식으로 변경**:
```bash
az aks nodepool add --resource-group <RG> --cluster-name <AKS_NAME> --name azlinuxnp --node-vm-size Standard_D4ds_v5 --node-count 3 --os-sku AzureLinux --mode User
```

`CLUSTER SECURITY` `ALL` `OPERATIONS` `SECURITY`
---

## [Medium] 규제 산업 요건에 맞게 클러스터 구성

**설명**: 금융·공공·의료 같은 환경은 단순 가동보다 감사 가능성, 장기 지원, 강화된 암호화 기준 충족이 더 중요합니다. 클러스터 티어, 지원 플랜, FIPS 이미지 사용 여부를 함께 점검해야 운영 감사나 규제 대응 시 재작업을 줄일 수 있습니다.

**확인 방법 (az cli)**:
```bash
az aks show --resource-group <RG> --name <AKS_NAME> --query "{skuTier:sku.tier,supportPlan:supportPlan,privateCluster:apiServerAccessProfile.enablePrivateCluster}" --output yaml
az aks nodepool list --resource-group <RG> --cluster-name <AKS_NAME> --query "[].{pool:name,osSKU:osSKU}" --output table
```

**권장 방식으로 변경**:
```bash
az aks update --resource-group <RG> --name <AKS_NAME> --tier standard --k8s-support-plan AKSLongTermSupport
az aks nodepool add --resource-group <RG> --cluster-name <AKS_NAME> --name fipsnp --node-vm-size Standard_D4ds_v5 --node-count 3 --os-sku AzureLinux --enable-fips-image
```

`CLUSTER SECURITY` `ALL` `SECURITY` `OPERATIONS`
---

## [Medium] Kubernetes Dashboard 필요 여부 재검토

**설명**: 대시보드는 편리하지만 운영 클러스터에서는 불필요한 노출면을 늘리기 쉽습니다. 이미 AKS에서 관리형 대시보드 추가 기능은 보안상 권장되지 않으므로, 남아 있다면 제거하고 표준 운영 도구로 대체하는 편이 안전합니다.

**확인 방법 (az cli)**:
```bash
az aks addon list-available --output table | grep kube-dashboard
kubectl get namespace kubernetes-dashboard
```

**권장 방식으로 변경**:
```bash
az aks disable-addons --addons kube-dashboard --name <AKS_NAME> --resource-group <RG>
kubectl delete namespace kubernetes-dashboard
```

`CLUSTER SECURITY` `ALL` `COMPLIANCE` `SECURITY`
---

## [Medium] 취약한 이미지 배포 차단 정책 적용

**설명**: 취약점이 있는 이미지는 빌드 단계에서 걸러도 운영 중 다른 팀이나 경로를 통해 다시 들어올 수 있습니다. Azure Policy와 Defender for Containers를 함께 써서 배포 시점에 차단해야 운영자가 사후 대응 대신 사전 차단으로 전환할 수 있습니다.

**확인 방법 (az cli)**:
```bash
az aks addon show --resource-group <RG> --name <AKS_NAME> --addon azure-policy --output yaml
az security pricing show --name Containers --output jsonc
az policy definition list --query "[?contains(displayName, 'Block vulnerable images')].[displayName,name]" --output table
```

**권장 방식으로 변경**:
```bash
az security pricing create --name Containers --tier standard --subplan P2
az aks addon enable --resource-group <RG> --name <AKS_NAME> --addon azure-policy
az policy assignment create --name block-vulnerable-images --policy <POLICY_DEFINITION_NAME_OR_ID> --scope <AKS_RESOURCE_ID>
```

`CLUSTER SECURITY` `ALL` `SECURITY` `OPERATIONS`
---

## [Medium] Microsoft Defender for Cloud로 클러스터 보안 모니터링

**설명**: 운영 클러스터는 단일 경고보다 중앙 수집과 상관 분석이 중요합니다. Defender for Cloud와 Log Analytics 연결이 되어 있어야 컨트롤 플레인 신호, 취약점, 권고사항을 한 곳에서 추적하고 대응 우선순위를 정할 수 있습니다.

**확인 방법 (az cli)**:
```bash
az security pricing show --name Containers --output jsonc
az security workspace-setting show --name default --output jsonc
az aks show --resource-group <RG> --name <AKS_NAME> --query "securityProfile.defender" --output yaml
```

**권장 방식으로 변경**:
```bash
az security workspace-setting create --name default --target-workspace <LAW_RESOURCE_ID>
az security pricing create --name Containers --tier standard --subplan P2
```

`CLUSTER SECURITY` `ALL` `SECURITY` `OPERATIONS`
---

## [Medium] Azure Policy for Kubernetes로 규정 준수 보장

**설명**: 운영 표준을 문서로만 두면 팀마다 적용 편차가 커집니다. Azure Policy를 연결하면 허용 이미지, 보안 컨텍스트, 레이블 같은 기준을 클러스터 차원에서 지속적으로 강제할 수 있습니다.

**확인 방법 (az cli)**:
```bash
az graph query -q "where type=='microsoft.containerservice/managedclusters' | extend compliant = (isnotnull(properties.addonProfiles.azurepolicy) and properties.addonProfiles.azurepolicy.enabled==true) | distinct id,compliant" --output table
az aks addon show --resource-group <RG> --name <AKS_NAME> --addon azure-policy --output yaml
```

**권장 방식으로 변경**:
```bash
az aks addon enable --resource-group <RG> --name <AKS_NAME> --addon azure-policy
```

`CLUSTER SECURITY` `ALL` `OPERATIONS`
---

## [Medium] user/system 노드풀로 컨트롤 플레인과 애플리케이션 분리

**설명**: 시스템 파드와 애플리케이션 파드가 같은 노드풀을 쓰면 자원 경합이 생겼을 때 클러스터 핵심 기능이 먼저 흔들릴 수 있습니다. 전용 system 노드풀과 taint를 두면 업그레이드, 장애, 확장 작업을 더 예측 가능하게 운영할 수 있습니다.

**확인 방법 (az cli)**:
```bash
az graph query -q "where type=='microsoft.containerservice/managedclusters' | project id,resourceGroup,name,pools=properties.agentPoolProfiles | project id,name,resourceGroup,poolcount=array_length(pools) | extend compliant = (poolcount > 1)" --output table
az aks nodepool list --resource-group <RG> --cluster-name <AKS_NAME> --query "[].{pool:name,mode:mode,taints:nodeTaints}" --output table
```

**권장 방식으로 변경**:
```bash
az aks nodepool add --resource-group <RG> --cluster-name <AKS_NAME> --name systemnp --node-vm-size Standard_D4ds_v5 --node-count 3 --mode System --os-sku AzureLinux --node-taints CriticalAddonsOnly=true:NoSchedule
```

`CLUSTER SECURITY` `ALL` `COMPLIANCE` `OPERATIONS`
---

## [Medium] Microsoft Defender for Cloud로 보안 상태 취약점 탐지

**설명**: 운영 리뷰에서는 경고 개수보다 어떤 통제가 빠졌는지 파악하는 보안 상태 점검이 더 중요합니다. Defender for Cloud의 assessment와 secure score를 주기적으로 확인하면 미적용 정책, 오래된 구성, 과도한 권한 같은 구조적 문제를 빠르게 찾을 수 있습니다.

**확인 방법 (az cli)**:
```bash
az security assessment list --output table | grep <AKS_CLUSTER_NAME>
az security secure-score-controls list --output table
```

**권장 방식으로 변경**:
```bash
az security pricing create --name Containers --tier standard --subplan P2
az security workspace-setting create --name default --target-workspace <LAW_RESOURCE_ID>
```

`CLUSTER SECURITY` `ALL` `COMPLIANCE`
---

## [Medium] 비밀번호 없이 AKS와 ACR 통합 사용

**설명**: 이미지 pull 자격 증명을 시크릿으로 배포하는 방식은 누출과 만료 대응 모두에 취약합니다. AKS-ACR 통합을 쓰면 클러스터 정체성으로 pull 권한을 부여할 수 있어 운영 중 패스워드 배포를 없앨 수 있습니다.

**확인 방법 (az cli)**:
```bash
az aks show --resource-group <RG> --name <AKS_NAME> --query "servicePrincipalProfile.clientId" --output tsv
az aks check-acr --resource-group <RG> --name <AKS_NAME> --acr <ACR_LOGIN_SERVER>
```

**권장 방식으로 변경**:
```bash
az aks update --resource-group <RG> --name <AKS_NAME> --attach-acr <ACR_NAME_OR_RESOURCE_ID>
```

`IDENTITY` `ALL` `SECURITY`
---

## [Medium] admin kubeconfig(get-credentials --admin) 접근 최소화

**설명**: admin kubeconfig는 Kubernetes RBAC를 우회하는 강한 권한이어서 운영상 꼭 필요한 사람에게만 열어야 합니다. 클러스터 관리자 역할 할당을 정기적으로 검토하고, 일반 운영은 사용자 kubeconfig와 Azure RBAC 기반으로 수행하는 것이 안전합니다.

**확인 방법 (az cli)**:
```bash
az role assignment list --scope <AKS_RESOURCE_ID> --query "[?roleDefinitionName=='Azure Kubernetes Service Cluster Admin Role'].[principalName,roleDefinitionName]" --output table
az aks show --resource-group <RG> --name <AKS_NAME> --query "disableLocalAccounts" --output tsv
```

**권장 방식으로 변경**:
```bash
az role assignment delete --assignee <OBJECT_ID_OR_UPN> --role "Azure Kubernetes Service Cluster Admin Role" --scope <AKS_RESOURCE_ID>
az aks update --resource-group <RG> --name <AKS_NAME> --disable-local-accounts
```

`IDENTITY` `ALL` `SECURITY`
---

## [Medium] AKS 비대화형 로그인에는 kubelogin 사용

**설명**: CI/CD나 자동화 계정이 사람용 kubeconfig 흐름을 재사용하면 토큰 만료나 캐시 문제로 배포 안정성이 떨어집니다. exec 형식 kubeconfig와 kubelogin을 사용하면 비대화형 인증을 표준화하고 파이프라인 재현성도 높일 수 있습니다.

**확인 방법 (az cli)**:
```bash
az aks get-credentials --resource-group <RG> --name <AKS_NAME> --format exec --file -
```

**권장 방식으로 변경**:
```bash
az aks get-credentials --resource-group <RG> --name <AKS_NAME> --format exec --overwrite-existing
kubelogin convert-kubeconfig -l spn
```

`IDENTITY` `ALL` `SECURITY`
---

## [Medium] AKS 로컬 계정 비활성화

**설명**: 로컬 계정은 Entra ID, 조건부 액세스, 중앙 감사 정책 밖에서 관리되는 우회 경로가 될 수 있습니다. 운영 클러스터에서는 가능한 한 로컬 계정을 끄고 중앙 ID 체계만 남기는 편이 보안 검토와 사고 대응에 유리합니다.

**확인 방법 (az cli)**:
```bash
az aks show --resource-group <RG> --name <AKS_NAME> --query "disableLocalAccounts" --output tsv
```

**권장 방식으로 변경**:
```bash
az aks update --resource-group <RG> --name <AKS_NAME> --disable-local-accounts
```

`IDENTITY` `ALL` `SECURITY`
---

## [Medium] 노드 크기 산정

**설명**: 노드 VM 크기를 잘못 고르면 과소 할당 시 성능 병목이, 과대 할당 시 비용 낭비가 반복됩니다. 현재 노드풀 크기와 autoscaler 범위를 같이 보고 워크로드 특성에 맞는 표준 SKU로 재정렬하는 것이 운영 안정성과 비용 모두에 중요합니다.

**확인 방법 (az cli)**:
```bash
az aks nodepool list --resource-group <RG> --cluster-name <AKS_NAME> --query "[].{pool:name,vmSize:vmSize,count:count,mode:mode,minCount:minCount,maxCount:maxCount}" --output table
az vm list-skus --location <REGION> --resource-type virtualMachines --size Standard_D --output table
```

**권장 방식으로 변경**:
```bash
az aks nodepool add --resource-group <RG> --cluster-name <AKS_NAME> --name apps01 --node-vm-size Standard_D4ds_v5 --node-count 3 --enable-cluster-autoscaler --min-count 3 --max-count 10 --os-sku AzureLinux
```

`RESOURCE MANAGEMENT` `ALL` `RESOURCES`
---

## [Medium] 가능하면 ARM64 노드 사용

**설명**: ARM64 호환 워크로드는 같은 성능 대비 비용과 전력 효율 측면에서 이점을 얻는 경우가 많습니다. 다만 이미지 아키텍처와 운영 에이전트 호환성을 먼저 확인한 뒤 전용 노드풀로 단계 도입하는 방식이 안전합니다.

**확인 방법 (az cli)**:
```bash
az vm list-skus --location <REGION> --resource-type virtualMachines --size Standard_Dps_v5 --output table
az aks nodepool list --resource-group <RG> --cluster-name <AKS_NAME> --query "[].{pool:name,vmSize:vmSize,osSKU:osSKU}" --output table
```

**권장 방식으로 변경**:
```bash
az aks nodepool add --resource-group <RG> --cluster-name <AKS_NAME> --name arm64np --node-vm-size Standard_D4ps_v5 --node-count 3 --os-sku AzureLinux --mode User
```

`RESOURCE MANAGEMENT` `ALL` `OPERATIONS`
---

## [Medium] 모든 컨테이너에 메모리 requests/limits 설정

**설명**: 메모리 요청량과 제한이 없으면 일부 워크로드의 급증이 같은 노드의 다른 서비스까지 연쇄적으로 흔들 수 있습니다. 운영 클러스터는 최소한 메모리 requests/limits를 표준화하고, 팀별 기본값은 LimitRange나 배포 파이프라인으로 강제하는 편이 좋습니다.

**확인 방법 (az cli)**:
```bash
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"/"}{.metadata.name}{"\t"}{range .spec.containers[*]}{.name}{":req="}{.resources.requests.memory}{",limit="}{.resources.limits.memory}{" "}{end}{"\n"}{end}'
kubectl get limitrange -A
```

**권장 방식으로 변경**:
```bash
kubectl set resources deployment <DEPLOYMENT_NAME> -n <NAMESPACE> --containers="*" --requests=memory=256Mi --limits=memory=512Mi
```

`RESOURCE MANAGEMENT` `ALL` `RESILIENCY` `MULTI-TENANCY`
---

## [Medium] 필요 시 AKS 클러스터에서 GPU MIG 사용

**설명**: A100 계열 GPU를 여러 워크로드가 나눠 써야 한다면 MIG로 자원을 세분화해 낭비를 줄일 수 있습니다. 단일 대형 GPU를 통째로 묶어 두기보다, 실제 추론·학습 패턴에 맞는 인스턴스 프로파일을 별도 노드풀로 설계하는 편이 운영 효율이 높습니다.

**확인 방법 (az cli)**:
```bash
az aks nodepool list --resource-group <RG> --cluster-name <AKS_NAME> --query "[].{pool:name,vmSize:vmSize,gpuInstanceProfile:gpuInstanceProfile}" --output table
```

**권장 방식으로 변경**:
```bash
az aks nodepool add --resource-group <RG> --cluster-name <AKS_NAME> --name migpool1 --node-vm-size Standard_ND96asr_v4 --node-count 1 --os-sku AzureLinux --gpu-instance-profile MIG1g
```

`RESOURCE MANAGEMENT` `ALL` `COST` `FINOPS`
---

## [Medium] Dev/Test 클러스터라면 NodePool Start/Stop 사용

**설명**: 개발·검증용 워크로드는 24시간 계속 돌릴 이유가 없는 경우가 많습니다. 안 쓰는 시간대에 노드풀을 중지해 두면 비용을 줄이면서도 필요한 순간에만 빠르게 재가동할 수 있습니다.

**확인 방법 (az cli)**:
```bash
az aks nodepool list --resource-group <RG> --cluster-name <AKS_NAME> --output table
```

**권장 방식으로 변경**:
```bash
az aks nodepool stop --resource-group <RG> --cluster-name <AKS_NAME> --name <NODEPOOL_NAME>
az aks nodepool start --resource-group <RG> --cluster-name <AKS_NAME> --name <NODEPOOL_NAME>
```

`RESOURCE MANAGEMENT` `ALL` `COST` `FINOPS`
---

## [Medium] 멀티리전 배포 계획 수립

**설명**: 영역 중복만으로는 지역 단위 장애를 막을 수 없습니다. 운영 리뷰에서는 보조 리전의 클러스터, 이미지, DNS 전환, 데이터 복제 방식까지 함께 확인해야 실제 DR이 가능합니다. 최소한 paired region 또는 지연이 허용되는 대체 리전을 사전에 정해 두는 것이 좋습니다.

**확인 방법**:
```bash
az aks list --query "[].{name:name,location:location,tier:sku.tier}" -o table
az acr replication list -r $ACR_NAME -o table
```

**권장 방식으로 변경**:
```bash
az acr replication create -r $ACR_NAME -g $ACR_RG -l eastus2
az aks create -g $DR_RG -n $DR_CLUSTER --tier standard --generate-ssh-keys
```

`BC/DR` `ALL` `RESILIENCY`
---

## [Medium] 사설 레지스트리(ACR)를 사용한다면 다중 리전에 복제 구성

**설명**: 이미지가 한 리전에만 있으면 클러스터는 살아 있어도 새 Pod가 올라오지 않을 수 있습니다. 멀티리전 AKS에서는 이미지 공급망도 같은 수준으로 다중화해야 배포와 스케일아웃이 계속 가능합니다. 특히 DR 리전은 평소에도 복제가 정상인지 주기적으로 확인해야 합니다.

**확인 방법**:
```bash
az acr replication list -r $ACR_NAME -g $ACR_RG -o table
az acr show -n $ACR_NAME -g $ACR_RG --query "{sku:sku.name,location:location}" -o yaml
```

**권장 방식으로 변경**:
```bash
az acr replication create -r $ACR_NAME -g $ACR_RG -l eastus2
```

`BC/DR` `ALL` `RESILIENCY`
---

## [Medium] 레지스트리에 영역 중복(zone redundancy) 구성

**설명**: 같은 리전 안에서도 ACR이 단일 영역 의존이면 이미지 pull 경로가 약해질 수 있습니다. 영역 중복을 켜 두면 노드 교체나 대규모 스케일아웃 시 레지스트리 가용성 리스크를 줄일 수 있습니다. 특히 Premium SKU를 쓰는 프로덕션 레지스트리라면 기본 검토 항목으로 보는 것이 좋습니다.

**확인 방법**:
```bash
az acr show -n $ACR_NAME -g $ACR_RG --query zoneRedundancy -o tsv
az acr replication show -n eastus -r $ACR_NAME -g $ACR_RG --query zoneRedundancy -o tsv
```

**권장 방식으로 변경**:
```bash
az acr create -n $NEW_ACR_NAME -g $ACR_RG --sku Premium --zone-redundancy enabled
az acr replication create -r $NEW_ACR_NAME -g $ACR_RG -l eastus2 --zone-redundancy enabled
```

`BC/DR` `ALL` `RESILIENCY`
---

## [Medium] 스토리지 요구사항에 맞게 노드 크기 산정

**설명**: 노드 크기는 CPU와 메모리만의 문제가 아니라 디스크 수, 네트워크 대역폭, 처리 가능한 IOPS 한계와도 연결됩니다. 특히 PVC가 많은 워크로드는 작은 VM 크기에서 첨부 디스크 수 제한에 먼저 걸릴 수 있습니다. 스토리지 병목은 보통 Pod 장애보다 늦게 드러나므로 사전 용량 검토가 중요합니다.

**확인 방법**:
```bash
az aks nodepool show -g $RG --cluster-name $CLUSTER -n $NODEPOOL --query "{vmSize:vmSize,osDiskSizeGB:osDiskSizeGb,maxPods:maxPods}" -o yaml
kubectl get pvc -A
```

**권장 방식으로 변경**:
```bash
az aks nodepool add -g $RG --cluster-name $CLUSTER -n storagepool --node-vm-size Standard_D8s_v5 --node-osdisk-size 256 --max-pods 110
```

`STORAGE` `ALL`
---

## [Medium] 데이터를 안전하게 보호하고 백업

**설명**: 스토리지 운영에서 가장 자주 놓치는 부분은 복원 경로와 보존 기간입니다. 디스크, 파일 공유, 데이터베이스마다 백업 도구와 복구 단위가 다르므로 워크로드별로 방법을 분리해야 합니다. 보안 측면에서는 최소 권한 접근과 백업 산출물의 위치까지 함께 검토해야 합니다.

**확인 방법**:
```bash
kubectl get pvc,pv -A
az snapshot list -g $NODE_RG -o table
```

**권장 방식으로 변경**:
```bash
DISK_ID=$(az disk show -g $NODE_RG -n $DISK_NAME --query id -o tsv)
az snapshot create -g $BACKUP_RG -n ${DISK_NAME}-$(date +%Y%m%d) --source "$DISK_ID" -l $LOCATION
```

`STORAGE` `ALL` `RESILIENCY`
---

## [Medium] 클러스터 내부에 상태를 두지 말고 외부(Azure Storage, SQL, Cosmos 등)에 저장

**설명**: 이 항목은 데이터베이스뿐 아니라 세션, 캐시, 업로드 파일처럼 애매한 상태까지 포함해 다시 보는 것이 좋습니다. 운영 중에는 이런 부가 상태가 점점 늘어나며, 장애 시 복구 순서를 복잡하게 만듭니다. 애플리케이션이 외부 상태 저장소를 기본값으로 사용하도록 바꾸면 롤링 업그레이드와 노드 교체가 훨씬 단순해집니다.

**확인 방법**:
```bash
kubectl get deployment,statefulset -A
kubectl get pvc -A
```

**권장 방식으로 변경**:
```bash
kubectl set env deployment/web SESSION_STORE=redis REDIS_HOST=$REDIS_HOST
kubectl rollout restart deployment/web
```

`STORAGE` `ALL` `RESILIENCY`
---

## [Medium] Azure Disk와 가용성 영역을 함께 쓴다면 영역 고정 nodepool 또는 ZRS 디스크 고려

**설명**: LRS 디스크는 기본적으로 단일 영역에 묶이므로 다중 영역 노드풀과 함께 쓰면 스케줄링 실패가 발생할 수 있습니다. 따라서 zonal nodepool에 맞춰 디스크를 프로비저닝하거나, 여러 영역을 가로지르는 설계라면 ZRS 기반 전략을 검토해야 합니다. StorageClass의 VolumeBindingMode 설정이 이 문제를 실제로 좌우합니다.

**확인 방법**:
```bash
kubectl get storageclass -o yaml
kubectl get nodes -L topology.kubernetes.io/zone
```

**권장 방식으로 변경**:
```bash
cat <<'EOF2' > managed-csi-wffc.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: managed-csi-zonal
provisioner: disk.csi.azure.com
parameters:
  skuName: Premium_LRS
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
EOF2
kubectl apply -f managed-csi-wffc.yaml

az disk create -g $NODE_RG -n $DISK_NAME --size-gb 256 --sku Premium_LRS -l $LOCATION --zone 1
```

`STORAGE` `ALL` `RESILIENCY`
---

## [Medium] 애플리케이션이 갑작스러운 종료를 견디도록 준비

**설명**: Windows 컨테이너에서는 Linux처럼 종료 유예 시간이 항상 기대대로 동작하지 않을 수 있습니다. 따라서 종료 직전 한 번에 정리하는 구조보다, 평소에 자주 체크포인트를 남기고 재시작을 전제로 설계하는 편이 안전합니다. 운영 관점에서는 단일 Pod 의존성을 낮추고 롤링 업데이트 실패 시 영향 범위를 줄이는 구성이 중요합니다.

**확인 방법**:
```bash
kubectl get deployment win-app -n $NS -o yaml
kubectl get pdb -A
```

**권장 방식으로 변경**:
```bash
kubectl patch deployment win-app -n $NS --type merge -p '{"spec":{"replicas":2,"strategy":{"rollingUpdate":{"maxUnavailable":0,"maxSurge":1}}}}'
kubectl apply -f - <<'EOF2'
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: win-app-pdb
  namespace: default
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: win-app
EOF2
```

`WINDOWS` `ALL`
---

## [Medium] Privileged 컨테이너를 사용하지 않기

**설명**: Windows 워크로드는 Linux의 privileged 모델과 다르므로 같은 운영 패턴을 그대로 가져오면 오해가 생깁니다. 보안 리뷰에서는 HostProcess가 정말 필요한지와 일반 애플리케이션이 과도한 호스트 권한을 요구하지 않는지를 분리해서 봐야 합니다. 기본값은 비권한 실행이며, 예외만 별도 격리하는 것이 안전합니다.

**확인 방법**:
```bash
kubectl get pods -A -o yaml | grep -n "hostProcess\|privileged"
```

**권장 방식으로 변경**:
```bash
kubectl patch deployment win-agent -n $NS --type merge -p '{"spec":{"template":{"spec":{"containers":[{"name":"agent","securityContext":{"windowsOptions":{"hostProcess":false}}}]}}}}'
```

`WINDOWS` `ALL`
---

## [Medium] 메모리 사용량 지속 모니터링

**설명**: Windows 노드는 메모리 압박 시 Linux와 다른 방식으로 성능 저하가 나타날 수 있고, 페이지 파일 사용이 늘면 지연이 급격히 커집니다. 따라서 Pod별 메모리 요청·제한이 비현실적이지 않은지와 실제 사용량이 얼마나 변동하는지를 함께 봐야 합니다. 문제는 보통 OOM 이벤트보다 먼저 응답 지연으로 드러납니다.

**확인 방법**:
```bash
kubectl top pods -A --containers
kubectl get deployment -A -o yaml | grep -n "resources:"
```

**권장 방식으로 변경**:
```bash
kubectl set resources deployment/win-api -n $NS -c app --requests=memory=1Gi --limits=memory=2Gi
```

`WINDOWS` `ALL`
---

## [Medium] Windows Server 노드에 gMSA 사용

**설명**: 도메인 자원을 써야 하는 Windows 워크로드는 일반 비밀값보다 gMSA가 운영과 보안 모두에서 낫습니다. 비밀번호 회전과 SPN 관리 부담을 줄이면서 여러 Pod가 같은 도메인 ID를 안전하게 사용할 수 있기 때문입니다. 온프레미스 AD 또는 하이브리드 환경과 연계된 서비스라면 특히 검토 가치가 큽니다.

**확인 방법**:
```bash
az aks show -g $RG -n $CLUSTER --query windowsProfile.gmsaProfile -o yaml
kubectl get gmsacredentialspec -A
```

**권장 방식으로 변경**:
```bash
az aks update -g $RG -n $CLUSTER --enable-windows-gmsa --gmsa-dns-server 10.0.0.10 --gmsa-root-domain-name contoso.com
kubectl apply -f - <<'EOF2'
apiVersion: windows.k8s.io/v1
kind: GMSACredentialSpec
metadata:
  name: web-gmsa
credspec: |
  {}
EOF2
```

`WINDOWS` `ALL` `IDENTITY` `SECURITY`
---

## [Medium] 필요한 경우 HostProcess 컨테이너 사용

**설명**: HostProcess는 노드 수준 작업이 꼭 필요할 때만 제한적으로 써야 하는 강한 권한 모델입니다. 보안 제품, 노드 부트스트랩, 진단 에이전트처럼 이유가 명확한 경우에만 별도 노드풀과 taint로 격리하는 것이 좋습니다. 일반 업무 애플리케이션까지 HostProcess에 기대기 시작하면 운영 복잡도가 급격히 커집니다.

**확인 방법**:
```bash
kubectl get pods -A -o yaml | grep -n "hostProcess"
az aks nodepool show -g $RG --cluster-name $CLUSTER -n $WIN_POOL --query "{osType:osType,osSKU:osSKU}" -o yaml
```

**권장 방식으로 변경**:
```bash
az aks nodepool add -g $RG --cluster-name $CLUSTER -n hostproc --os-type Windows --os-sku Windows2022 --node-taints hostprocess=true:NoSchedule
kubectl apply -f - <<'EOF2'
apiVersion: v1
kind: Pod
metadata:
  name: hostprocess-example
spec:
  nodeSelector:
    kubernetes.io/os: windows
  tolerations:
  - key: hostprocess
    operator: Equal
    value: "true"
    effect: NoSchedule
  hostNetwork: true
  containers:
  - name: hostprocess
    image: mcr.microsoft.com/windows/nanoserver:ltsc2022
    command:
    - powershell.exe
    - -Command
    - Start-Sleep -Seconds 3600
    securityContext:
      windowsOptions:
        hostProcess: true
        runAsUserName: "NT AUTHORITY\\SYSTEM"
  restartPolicy: Never
EOF2
```

`WINDOWS` `ALL` `APPLICATION`
---

## [Medium] Windows 2019/2022 노드에서는 Calico Network Policy 활용

**설명**: Windows 워크로드에서 네트워크 격리가 필요하다면 클러스터 수준에서 지원되는 정책 엔진을 먼저 확인해야 합니다. Calico를 사용할 수 있는 환경이라면 최소한 네임스페이스·애플리케이션 간 기본 차단 규칙을 적용해 lateral movement를 줄이는 것이 좋습니다. 다만 기존 클러스터의 네트워크 구성과 호환성을 먼저 검토해야 합니다.

**확인 방법**:
```bash
az aks show -g $RG -n $CLUSTER --query "{networkPlugin:networkProfile.networkPlugin,networkPolicy:networkProfile.networkPolicy}" -o yaml
az graph query -q "where type=='microsoft.containerservice/managedclusters' | where isnotnull(properties.apiServerAccessProfile.enablePrivateCluster) | extend compliant = (properties.apiServerAccessProfile.enablePrivateCluster==true) | distinct id, compliant" -o table
kubectl get networkpolicy -A
```

**권장 방식으로 변경**:
```bash
az aks update -g $RG -n $CLUSTER --network-policy calico
kubectl apply -f - <<'EOF2'
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-frontend
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: frontend
EOF2
```

`WINDOWS` `ALL` `SECURITY` `COMPLIANCE`
---

## [Medium] Windows 워크로드에는 Accelerated Networking 사용

**설명**: Windows 노드는 네트워크 오버헤드의 영향을 크게 받기 때문에 지원되는 VM 크기를 선택해 지연과 CPU 부담을 줄이는 것이 좋습니다. 특히 이미지 pull, east-west 통신, SMB 기반 워크로드가 섞이면 네트워크 효율 차이가 운영 안정성으로 이어집니다. 신규 노드풀을 설계할 때는 VM SKU가 가속 네트워킹을 지원하는지 먼저 확인해야 합니다.

**확인 방법**:
```bash
VM_SIZE=$(az aks nodepool show -g $RG --cluster-name $CLUSTER -n $WIN_POOL --query vmSize -o tsv)
az vm list-skus -l $LOCATION --size $VM_SIZE -o json
az graph query -q "where type=='microsoft.containerservice/managedclusters' | extend compliant = (isnull(properties.addonProfiles.httpApplicationRouting) or properties.addonProfiles.httpApplicationRouting.enabled==false) | distinct id,compliant" -o table
```

**권장 방식으로 변경**:
```bash
az aks nodepool add -g $RG --cluster-name $CLUSTER -n winfast --os-type Windows --os-sku Windows2022 --node-vm-size Standard_D4s_v5
```

`WINDOWS` `ALL` `OPERATIONS`
---

## [Medium] Node Autoprovisioning(NAP, Karpenter 기반)으로 노드 프로비저닝 자동화 검토

**설명**: 기존 Cluster Autoscaler는 미리 정의한 node pool 내에서만 스케일하지만, Node Autoprovisioning(NAP)은 Karpenter 기반으로 워크로드의 실제 요구사항(vCPU/메모리/GPU 등)에 맞춰 최적의 VM SKU를 가진 노드를 즉시 프로비저닝한다. 체크리스트 사이트의 "Cluster Autoscaler 사용" 항목보다 한 단계 진보된 최신(Preview) 기능으로, 다양한 워크로드 크기가 혼재된 클러스터에서 노드 활용률과 비용 효율을 높일 수 있다.

**확인 방법 (az cli)**:
```bash
az aks show -g $RESOURCE_GROUP -n $CLUSTER \
  --query "nodeProvisioningProfile.mode" -o tsv
```

**권장 방식으로 변경**:
```bash
# Preview 기능 — 기존 클러스터 적용 전 워크로드 호환성 검토 필요
az aks update -g $RESOURCE_GROUP -n $CLUSTER --node-provisioning-mode Auto
```

`ADDITIONAL` `OPERATIONS` `COST` `SCALING`
---

## [Medium] Vertical Pod Autoscaler(VPA)로 request/limit 자동 튜닝

**설명**: 체크리스트 사이트는 "리소스 request/limit을 설정하라"는 정적 권장만 다루지만, 실제 운영에서는 워크로드 특성이 바뀌며 초기 설정값이 금방 부정확해진다. VPA는 실측 사용량 기반으로 request/limit을 자동으로 추천하거나(Off 모드) 재시작을 통해 실제로 조정(Auto/Initial 모드)해주는 AKS 관리형 add-on이다.

**확인 방법 (az cli)**:
```bash
az aks show -g $RESOURCE_GROUP -n $CLUSTER \
  --query "workloadAutoScalerProfile.verticalPodAutoscaler.enabled" -o tsv

kubectl get vpa -A
```

**권장 방식으로 변경**:
```bash
az aks update -g $RESOURCE_GROUP -n $CLUSTER --enable-vpa
```
```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: my-app-vpa
spec:
  targetRef:
    apiVersion: "apps/v1"
    kind: Deployment
    name: my-app
  updatePolicy:
    updateMode: "Off"   # 처음엔 권장값만 확인(Off), 검증 후 Auto로 전환
```

`ADDITIONAL` `OPERATIONS` `COST` `APPLICATION`
---

## [Medium] KEDA 관리형 add-on으로 이벤트 기반 오토스케일링

**설명**: HPA는 CPU/메모리 등 리소스 메트릭 기반 스케일링만 다루지만, 메시지 큐 길이·이벤트 허브 lag·HTTP 요청 수 등 이벤트 기반 스케일링이 필요한 워크로드(배치, 큐 컨슈머 등)는 KEDA가 필요하다. AKS는 KEDA를 서드파티 설치 없이 관리형 add-on으로 제공한다. 체크리스트 사이트에는 HPA 항목만 있고 KEDA는 없다.

**확인 방법 (az cli)**:
```bash
az aks show -g $RESOURCE_GROUP -n $CLUSTER \
  --query "workloadAutoScalerProfile.keda.enabled" -o tsv

kubectl get scaledobjects -A
```

**권장 방식으로 변경**:
```bash
az aks update -g $RESOURCE_GROUP -n $CLUSTER --enable-keda
```

`ADDITIONAL` `OPERATIONS` `APPLICATION` `SCALING`
---

## [Medium] AKS Long Term Support(LTS) 플랜으로 안정성이 중요한 클러스터의 패치 기간 연장

**설명**: 일반 AKS Kubernetes 버전은 표준 지원 기간이 지나면 강제 업그레이드 대상이 되지만, 자주 업그레이드하기 어려운 레거시/규제 워크로드가 있다면 LTS 플랜으로 전환해 CVE 패치를 1년 추가로 받을 수 있다(비용 추가 발생). 체크리스트 사이트는 "정기적으로 업그레이드하라"는 원론적 권장만 있고 LTS 옵션 자체는 다루지 않는다.

**확인 방법 (az cli)**:
```bash
az aks show -g $RESOURCE_GROUP -n $CLUSTER \
  --query "supportPlan" -o tsv
```

**권장 방식으로 변경**:
```bash
# 업그레이드 주기를 짧게 못 가져가는 클러스터에 한해 검토 (추가 비용 발생)
az aks update -g $RESOURCE_GROUP -n $CLUSTER --k8s-support-plan AKSLongTermSupport
```

`ADDITIONAL` `OPERATIONS` `BC/DR`
---

## [Medium] Azure Linux로 node OS 전환 검토 (공격 표면 축소)

**설명**: Azure Linux(구 CBL-Mariner)는 Microsoft가 컨테이너 워크로드 전용으로 최소화·하드닝한 리눅스 배포판으로, Ubuntu 대비 설치 패키지 수가 훨씬 적어 공격 표면과 CVE 노출이 줄어든다. 체크리스트 사이트는 일반적인 "노드 OS 최신화"만 언급하고 OS SKU 선택 자체는 다루지 않는다. 기존 node pool은 in-place 전환이 불가하므로 신규 node pool 추가 후 워크로드 이관이 필요하다.

**확인 방법 (az cli)**:
```bash
az aks nodepool list -g $RESOURCE_GROUP --cluster-name $CLUSTER \
  --query "[].{name:name, osSku:osSku, osType:osType}" -o table
```

**권장 방식으로 변경**:
```bash
# 기존 node pool은 os-sku 변경 불가 → 신규 node pool을 추가하고 워크로드를 옮긴 뒤 기존 pool 제거
az aks nodepool add -g $RESOURCE_GROUP --cluster-name $CLUSTER \
  --name azurelinuxpool --os-sku AzureLinux --node-count 3
```

`ADDITIONAL` `SECURITY` `CONTAINER` `OPERATIONS`
---


# AKS Artifact Streaming (Linux) 적용 가이드

Artifact Streaming은 Azure Container Registry(ACR)의 이미지를 AKS 노드로 스트리밍해, Pod 시작에 필요한 레이어만 우선 받아오도록 하는 기능이다. 대형 이미지일수록 최초 이미지 pull → 컨테이너 시작까지의 시간을 크게 줄일 수 있다.

> **GA 상태**: Linux(AMD64) 노드풀 기준으로 GA되었다 ([공식 문서](https://learn.microsoft.com/en-us/azure/aks/artifact-streaming), [발표](https://aka.ms/aks/artifact-streaming)). 단, GA 사용을 위해서는 **az cli 2.87.0 이상**이 필요하다. 2026-07 기준 검증에 사용한 로컬 az cli(2.86.0)에서는 관련 명령어가 여전히 `[Preview]`로 표시되므로, 적용 전 반드시 `az --version`으로 로컬 CLI 버전을 확인하고 `az upgrade`로 최신화할 것.

---

## 1. 사용 전제 조건 (지원 안 되는 시나리오 포함)

### 필수 조건
| 항목 | 요구 사항 |
|---|---|
| Azure CLI | 2.87.0 이상 (`az --version`으로 확인) |
| AKS 클러스터 | Kubernetes 1.25 이상 |
| ACR SKU | **Premium SKU만 지원** (Standard/Basic 불가) |
| 노드 OS | Ubuntu 20.04 이상, 또는 Azure Linux |
| 이미지 아키텍처 | **Linux AMD64만 지원** |
| 클러스터-ACR 연동 | AKS-ACR 통합(`--attach-acr`)이 되어 있어야 함 |
| 권한 | 노드풀 구성 변경을 위해 Azure Kubernetes Service Contributor 역할 필요 |

### 지원되지 않는 시나리오 (반드시 사전 확인)
- **Windows 컨테이너 이미지** — 미지원 (Linux AMD64만 지원)
- **ARM64 이미지** — 미지원 (멀티 아키텍처 이미지도 AMD64만 스트리밍 대상)
- **Standard/Basic SKU ACR** — Premium SKU만 지원, 하위 티어는 기능 자체가 노출되지 않음
- **CMK(Customer-Managed Key)로 암호화된 ACR** — 미지원
- **이미지 digest(`@sha256:...`) 기준 pull** — 미지원, **태그(tag) 기준 pull만 지원**. GitOps/Flux 등이 digest pinning을 쓰는 경우 스트리밍이 적용되지 않음
- **`imagePullSecrets` 기반 인증(regcred)** — 미지원. Entra 기반 AKS-ACR 통합(kubelet identity)이 아닌 non-Entra 스코프 토큰이나 ACR admin 계정 자격 증명으로 pull하면 자동으로 일반 pull로 폴백됨(에러는 아니지만 스트리밍 혜택 없음)
- **지리적 복제(geo-replication)된 레지스트리** — 현재 스트리밍 아티팩트 생성/동기화가 지원되지 않음
- **30GB 초과 대형 이미지** — 기술적으로는 동작하지만 체감 효과가 줄어듦. 대용량 정적 파일은 이미지에 넣기보다 볼륨 마운트를 권장

### 사전 확인 명령어 (az cli)
```bash
# 1) 로컬 CLI 버전 확인 (2.87.0 미만이면 upgrade 필요)
az --version

# 2) 클러스터 Kubernetes 버전 확인 (1.25 이상)
az aks show -g $RESOURCE_GROUP -n $CLUSTER --query kubernetesVersion -o tsv

# 3) ACR SKU 확인 (Premium이어야 함)
az acr show --name $ACR_NAME --query sku.name -o tsv

# 4) AKS-ACR 통합 여부 확인
az aks check-acr --resource-group $RESOURCE_GROUP --name $CLUSTER --acr $ACR_NAME.azurecr.io

# 5) 대상 노드풀의 OS/아키텍처 확인
az aks nodepool list -g $RESOURCE_GROUP --cluster-name $CLUSTER \
  --query "[].{pool:name, osType:osType, osSKU:osSKU}" -o table

# 6) 배포 매니페스트가 digest pinning을 쓰는지 확인 (스트리밍 미적용 대상 식별)
kubectl get deployments -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}{"\t"}{range .spec.template.spec.containers[*]}{.image}{" "}{end}{"\n"}{end}' | grep '@sha256'
```

---

## 2. 적용 전 이미지 배포 시간 측정 (베이스라인)

구성 전/후 효과를 비교하려면 **동일 이미지, 동일 노드 조건**에서 아래 시나리오로 베이스라인을 먼저 측정해야 한다. 캐시된 이미지가 있으면 효과 차이를 왜곡하므로, 신규 노드(또는 이미지가 없는 노드)에서 최초 pull 기준으로 측정하는 것이 핵심이다.

### 2-1. 측정 대상 노드 준비 (캐시 영향 배제)
```bash
# 테스트 전용 노드풀을 별도로 만들어 기존 워크로드와 캐시 상태를 분리
az aks nodepool add -g $RESOURCE_GROUP --cluster-name $CLUSTER \
  --name streamtest --node-count 1 --node-vm-size Standard_D4ds_v5 \
  --mode User --os-sku Ubuntu

# 측정 대상 Pod만 이 노드풀에 배치되도록 taint 부여(선택)
az aks nodepool update -g $RESOURCE_GROUP --cluster-name $CLUSTER \
  --name streamtest --node-taints stream-test=true:NoSchedule
```

### 2-2. 배포 후 Pod 이벤트 타임스탬프로 시간 측정
Kubernetes 이벤트에는 `Pulling`(이미지 pull 시작), `Pulled`(pull 완료), `Started`(컨테이너 시작) 타임스탬프가 남는다. 이 세 시점의 차이로 순수 pull 소요 시간과 pod 준비(Ready)까지의 전체 시간을 각각 측정한다.

```bash
# 1) 측정 대상 배포 (테스트 노드풀로 스케줄링)
kubectl run streaming-test --image=$ACR_NAME.azurecr.io/$REPOSITORY:$TAG \
  --overrides='{"spec":{"tolerations":[{"key":"stream-test","operator":"Equal","value":"true","effect":"NoSchedule"}],"nodeSelector":{"agentpool":"streamtest"}}}' \
  --restart=Never -n default

# 2) 배포 시각(요청 시각) 기록
date -u +"%Y-%m-%dT%H:%M:%S.%3NZ"

# 3) Pod가 Ready 상태가 될 때까지 대기하며 소요 시간 측정
time kubectl wait --for=condition=Ready pod/streaming-test -n default --timeout=600s

# 4) 이벤트에서 Pulling/Pulled/Started 타임스탬프 추출 (핵심 측정치)
kubectl get events -n default --field-selector involvedObject.name=streaming-test \
  --sort-by=.lastTimestamp \
  -o custom-columns=REASON:.reason,TIME:.lastTimestamp,MESSAGE:.message

# 5) 측정 후 정리
kubectl delete pod streaming-test -n default
```

### 2-3. 기록해 둘 베이스라인 지표
| 지표 | 확인 방법 |
|---|---|
| 이미지 pull 소요 시간 | `Pulling` ~ `Pulled` 이벤트 시각 차 |
| Pod Ready까지 총 시간 | Pod 생성 요청 ~ `kubectl wait --for=condition=Ready` 완료 시각 |
| 이미지 크기 | `az acr repository show --name $ACR_NAME --image $REPOSITORY:$TAG --query "imageSize" -o tsv` (바이트, MB 환산 필요) |
| 노드 조건 | 신규 노드풀 여부, VM 사이즈, 이미지 캐시 유무를 함께 기록 |

동일 이미지·동일 조건으로 **최소 3회 반복 측정 후 평균**을 베이스라인으로 삼는 것을 권장한다 (네트워크 변동 때문에 1회 측정은 신뢰도가 낮음).

---

## 3. 구성 단계 (az cli 위주)

### 3-1. ACR 측 Artifact Streaming 활성화
```bash
# 리소스 그룹/ACR이 없다면 생성 (Premium SKU 필수)
az group create --name $RESOURCE_GROUP --location $LOCATION
az acr create --resource-group $RESOURCE_GROUP --name $ACR_NAME --sku Premium

# 이후 명령에서 매번 --registry 지정하지 않도록 기본값 설정
az configure --defaults acr=$ACR_NAME

# 대상 이미지 push/import (이미 있다면 생략)
az acr import --source docker.io/<repository>:<tag> --image $REPOSITORY:$TAG

# 해당 이미지에 대한 스트리밍 아티팩트 생성
az acr artifact-streaming create --image $REPOSITORY:$TAG

# (선택) 레지스트리에 새로 push되는 이미지에 자동으로 스트리밍 아티팩트를 생성하도록 설정
az acr artifact-streaming update --name $ACR_NAME --repository $REPOSITORY --enable-streaming true

# 스트리밍 아티팩트가 정상 생성되었는지 확인
az acr manifest list-referrers --name $REPOSITORY:$TAG --registry $ACR_NAME
```

### 3-2. AKS 노드풀에서 Artifact Streaming 활성화

**신규 노드풀 생성 시 활성화**:
```bash
az aks nodepool add \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $CLUSTER \
  --name $NODEPOOL \
  --enable-artifact-streaming
```

**기존 노드풀에 적용**:
```bash
az aks nodepool update \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $CLUSTER \
  --name $NODEPOOL \
  --enable-artifact-streaming
```

**필요 시 비활성화(롤백)**:
```bash
az aks nodepool update \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $CLUSTER \
  --name $NODEPOOL \
  --disable-artifact-streaming
```

### 3-3. 활성화 상태 확인
```bash
az aks nodepool show \
  --resource-group $RESOURCE_GROUP \
  --cluster-name $CLUSTER \
  --name $NODEPOOL \
  --query artifactStreamingProfile
# 출력의 "enabled" 필드가 true면 정상 활성화
```

---

## 4. 구성 후 배포 시간 측정 (효과 검증)

2번 단계와 **완전히 동일한 이미지/노드 사이즈/측정 절차**로 재측정해야 비교가 유효하다. 이번에는 Artifact Streaming이 활성화된 노드풀에 배포한다.

```bash
# 1) 스트리밍이 활성화된 노드풀로 동일 이미지 배포
kubectl run streaming-test-after --image=$ACR_NAME.azurecr.io/$REPOSITORY:$TAG \
  --overrides='{"spec":{"nodeSelector":{"agentpool":"'$NODEPOOL'"}}}' \
  --restart=Never -n default

# 2) Pod Ready까지 소요 시간 측정 (2-2와 동일한 방식)
time kubectl wait --for=condition=Ready pod/streaming-test-after -n default --timeout=600s

# 3) 이벤트 타임스탬프 비교
kubectl get events -n default --field-selector involvedObject.name=streaming-test-after \
  --sort-by=.lastTimestamp \
  -o custom-columns=REASON:.reason,TIME:.lastTimestamp,MESSAGE:.message

# 4) 측정 후 정리
kubectl delete pod streaming-test-after -n default

# (선택) 테스트용 노드풀 정리
az aks nodepool delete -g $RESOURCE_GROUP --cluster-name $CLUSTER --name streamtest --no-wait
```

### 결과 비교표 (측정 후 채워 넣을 템플릿)
| 지표 | 적용 전 (2번) | 적용 후 (4번) | 개선율 |
|---|---|---|---|
| 이미지 pull 소요 시간 | | | |
| Pod Ready까지 총 시간 | | | |

### 검증 시 주의할 점
- **동일 이미지 태그 재사용**: 노드에 이미 캐시된 레이어가 있으면 효과가 왜곡되므로, 적용 후 측정도 **신규 노드(또는 이미지 미보유 노드)** 에서 진행해야 한다.
- **digest pull로 확인하지 말 것**: 배포 YAML/Helm 값이 `image@sha256:...` 형태라면 스트리밍이 적용되지 않으므로 측정 자체가 무의미하다. 반드시 태그 기준으로 배포해 확인한다.
- **효과가 잘 나타나는 조건**: 이미지가 클수록(수백 MB~수 GB), 노드 스케일아웃/대량 Pod 기동 시나리오일수록 개선폭이 크게 나타난다. 이미지가 매우 작다면(수십 MB 이하) 체감 차이가 크지 않을 수 있다.
- **imagePullSecrets 사용 여부 재확인**: AKS-ACR 통합(kubelet managed identity)이 아니라 `imagePullSecrets`로 pull하는 워크로드가 섞여 있으면 해당 워크로드는 자동으로 일반 pull로 폴백되어 측정치에 포함되지 않는다.

---

## 참고 자료
- [Reduce Time to Deployment with Artifact Streaming on AKS (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/aks/artifact-streaming)
- [Overview of Artifact Streaming on AKS (Microsoft Learn)](https://learn.microsoft.com/en-us/azure/aks/artifact-streaming-overview)
- [AKS Artifact Streaming GA 안내](https://aka.ms/aks/artifact-streaming)
- [Azure/AKS GitHub Roadmap Issue #3928](https://github.com/Azure/AKS/issues/3928)

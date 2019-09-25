# 使用Azure DevOps, GitHub，AKS和Istio支撑微服务应用的持续交付

本演示使用GitHub作为代码托管服务，Azure Pipeline作为CI/CD，将一个微服务应用完整部署到Azure Kubernetes Services的k8s集群中，并使用istio完成蓝绿部署，灰度发布和金丝雀发布等发布场景。

## 前期准备

完成此演示需要一下资源：

1. GitHub 账号
2. Azure DevOps 账号
3. Azure 全球版账号以及一个可用的订阅，请确保订阅中有足够的费用额度支撑部署。

需要部署的资源如下：

1. GitHub Repo 2个
2. Azure Pipeline 流水线， CI配置4个，CD配置4个
3. Azure Kubernetes Service 集群1个（至少2个节点)

### 准备1 - 部署Azure Kubernetes Services

环境要求：
- Kubernetes 1.11 and above, with RBAC enabled
- Helm 2.12.2 or above
- Azure CLI 2.0.53 or above

## 环境搭建流程

> 以下操作全部在Windows 10环境中使用Windows PowerShell Scritps环境完成

### 下载并在本地准备istio安装部署文件

使用以下脚本完成istio 1.1.3的安装包下载，并解压在本地磁盘上。后续针对k8s集群的操作如果无特别说明均在这个目录中完成

```powershell
# Specify the Istio version that will be leveraged throughout these instructions
$ISTIO_VERSION="1.1.3"

# Windows
# Use TLS 1.2
[Net.ServicePointManager]::SecurityProtocol = "tls12"
$ProgressPreference = 'SilentlyContinue'; Invoke-WebRequest -URI "https://github.com/istio/istio/releases/download/$ISTIO_VERSION/istio-$ISTIO_VERSION-win.zip" -OutFile "istio-$ISTIO_VERSION.zip"
Expand-Archive -Path "istio-$ISTIO_VERSION.zip" -DestinationPath .
```

### 在本地环境安装istioctl命令行工具

此处假设将istio命令行工具放置在d:\tools\目录之下，您可以选择自己的目录。

```powershell
# Copy istioctl.exe to C:\Istio
cd istio-$ISTIO_VERSION
New-Item -ItemType Directory -Force -Path "d:\tools\Istio"
Copy-Item -Path .\bin\istioctl.exe -Destination "d:\tools\Istio"

# Add C:\Istio to PATH.
# Make the new PATH permanently available for the current User, and also immediately available in the current shell.
```

### 创建Azure Kubernetes Services集群

创建服务账号 Service Principle

```shell
az ad sp create-for-rbac --skip-assignment
```

记录服务账号内容（以下仅为示例）

```json
{
  "appId": "xxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "displayName": "azure-cli-2019-09-25-06-43-43",
  "name": "http://azure-cli-2019-09-25-06-43-43",
  "password": "xxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx",
  "tenant": "xxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
}
```

创建 AKS 集群

```powershell
az aks create `
    --resource-group myResourceGroup `
    --name myAKSCluster `
    --node-count 2 `
    --service-principal <appId> `
    --client-secret <password> `
    --generate-ssh-keys `
    --node-vm-size Standard_DS2_v2 `
    --location eastasia
```

安装Kubectl CLI工具并获取k8s访问密钥

```shell
az aks install-cli
## 获取k8s访问密钥
az aks get-credentials --resource-group myResourceGroup --name myAKSCluster
## 启用k8s仪表盘
kubectl create clusterrolebinding kubernetes-dashboard --clusterrole=cluster-admin --serviceaccount=kube-system:kubernetes-dashboard
az aks browse --resource-group myResourceGroup --name myAKSCluster
## 获取k8s节点列表，确保这里可以显示2个节点的信息
kubectl get nodes
```

### 在AKS集群安装istio所需要的CRD

> 注意以下操作需要在istio解压目录中运行，因为需要引用此目录中的资源文件

安装并启用helm和tiller

```powershell
cd .\istio-1.3.3

## 创建系统服务账号Tiller
kubectl create serviceaccount --namespace kube-system tiller
kubectl create clusterrolebinding tiller-cluster-rule --clusterrole=cluster-admin --serviceaccount=kube-system:tiller
## 初始化helm使用tiller服务账号
helm init --service-account tiller

## 检查Kube-system命名空间中已经成功部署了tiller服务端
kubectl get pods -n kube-system
```

使用helm chart完成istio的部署

```powershell
helm install install/kubernetes/helm/istio-init --name istio-init --namespace istio-system
kubectl get jobs -n istio-system
# check this result = 53
(kubectl get crds | Select-String -Pattern 'istio.io').Count
```

### 在AKS集群上设置istio扩展组件Grafana和Kiali

本目录中已经包含了设置Grafana和Kiali所需要的资源文件

```powershell
# 设置istio扩展组件Grafana和Kiali所需要的密钥文件
# Grafana --usernmae=grafana --password=P2ssw0rd@123
kubectl apply -f ../02-azuredevops-aks-istio/grafana-secret.yaml

# kiali --usernmae=kiali --password=P2ssw0rd@123
kubectl apply -f ../02-azuredevops-aks-istio/kiali-secret.yaml

# 启动istio组件安装部署
helm install install/kubernetes/helm/istio --name istio --namespace istio-system `
  --set global.controlPlaneSecurityEnabled=true `
  --set mixer.adapters.useAdapterCRDs=false `
  --set grafana.enabled=true `
  --set grafana.security.enabled=true `
  --set tracing.enabled=true `
  --set kiali.enabled=true
```

### 验证istio安装正确

```powershell
kubectl get svc --namespace istio-system --output wide
kubectl get pods --namespace istio-system
```

### 访问istio扩展组件

以下命令可以将istio扩展组件的界面映射到本地localhost的不同端口上以便进行访问

```powershell
# Grafana http://localhost:3000
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000

# Prometheus http://localhost:9090
kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=prometheus -o jsonpath='{.items[0].metadata.name}') 9090:9090

# Jaeger http://localhost:16686
kubectl port-forward -n istio-system $(kubectl get pod -n istio-system -l app=jaeger -o jsonpath='{.items[0].metadata.name}') 16686:16686

# Kiali http://localhost:20001/kiali/console/
kubectl port-forward -n istio-system $(kubectl get pod -n istio-system -l app=kiali -o jsonpath='{.items[0].metadata.name}') 20001:20001
```

## 使用AKS和Istio完成灰度金丝雀和功能开关场景

### 部署示例应用的1.0版本并启用istio

```poweshell
git clone https://github.com/Azure-Samples/aks-voting-app.git
cd aks-voting-app/scenarios/intelligent-routing-with-istio
kubectl create namespace voting
kubectl label namespace voting istio-injection=enabled
kubectl apply -f kubernetes/step-1-create-voting-app.yaml --namespace voting
kubectl get pods -n voting
kubectl describe pod voting-app-1-0-xxxxxxxx --namespace voting
```

### 部署Gateway和Virtual Service以便可以访问示例应用

```powershell
kubectl apply -f istio/step-1-create-voting-app-gateway.yaml --namespace voting
kubectl get service istio-ingressgateway --namespace istio-system -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
```

Open <http://gatewayaddress>

### 更新示例应用的Analytic服务到1.1版本

以下脚本将完成analytics服务1.1版本的部署，这个版本将提供带有百分比显示的投票数据统计。

```powershell
kubectl apply -f kubernetes/step-2-update-voting-analytics-to-1.1.yaml --namespace voting
# use gatewayaddress
$INGRESS_IP="52.184.35.191"
(1..5) |% { (Invoke-WebRequest -Uri $INGRESS_IP).Content.Split("`n") | Select-String -Pattern "results" }
```

此时应用处于1.0/1.1随机访问的状态，用户可能随机的访问这两个版本的应用。

### 锁定应用到1.1版本

```powershell
kubectl apply -f istio/step-2-update-and-add-routing-for-all-components.yaml --namespace voting
```

### 部署2.0版本的应用

2.0版本的应用提供了基于mysql的storage服务，此服务使用AKS所提供的数据持久化卷将mysql数据库的数据存放在Azure DISK上。这样可以确保即便pod被迁移或者崩溃的情况下数据仍然有效。

更新文件step-3-update-voting-app-with-new-storage.yaml 添加 'storageClassName: managed-premium' 配置

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: mysql-pv-claim
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: managed-premium
  resources:
    requests:
      storage: 1Gi
```

启动2.0版本的部署，2.0版本包含了全新的app, analytic以及storage服务以便适配mysql的数据存储。但在这次上线中我们使用了一个基于cookie的功能开关来完成用户到导流（灰度/金丝雀发布模式），这样我们可以在线上完成2.0版本的测试，同时又不必暴露新功能给所有用户。

```powershell
kubectl apply -f istio/step-3-add-routing-for-2.0-components.yaml --namespace voting
kubectl apply -f kubernetes/step-3-update-voting-app-with-new-storage.yaml --namespace voting
kubectl get pods --namespace voting

# get the presisted disk
# https://docs.microsoft.com/en-us/azure/aks/concepts-storage
# https://docs.microsoft.com/en-us/azure/aks/azure-disks-dynamic-pv
kubectl get pvc mysql-pv-claim -n voting
az disk list --query '[].id | [?contains(@,`pvc-a339f391-df69-11e9-a8c0-cedeaf948693`)]' -o tsv
```

### 清理voting工作区

```powershell
kubectl delete namespace voting
```

**完成**

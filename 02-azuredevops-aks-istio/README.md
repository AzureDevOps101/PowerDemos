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

```shell
## azure cli 登录
az login

## 设置默认订阅
az account set -s {subscription-id}


```

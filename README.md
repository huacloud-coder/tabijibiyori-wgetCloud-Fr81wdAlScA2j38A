[合集 \- K8S(2\)](https://github.com)1\.深入理解Argo CD工作原理09\-09[2\.Argo CD初体验09\-08](https://github.com/chenyishi/p/18402683)收起**目录**

* [1\. ArgoCD 的架构](#_label0)
* [2\. ArgoCD 的工作原理](#_label1)
* [3\. ArgoCD 自动拉取更新的机制](#_label2)
* [4\. 如何与 GitLab 集成](#_label3)
	+ [1\. 配置 GitLab 项目](#_label3_0)
	+ [2\. 配置 ArgoCD Application](#_label3_1)
	+ [3\. 配置 GitLab Webhook](#_label3_2)
	+ [4\. 验证集成](#_label3_3)
* [5\. 示例代码](#_label4)
 

**正文**


[回到顶部](#_labelTop):[FlowerCloud机场](https://hushicha.org)## 1\. ArgoCD 的架构


ArgoCD 是一个 Kubernetes 原生的持续交付工具，它通过监控 Git 仓库中的应用定义来自动部署应用到 Kubernetes 集群。其核心架构由以下几个关键组件构成：


* **API Server**: ArgoCD 的 API 入口，提供了外部接口以便用户或外部工具与 ArgoCD 进行交互。API Server 同时也是 Web UI 的后台服务。
* **Repository Server**: 负责与 Git 仓库交互。它从仓库中拉取应用定义，并将这些定义转化为 Kubernetes 清单文件。Repository Server 会缓存从 Git 仓库中获取的文件，以加快后续的操作。
* **Controller**: 核心控制器，持续监控 Kubernetes 集群的当前状态与期望状态（定义在 Git 仓库中）之间的差异。Controller 负责将集群的状态与 Git 中的期望状态保持一致。
* **Application Controller**: 负责处理用户定义的 ArgoCD Application 资源。它会检查 Git 仓库中的定义，并确保这些定义与 Kubernetes 集群中的应用状态保持同步。
* **Redis Server**: 用于缓存数据和提升系统性能，尤其在处理大量应用和频繁同步操作时显得尤为重要。
* **Web UI**: 提供了一个友好的图形化界面，用户可以通过 Web UI 查看应用状态、同步状态以及进行手动操作。


![](https://img2024.cnblogs.com/blog/1033233/202409/1033233-20240909080720383-326502961.png)![](https://img2024.cnblogs.com/blog/1033233/202409/1033233-20240909080811001-2042424629.png)


[回到顶部](#_labelTop)## 2\. ArgoCD 的工作原理


ArgoCD 的核心理念是 GitOps，即以 Git 仓库作为单一的真理源，通过自动化的方式将仓库中的应用配置同步到 Kubernetes 集群中。


1. **定义应用**: 用户在 Git 仓库中定义应用的 Kubernetes 资源清单，并将这些清单文件提交到 Git 仓库。
2. **创建 ArgoCD Application**: 在 ArgoCD 中创建一个 `Application` 资源，该资源描述了应用在 Git 仓库中的位置，以及在 Kubernetes 集群中部署的位置。
3. **同步状态监控**: ArgoCD Controller 持续监控 Git 仓库中的配置，并与当前集群状态进行对比。每次检测到 Git 仓库中的应用配置发生变化时，Controller 会自动更新集群中的资源，保持与 Git 仓库的一致性。
4. **自动同步与手动同步**: ArgoCD 支持自动同步和手动同步。自动同步模式下，一旦检测到 Git 仓库有变化，ArgoCD 会自动更新 Kubernetes 集群中的资源。而在手动同步模式下，用户需要手动触发同步操作。
5. **回滚功能**: 如果应用更新导致问题，ArgoCD 提供了回滚功能，用户可以轻松恢复到先前的状态。


[回到顶部](#_labelTop)## 3\. ArgoCD 自动拉取更新的机制


ArgoCD 通过持续监控 Git 仓库的变更来实现自动拉取和更新机制。其背后工作原理如下：


1. **定时轮询**: ArgoCD Controller 会定期轮询指定的 Git 仓库，以检查是否有新的提交。默认情况下，这个轮询周期是每 3 分钟一次。
2. **Webhook 触发**: 为了更快地响应更新，ArgoCD 支持通过 Git 仓库的 Webhook 来触发同步操作。当仓库中发生提交或合并请求时，GitLab、GitHub 等平台可以通过 Webhook 通知 ArgoCD 立即进行同步。
3. **状态对比**: 每次拉取到最新的 Git 仓库状态后，ArgoCD 会与当前集群中的应用状态进行对比。如果发现差异，ArgoCD 会自动执行同步操作，确保集群与 Git 仓库的配置一致。


[回到顶部](#_labelTop)## 4\. 如何与 GitLab 集成


ArgoCD 可以无缝集成 GitLab，通过 Webhook 和 CI/CD 流水线实现自动化部署。以下是集成步骤：


### 1\. 配置 GitLab 项目


在 GitLab 中创建或选择一个项目，确保项目中包含 Kubernetes 资源清单文件，并将这些文件存储在仓库的一个目录中，例如 `manifests/`。


### 2\. 配置 ArgoCD Application


在 ArgoCD 中创建一个 `Application` 资源，指向 GitLab 仓库，并设置好相关的路径和目标集群。例如：






```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: my-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://gitlab.com/your-username/your-repo.git'
    path: 'manifests'
    targetRevision: HEAD
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: my-namespace
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```


 




### 3\. 配置 GitLab Webhook


进入 GitLab 项目的 `Settings -> Webhooks`，添加一个新的 Webhook，URL 填写 ArgoCD 的 Webhook 地址，通常为 `https://argocd.example.com/api/webhook`，选择 `Push events` 或 `Merge request events` 触发器。


### 4\. 验证集成


每当 GitLab 仓库中有新的提交或合并请求时，ArgoCD 会收到 Webhook 通知，并立即触发同步操作。您可以通过 ArgoCD 的 Web UI 查看同步进度和结果。


[回到顶部](#_labelTop)## 5\. 示例代码


以下是一个完整的 ArgoCD Application 配置示例，与 GitLab 集成并自动同步：






```
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: demo-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: 'https://gitlab.com/your-username/demo-app.git'
    path: 'k8s-manifests'
    targetRevision: HEAD
  destination:
    server: 'https://kubernetes.default.svc'
    namespace: demo
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
  webhook:
    gitlab:
      - url: https://argocd.example.com/api/webhook
        secret: your-webhook-secret
```


 




在上述配置中，`syncPolicy` 中的 `automated` 参数启用了自动同步和自动修复功能，这意味着 ArgoCD 会自动拉取 GitLab 中的最新配置，并将其应用到 Kubernetes 集群中。




---


通过上述内容，您应该已经了解了 ArgoCD 的架构、工作原理、自动拉取更新的机制，以及如何将其与 GitLab 集成。ArgoCD 强大的自动化和 GitOps 能力，让您的应用部署更加高效和可靠。


![](https://images.cnblogs.com/cnblogs_com/chenyishi/1348350/o_240408130234_wx.png)

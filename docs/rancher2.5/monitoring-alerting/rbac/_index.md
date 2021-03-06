---
title: 基于角色的访问控制
description: 本文介绍了 RBAC 在 Rancher 监控中起到的作用。
keywords:
  - rancher
  - rancher中文
  - rancher中文文档
  - rancher官网
  - rancher文档
  - Rancher
  - rancher 中文
  - rancher 中文文档
  - rancher cn
  - rancher 2.5
  - 监控和告警
  - RBAC
---

## 概述

本文介绍了 RBAC 在 Rancher 监控中起到的作用。

## 集群管理员

集群管理员有权限进行以下操作：

- 在集群上安装 "rancher-monitoring "App，并在 Chart 部署上进行所有其他相关配置。例如，是否创建了默认的仪表盘，在集群上部署了哪些导出器来收集指标等。
- 通过 Prometheus CRs 在集群中创建/修改/删除 Prometheus 部署。
- 通过 Alertmanager CRs 在集群中创建/修改/删除 Alertmanager 部署。
- 通过在适当的命名空间中创建 ConfigMaps 来坚持新的 Grafana 仪表盘或数据资源。
- 通过`cattle-monitoring-system`命名空间中的密钥，将某些 Prometheus 指标暴露给 HPA 的 K8s 自定义指标 API。

## 默认的 K8s ClusterRole

`rancher-monitoring`chart 安装了以下三个`ClusterRoles`。默认情况下，它们与默认的 K8s ClusterRole 的对应关系如下表所示。

| ClusterRole        | 默认的 K8s ClusterRole |
| ------------------ | ---------------------- |
| `monitoring-admin` | `admin`                |
| `monitoring-edit`  | `edit`                 |
| `monitoring-view`  | `view `                |

这些`ClusterRoles`根据可以执行的行动，提供了对监控 CRD 的不同级别的访问。

| CRDs (monitoring.coreos.com)                                                        | Admin            | Edit             | View             |
| ----------------------------------------------------------------------------------- | ---------------- | ---------------- | ---------------- |
| <ul><li>`prometheuses`</li><li>`alertmanagers`</li></ul>                            | Get, List, Watch | Get, List, Watch | Get, List, Watch |
| <ul><li>`servicemonitors`</li><li>`podmonitors`</li><li>`prometheusrules`</li></ul> | \*               | \*               | Get, List, Watch |

### 拥有 k8s admin/edit 权限的用户

拥有集群管理员、`admin`或`edit`权限的用户有权限执行以下操作：

- 通过 ServiceMonitor 和 PodMonitor CRs 修改 Prometheus 部署的拉取配置。
- 通过 PrometheusRules CRs 修改 Prometheus 部署的告警和记录规则。

### 拥有 k8s view 权限的用户

拥有 k8s`view`或的用户有权限执行以下操作：

- 查看部署在集群内的 Prometheuses 的配置。
- 查看部署在集群中的警报器管理员的配置。
- 通过 ServiceMonitor 和 PodMonitor CRs 查看 Prometheus 部署的拉取配置。
- 通过 PrometheusRules CRs 查看 Prometheus 部署的告警/记录规则。

## 其他角色的权限说明

监控还创建了六个额外的 "角色"，这些角色不是默认分配给用户的，而是在集群内创建的。管理员应使用这些角色为用户提供更精细的访问。

### 额外的监控角色

| 角色                       | 目的                                                                                                                                                                                                                                                  |
| :------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| monitoring-config-admin    | 允许管理员给用户分配角色，使其能够查看/修改 cattle-monitoring-system 命名空间内的 Secrets 和 ConfigMaps。在这个命名空间中修改 Secrets / ConfigMaps 可以让用户改变集群的 Alertmanager 配置、Prometheus 适配器配置、额外的 Grafana 数据源、TLS 密钥等。 |
| monitoring-config-edit     | 允许管理员给用户分配角色，使其能够查看/修改 cattle-monitoring-system 命名空间内的 Secrets 和 ConfigMaps。在这个命名空间中修改 Secrets / ConfigMaps 可以让用户改变集群的 Alertmanager 配置、Prometheus 适配器配置、额外的 Grafana 数据源、TLS 密钥等。 |
| monitoring-config-view     | 允许管理员给用户分配角色，使其能够在 cattle-monitoring-system 命名空间内查看 Secrets 和 ConfigMaps。在这个命名空间中查看密钥/配置图可以让用户观察集群的 Alertmanager 配置、Prometheus 适配器配置、额外的 Grafana 数据源、TLS 密钥等。                 |
| monitoring-dashboard-admin | 允许管理员将角色分配给用户，以便能够编辑/查看 cattle-dashboards 命名空间中的 ConfigMaps。该命名空间中的 ConfigMaps 将对应于持久化到集群上的 Grafana Dashboards。                                                                                      |
| monitoring-dashboard-edit  | 允许管理员将角色分配给用户，以便能够编辑/查看 cattle-dashboards 命名空间中的 ConfigMaps。该命名空间中的 ConfigMaps 将对应于持久化到集群上的 Grafana Dashboards。                                                                                      |
| monitoring-dashboard-view  | 允许管理员将角色分配给用户，以便能够查看 cattle-dashboards 命名空间内的 ConfigMaps。此命名空间中的 ConfigMaps 将对应于持久化到集群上的 Grafana Dashboards。                                                                                           |

### 额外的监控集群角色

| 角色            | 目的                                                                                                                                                                                                           |
| :-------------- | :------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| monitoring-view | 从 Monitoring v2 14.5.100+开始提供对外部 Monitoring UIs 的只读访问，给予用户列出 Prometheus、Alertmanager 和 Grafana 端点的权限，并通过 Rancher 代理向 Prometheus、Grafana 和 Alertmanager UIs 发出 GET 请求。 |

## 具有基于 Rancher 集群管理器权限的用户

Rancher 集群管理器部署的默认角色（即集群所有者、集群成员、项目所有者和项目成员）、默认的 k8s 角色和 rancher-监控图部署的角色之间的关系详见下表。

| Cluster Manager Role | k8s Role      | Monitoring ClusterRole / Role | ClusterRoleBinding or RoleBinding?   |
| :------------------- | :------------ | :---------------------------- | :----------------------------------- |
| cluster-owner        | cluster-admin | N/A                           | ClusterRoleBinding                   |
| cluster-member       | admin         | monitoring-admin              | ClusterRoleBinding                   |
| project-owner        | edit          | monitoring-admin              | RoleBinding within Project namespace |
| project-member       | view          | monitoring-edit               | RoleBinding within Project namespace |

除了这些默认角色外，以下额外的 Rancher 项目角色可以应用于你的集群成员，以提供对监控的额外访问。这些 Rancher 角色将与监控图部署的 ClusterRoles 联系在一起。

<figcaption>非默认的Rancher权限和对应的Kubernetes ClusterRoles</figcaption>

| Cluster Manager Role | Kubernetes ClusterRole                    | Available In Rancher From | Available in Monitoring v2 From |
| -------------------- | ----------------------------------------- | ------------------------- | ------------------------------- |
| View Monitoring\*    | [monitoring-ui-view](#monitoring-ui-view) | 2.4.8+                    | 9.4.204+                        |

\* 绑定到**查看监控**Rancher 角色的用户只有在被提供到外部监控 UI 的链接时才有权限访问这些 UI。为了访问集群资源管理器上的监控窗格以获得这些链接，用户必须是至少一个项目的项目成员。

### 2.5.x 中的差异

Rancher 2.5.x 对项目监控进行了改造，所以在 2.5.x 中，**项目成员或项目所有者**将无法访问 Prometheus 或 Grafana，因为我们只在集群级别创建 Grafana 或 Prometheus。

此外，虽然项目所有者仍然只能添加默认在其项目的命名空间内拉取资源的 ServiceMonitors/PodMonitors，但 PrometheusRules 的范围并不限于单个命名空间/项目。因此，项目所有者在其项目命名空间内创建的任何警报规则或记录规则将适用于整个集群，尽管他们将无法查看、编辑或删除在项目命名空间之外创建的任何规则。

### 分配额外的权限

如果集群管理员想为 rancher-监控图提供的角色之外的用户提供额外的管理和编辑访问权限，下表确定了潜在的影响。

| CRDs (monitoring.coreos.com)                              | 它是否会在命名空间/项目之外造成影响？                                                   | 影响                                                                                                                                                                           |
| --------------------------------------------------------- | --------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `prometheuses`                                            | 是的，此资源可以从整个集群的任何目标中刮取指标（除非操作员本身另有配置）。              | 用户将能够定义新的集群级 Prometheus 部署的配置，应该在集群中创建。                                                                                                             |
| `alertmanagers`                                           | 否                                                                                      | 用户将能够定义应该在集群中创建的新集群级 Alertmanager 部署的配置。注意：如果你只是想让用户配置路由和接收器等设置，你应该只提供对 Alertmanager Config Secret 的访问权限来代替。 |
| <ul><li>`servicemonitors`</li><li>`podmonitors`</li></ul> | 没有，默认情况下没有；这可以通过 Prometheus CR 上的`ignoreNamespaceSelectors`进行配置。 | 用户将能够在他们被赋予此权限的命名空间内，在服务/Pod 暴露的端点上设置 Prometheus 的拉取。                                                                                      |
| `prometheusrules`                                         | 是的，PrometheusRules 的作用范围在集群内。                                              | 用户将能够根据整个集群收集的任何系列在 Prometheus 上定义警报或记录规则。                                                                                                       |

| k8s Resources                                    | Namespace                  | 它是否会在命名空间/项目之外造成影响？                                                    | 影响                                                                                                                                                                                |
| ------------------------------------------------ | -------------------------- | ---------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| <ul><li>`secrets`</li><li>`configmaps`</li></ul> | `cattle-monitoring-system` | 是的，此命名空间中的 Configs 和 Secrets 可能会影响整个监控/告警线路。                    | 用户将能够创建或编辑 Secrets / ConfigMaps，如 Alertmanager Config、Prometheus Adapter Config、TLS secrets、额外的 Grafana datasoruces 等。这对所有集群监控/告警都会产生广泛的影响。 |
| <ul><li>`secrets`</li><li>`configmaps`</li></ul> | `cattle-dashboards`        | 是的，此命名空间中的 Configs 和 Secrets 可以创建仪表盘，对集群级收集的所有指标进行查询。 | 用户将能够创建密钥/ConfigMaps，只坚持新的 Grafana 仪表板。                                                                                                                          |

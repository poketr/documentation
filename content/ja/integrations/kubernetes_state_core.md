---
algolia:
  tags:
  - ksm
  - ksm core
categories:
- cloud
- 構成 & デプロイ
- containers
- orchestration
dependencies:
- https://github.com/DataDog/documentation/blob/master/content/en/integrations/kubernetes_state_core.md
description: Kubernetes クラスターで実行されているワークロードの状態を監視します。Kubernetes オブジェクトのステータスを追跡し、マイクロサービスメトリクスを相互に関連付けます。
doc_link: /integrations/kubernetes_state_core/
further_reading:
- link: https://www.datadoghq.com/blog/engineering/our-journey-taking-kubernetes-state-metrics-to-the-next-level/
  tag: ブログ
  text: Kubernetes のステートメトリクスを次のレベルへ進化させる旅
has_logo: true
integration_id: kube-state-metrics
integration_title: Kubernetes State Metrics Core
is_public: true
kind: インテグレーション
name: kubernetes_state_core
public_title: Datadog-Kubernetes State Metrics Core インテグレーション
short_description: Kubernetes オブジェクトのステータスを追跡し、マイクロサービスメトリクスを相互に関連付けます。
supported_os:
- linux
- mac_os
- windows
title: Kubernetes State Metrics Core
---

## 概要

Kubernetes サービスからメトリクスをリアルタイムに取得すると、以下のことが可能になります。

- Kubernetes の状態を視覚化および監視できます。
- Kubernetes のフェイルオーバーとイベントの通知を受けることができます。

Kubernetes State Metrics Core チェックは [kube-state-metrics バージョン 2+][1] を活用し、レガシーの `kubernetes_state` チェックと比較してパフォーマンスとタグ付けが大幅に改善されています。

レガシーチェックとは対照的に、Kubernetes State Metrics Core チェックでは、クラスターに `kube-state-metrics` をデプロイする必要がなくなりました。

Kubernetes State Metrics Core は、より詳細なメトリクスとタグを提供するため、レガシーの `kubernetes_state` チェックのより良い代替手段になります。詳細については、[主な変更点][2]および[収集されたデータ][3]を参照してください。

## セットアップ

### APM に Datadog Agent を構成する

Kubernetes State Metrics Core チェックは [Datadog Cluster Agent][4] イメージに含まれているため、Kubernetes サーバーに他に何もインストールする必要はありません。

### 要件

- Datadog Cluster Agent v1.12+

### 構成

{{< tabs >}}
{{% tab "Helm" %}}

Helm `values.yaml` で、以下を追加します。

```yaml
datadog:
  # (...)
  kubeStateMetricsCore:
    enabled: true
```

{{% /tab %}}
{{% tab "Operator" %}}

`kubernetes_state_core` のチェックを有効にするには、DatadogAgent リソースの設定 `spec.features.kubeStateMetricsCore.enabled` を `true` に設定する必要があります。

```yaml
kind: DatadogAgent
apiVersion: datadoghq.com/v2alpha1
metadata:
  name: datadog
spec:
  global:
    credentials:
      apiKey: <DATADOG_API_KEY>
  features:
    kubeStateMetricsCore:
      enabled: true
```

注: Datadog Operator v0.7.0 以降が必要です。

{{% /tab %}}
{{< /tabs >}}

## kubernetes_state から kubernetes_state_core への移行

### タグの削除

元の `kubernetes_state` のチェックでは、いくつかのタグが非推奨とフラグが立てられ、新しいタグに置き換えられています。移行経路を決定するために、どのタグがメトリクスで送信されるかを確認します。

`kubernetes_state_core` のチェックでは、非推奨のタグのみが提出されます。`kubernetes_state` から `kubernetes_state_core` に移行する前に、モニターやダッシュボードで公式タグのみが使用されているか確認します。

以下は、非推奨タグとそれに代わる公式タグの対応表です。

| 非推奨タグ        | 公式タグ                |
|-----------------------|-----------------------------|
| cluster_name          | kube_cluster_name           |
| container             | kube_container_name         |
| cronjob               | kube_cronjob                |
| daemonset             | kube_daemon_set             |
| deployment            | kube_deployment             |
| hpa                   | horizontalpodautoscaler     |
| image                 | image_name                  |
| job                   | kube_job                    |
| job_name              | kube_job                    |
| namespace             | kube_namespace              |
| phase                 | pod_phase                   |
| pod                   | pod_name                    |
| replicaset            | kube_replica_set            |
| replicationcontroller | kube_replication_controller |
| statefulset           | kube_stateful_set           |

### 後方の非互換性の変更

Kubernetes State Metrics Core チェックには後方互換性がありません。レガシーの `kubernetes_state` チェックから移行する前に、変更点を注意深くお読みください。

`kubernetes_state.node.by_condition`
: ノード名の粒度を持つ新しいメトリクス。レガシーメトリクスの `kubernetes_state.nodes.by_condition` はこのメトリクスに置き換えるため非推奨とされています。**注:** このメトリクスはレガシーチェックにバックポートされ、両方のメトリクス (このメトリクスおよびこのメトリクスと置き換えられるレガシーメトリクス) が利用できます。

`kubernetes_state.persistentvolume.by_phase`
: 永続ボリューム名の粒度を備えた新しいメトリクス。`kubernetes_state.persistentvolumes.by_phase` を置き換えます。

`kubernetes_state.pod.status_phase`
: メトリクスは、`pod_name` のようにポッドレベルのタグでタグ付けされます。

`kubernetes_state.node.count`
: このメトリクスには、もう `host` というタグは付いていません。このメトリクスは、ノード数を `kernel_version` `os_image` `container_runtime_version` `kubelet_version` によって集計します。

`kubernetes_state.container.waiting` と `kubernetes_state.container.status_report.count.waiting`
: これらのメトリクスは、ポッドが待機していない場合、0 の値を出力しなくなりました。0 以外の値のみを報告します。

`kube_job`
: `kubernetes_state` では、`Job` が `CronJob` をオーナーとしていた場合は `kube_job` タグの値が `CronJob` 名となり、それ以外の場合は `Job` 名となります。`kubernetes_state_core` では、`kube_job` タグの値は常に `Job` 名となり、新たに `kube_cronjob` タグキーが追加されて `CronJob` 名をタグ値として持つようになります。`kubernetes_state_core` に移行する場合、クエリフィルターには新しいタグか `kube_job:foo*` (`foo` は `CronJob` 名) を使用することが推奨されます。

`kubernetes_state.job.succeeded`
: `kubernetes_state` では `kuberenetes.job.succeeded` は `count` 型でした。`kubernetes_state_core` では `gauge` 型です。

### ノードレベルのタグの割り当て

ホストレベルまたはノードレベルのタグは、クラスター中心のメトリクスに表示されなくなりました。クラスター内の実際のノードに関連する `kubernetes_state.node.by_condition` や `kubernetes_state.container.restarts` などのメトリクスのみが、それぞれのホストレベルまたはノードレベルのタグを引き続き継承します。

タグをグローバルに追加する場合は、環境変数 `DD_TAGS` を使用するか、それぞれの Helm または Operator 構成を使用します。インスタンス専用レベルのタグは、カスタムの `kubernetes_state_core.yaml` を Cluster Agent にマウントすることで指定することができます。

{{< tabs >}}

{{% tab "Helm" %}}
```yaml
datadog:
  kubeStateMetricsCore:
    enabled: true
  tags: 
    - "<TAG_KEY>:<TAG_VALUE>"
```
{{% /tab %}}
{{% tab "Operator" %}}
```yaml
kind: DatadogAgent
apiVersion: datadoghq.com/v2alpha1
metadata:
  name: datadog
spec:
  global:
    credentials:
      apiKey: <DATADOG_API_KEY>
    tags:
      - "<TAG_KEY>:<TAG_VALUE>"
  features:
    kubeStateMetricsCore:
      enabled: true
```
{{% /tab %}}
{{< /tabs >}}

 `kubernetes_state.container.memory_limit.total` や `kubernetes_state.node.count` のようなメトリクスは、クラスター内のグループの集計カウントで、ホストレベルやノードレベルのタグは追加されません。

### レガシーチェック

{{< tabs >}}
{{% tab "Helm" %}}

Helm の `values.yaml` で `kubeStateMetricsCore` を有効にすると、レガシーの `kubernetes_state` チェックの自動コンフィギュレーションファイルを無視するように Agent が構成されます。目的は、両方のチェックを同時に実行しないようにすることです。

それでも移行フェーズで両方のチェックを同時に有効にする場合は、`values.yaml` の `ignoreLegacyKSMCheck` フィールドを無効にします。

**注**: `ignoreLegacyKSMCheck` は、Agent がレガシーの `kubernetes_state` チェックの自動構成のみを無視するようにします。カスタムの `kubernetes_state` 構成は手動で削除する必要があります。

Kubernetes State Metrics Core チェックでは、クラスターに `kube-state-metrics` をデプロイする必要がなくなりました。Datadog Helm Chart の一部として `kube-state-metrics` のデプロイを無効にできます。これを行うには、Helm の `values.yaml` に以下を追加します。

```yaml
datadog:
  # (...)
  kubeStateMetricsEnabled: false
```

{{% /tab %}}
{{< /tabs >}}

**重要な注意:** Kubernetes State Metrics Core チェックは、レガシーの `kubernetes_state` チェックに代わるものです。Datadog は、一貫したメトリクスを保証するために、両方のチェックを同時に有効にしないことをお勧めします。

## データ収集

### メトリクス

`kubernetes_state.apiservice.count`
: Kubernetes API サービスの数。

`kubernetes_state.apiservice.condition`
: この API サービスの状態。タグ:`apiservice` `condition` `status`。

`kubernetes_state.configmap.count`
: ConfigMaps の数。タグ:`kube_namespace`.

`kubernetes_state.daemonset.count`
: DaemonSets の数。タグ: `kube_namespace`。

`kubernetes_state.daemonset.scheduled`
: 少なくとも 1 つのデーモンポッドを実行しているノードの数。タグ: `kube_daemon_set` `kube_namespace` (標準ラベルの `env` `service` `version`)。 

`kubernetes_state.daemonset.desired`
: デーモンポッドを実行する必要があるノードの数。タグ: `kube_daemon_set` `kube_namespace` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.daemonset.misscheduled`
: デーモンポッドを実行しているが、実行すべきでないノードの数。タグ: `kube_daemon_set` `kube_namespace` (標準ラベルの `env` `service` `version`)。 

`kubernetes_state.daemonset.ready`
: デーモンポッドを実行する必要があり、実行して準備ができているデーモンポッドが 1 つ以上あるノードの数。タグ: `kube_daemon_set` `kube_namespace` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.daemonset.updated`
: 更新されたデーモンポッドを実行しているノードの総数。タグ: `kube_daemon_set` `kube_namespace` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.daemonset.daemons_unavailable`
: デーモンポッドを実行する必要があり、実行して使用可能になっているデーモンポッドがないノードの数。タグ: `kube_daemon_set` `kube_namespace` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.daemonset.daemons_available`
: デーモンポッドを実行する必要があり、実行して使用可能になっているデーモンポッドが 1 つ以上あるノードの数。タグ: `kube_daemon_set` `kube_namespace` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.deployment.count`
: デプロイメントの数。タグ: `kube_namespace`。

`kubernetes_state.deployment.paused`
: デプロイメントが一時停止され、デプロイメントコントローラーによって処理されないかどうか。タグ: `kube_deployment` `kube_namespace` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.deployment.replicas_desired`
: デプロイに必要なポッドの数。タグ: `kube_deployment` `kube_namespace` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.deployment.rollingupdate.max_unavailable`
: デプロイのローリング更新中に使用できないレプリカの最大数。タグ: `kube_deployment` `kube_namespace` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.deployment.rollingupdate.max_surge`
: デプロイのローリング更新中に、必要なレプリカ数を超えてスケジュールできるレプリカの最大数。タグ: `kube_deployment` `kube_namespace` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.deployment.replicas`
: デプロイメントごとのレプリカの数。タグ: `kube_deployment` `kube_namespace` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.deployment.replicas_available`
: デプロイごとに使用可能なレプリカの数。タグ: `kube_deployment` `kube_namespace` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.deployment.replicas_ready `
: デプロイごとの準備ができたレプリカの数。タグ: `kube_deployment` `kube_namespace` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.deployment.replicas_unavailable`
: デプロイごとに使用できないレプリカの数。タグ: `kube_deployment` `kube_namespace` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.deployment.replicas_updated`
: デプロイごとの更新されたレプリカの数。タグ: `kube_deployment` `kube_namespace` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.deployment.condition`
: デプロイの現在のステータス条件。タグ: `kube_deployment` `kube_namespace` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.endpoint.count`
: エンドポイントの数。タグ:`kube_namespace`。

`kubernetes_state.endpoint.address_available`
: エンドポイントで使用可能なアドレスの数。タグ: `kube_endpoint` `kube_namespace`。

`kubernetes_state.endpoint.address_not_ready`
: エンドポイントで準備ができていないアドレスの数。タグ: `kube_endpoint` `kube_namespace`。

`kubernetes_state.namespace.count`
: ネームスペースの数。タグ: `phase`。

`kubernetes_state.node.count`
: ノードの数。タグ: `kernel_version` `os_image` `container_runtime_version` `kubelet_version`。

`kubernetes_state.node.cpu_allocatable`
: スケジューリングに使用できるノードの割り当て可能な CPU。タグ: `node` `resource` `unit`。

`kubernetes_state.node.memory_allocatable`
: スケジューリングに使用できるノードの割り当て可能なメモリ。タグ: `node` `resource` `unit`。

`kubernetes_state.node.pods_allocatable`
: スケジューリングに使用できるノードの割り当て可能なメモリ。タグ: `node` `resource` `unit`。

`kubernetes_state.node.ephemeral_storage_allocatable`
: スケジューリングに使用できるノードの割り当て可能なエフェメラルストレージ。タグ:`node` `resource` `unit`。

`kubernetes_state.node.network_bandwidth_allocatable`
: スケジューリングに使用できるノードの割り当て可能なネットワーク帯域幅。タグ: `node` `resource` `unit`。

`kubernetes_state.node.cpu_capacity`
: ノードの CPU 容量。タグ: `node` `resource` `unit`。

`kubernetes_state.node.memory_capacity`
: ノードのメモリ容量。タグ: `node` `resource` `unit`。

`kubernetes_state.node.pods_capacity`
: ノードのポッド容量。タグ: `node` `resource` `unit`。

`kubernetes_state.node.ephemeral_storage_capacity`
: ノードのエフェメラルストレージ容量。タグ:`node` `resource` `unit`。

`kubernetes_state.node.network_bandwidth_capacity`
: ノードのネットワーク帯域幅容量。タグ: `node` `resource` `unit`。

`kubernetes_state.node.by_condition`
: クラスターノードの状態。タグ: `condition` `node` `status`。

`kubernetes_state.node.status`
: ノードが新しいポッドをスケジュールできるかどうか。タグ: `node` `status`。

`kubernetes_state.node.age`
: ノード作成からの経過時間 (秒)。"タグ: `node`。

`kubernetes_state.container.terminated`
: コンテナが現在終了状態にあるかどうかを説明します。タグ: `kube_namespace` `pod_name` `kube_container_name` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.container.cpu_limit`
: コンテナによる CPU 制限の値。タグ: `kube_namespace` `pod_name` `kube_container_name` `node` `resource` `unit` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.container.memory_limit`
: コンテナによるメモリ制限の値。タグ: `kube_namespace` `pod_name` `kube_container_name` `node` `resource` `unit` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.container.network_bandwidth_limit`
: コンテナによるネットワーク帯域幅制限の値。タグ: `kube_namespace` `pod_name` `kube_container_name` `node` `resource` `unit` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.container.cpu_requested`
: コンテナによってリクエストされた CPU の値。タグ: `kube_namespace` `pod_name` `kube_container_name` `node` `resource` `unit` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.container.memory_requested`
: コンテナによってリクエストされたメモリの値。タグ: `kube_namespace` `pod_name` `kube_container_name` `node` `resource` `unit` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.container.network_bandwidth_requested`
: コンテナによってリクエストされたネットワーク帯域幅の値。タグ: `kube_namespace` `pod_name` `kube_container_name` `node` `resource` `unit` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.container.ready`
: コンテナの準備チェックが成功したかどうかを説明します。タグ: `kube_namespace` `pod_name` `kube_container_name` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.container.restarts`
: コンテナごとのコンテナー再起動の数。タグ: `kube_namespace` `pod_name` `kube_container_name` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.container.running`
: コンテナが現在実行状態にあるかどうかを説明します。タグ: `kube_namespace` `pod_name` `kube_container_name` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.container.waiting`
: コンテナが現在待機状態にあるかどうかを説明します。タグ: `kube_namespace` `pod_name` `kube_container_name` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.container.status_report.count.waiting`
: コンテナが現在待機状態にある理由を説明します。タグ: `kube_namespace` `pod_name` `kube_container_name` `reason` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.container.status_report.count.terminated`
: コンテナが現在終了状態にある理由を示します。タグ: `kube_namespace` `pod_name` `kube_container_name` `reason` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.container.status_report.count.waiting`
: コンテナが現在待機状態にある理由を説明します。タグ: `kube_namespace` `pod_name` `kube_container_name` `reason` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.container.status_report.count.terminated`
: コンテナが現在終了状態にある理由を示します。タグ: `kube_namespace` `pod_name` `kube_container_name` `reason` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.crd.count`
: カスタムリソース定義の数。

`kubernetes_state.crd.condition`
: このカスタムリソース定義の条件。タグ:`customresourcedefinition` `condition` `status`。

`kubernetes_state.pod.ready`
: ポッドがリクエストを処理する準備ができているかどうかを説明します。 タグ: `node` `kube_namespace` `pod_name` `condition` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.pod.scheduled`
: ポッドのスケジューリングプロセスのステータスについて説明します。タグ: `node` `kube_namespace` `pod_name` `condition` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.pod.volumes.persistentvolumeclaims_readonly`
: 永続ボリュームクレームが読み取り専用でマウントされているかどうかを説明します。 タグ: `node` `kube_namespace` `pod_name` `volume` `persistentvolumeclaim` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.pod.unschedulable`
: ポッドのスケジュール不可能なステータスについて説明します。 タグ: `kube_namespace` `pod_name` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.pod.status_phase`
: ポッドの現在のフェーズ。タグ: `node` `kube_namespace` `pod_name` `pod_phase` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.pod.age`
: ポッドが作成されてからの秒単位の時間。タグ: `node` `kube_namespace` `pod_name` `pod_phase` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.pod.uptime`
: ポッドが Kubelet によってスケジュールされ、確認されてからの秒単位の時間。タグ: `node` `kube_namespace` `pod_name` `pod_phase` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.pod.count`
: ポッドの数。タグ: `node` `kube_namespace` `kube_<owner kind>`。

`kubernetes_state.persistentvolumeclaim.status`
: 永続ボリュームクレームの現在のフェーズ。タグ: `kube_namespace` `persistentvolumeclaim` `phase` `storageclass`。

`kubernetes_state.persistentvolumeclaim.access_mode`
: 永続ボリュームクレームで指定されたアクセスモード。タグ: `kube_namespace` `persistentvolumeclaim` `access_mode` `storageclass`。

`kubernetes_state.persistentvolumeclaim.request_storage`
: 永続ボリュームクレームによってリクエストされたストレージの容量。タグ:`kube_namespace` `persistentvolumeclaim` `storageclass`。

`kubernetes_state.persistentvolume.capacity`
: バイト単位の永続ボリューム容量。タグ: `persistentvolume` `storageclass`。

`kubernetes_state.persistentvolume.by_phase`
: このフェーズは、ボリュームが使用可能か、クレームにバインドされているか、クレームによってリリースされているかを示します。タグ: `persistentvolume` `storageclass` `phase`。

`kubernetes_state.pdb.pods_healthy`
: 正常なポッドの現在の数。タグ: `kube_namespace` `poddisruptionbudget`。

`kubernetes_state.pdb.pods_desired`
: 正常なポッドの最小希望数。タグ: `kube_namespace` `poddisruptionbudget`。

`kubernetes_state.pdb.disruptions_allowed`
: 現在許可されているポッドの中断の数。タグ: `kube_namespace` `poddisruptionbudget`。

`kubernetes_state.pdb.pods_total`
: この停止状態の予算によってカウントされたポッドの総数。タグ: `kube_namespace` `poddisruptionbudget`。

`kubernetes_state.secret.count`
: シークレットの数。タグ:`kube_namespace`

`kubernetes_state.secret.type`
: シークレットに関するタイプ。タグ:`kube_namespace` `secret` `type`。

`kubernetes_state.replicaset.count`
: ReplicaSet の数。タグ: `kube_namespace` `kube_deployment`。

`kubernetes_state.replicaset.replicas_desired`
: ReplicaSet に必要なポッドの数。タグ: `kube_namespace` `kube_replica_set` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.replicaset.fully_labeled_replicas`
: ReplicaSet ごとの完全にラベル付けされたレプリカの数。タグ: `kube_namespace` `kube_replica_set` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.replicaset.replicas_ready`
: ReplicaSet ごとの準備ができているレプリカの数。タグ: `kube_namespace` `kube_replica_set` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.replicaset.replicas`
: ReplicaSet ごとのレプリカの数。タグ: `kube_namespace` `kube_replica_set` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.replicationcontroller.replicas_desired`
: ReplicationController に必要なポッドの数。タグ: `kube_namespace` `kube_replication_controller`。

`kubernetes_state.replicationcontroller.replicas_available`
: ReplicationController ごとの使用可能なレプリカの数。タグ: `kube_namespace` `kube_replication_controller`

`kubernetes_state.replicationcontroller.fully_labeled_replicas`
: ReplicationController ごとの完全にラベル付けされたレプリカの数。タグ: `kube_namespace` `kube_replication_controller`

`kubernetes_state.replicationcontroller.replicas_ready`
: ReplicationController ごとの準備ができたレプリカの数。タグ: `kube_namespace` `kube_replication_controller`

`kubernetes_state.replicationcontroller.replicas`
: ReplicationController ごとのレプリカの数。タグ: `kube_namespace` `kube_replication_controller`

`kubernetes_state.statefulset.count`
: StatefulSet の数。タグ: `kube_namespace`。

`kubernetes_state.statefulset.replicas_desired`
: StatefulSet に必要なポッドの数。タグ: `kube_namespace` `kube_stateful_set` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.statefulset.replicas`
: StatefulSet ごとのレプリカの数。タグ: `kube_namespace` `kube_stateful_set` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.statefulset.replicas_current`
: StatefulSet ごとの現在のレプリカの数。タグ: `kube_namespace` `kube_stateful_set` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.statefulset.replicas_ready`
: StatefulSet ごとの準備ができたレプリカの数。タグ: `kube_namespace` `kube_stateful_set` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.statefulset.replicas_updated`
: StatefulSet ごとの更新されたレプリカの数。タグ: `kube_namespace` `kube_stateful_set` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.hpa.count`
: 水平ポッドオートスケーラーの数。タグ: `kube_namespace`。

`kubernetes_state.hpa.min_replicas`
: オートスケーラーで設定できるポッド数の下限、デフォルトは 1。タグ: `kube_namespace` `horizontalpodautoscaler`。

`kubernetes_state.hpa.max_replicas`
: オートスケーラーが設定できるポッド数の上限。MinReplicas より小さくすることはできません。タグ: `kube_namespace` `horizontalpodautoscaler`。

`kubernetes_state.hpa.condition`
: このオートスケーラーの状態。タグ: `kube_namespace` `horizontalpodautoscaler` `condition` `status`。

`kubernetes_state.hpa.desired_replicas`
: このオートスケーラーによって管理されるポッドのレプリカの望ましい数。タグ: `kube_namespace` `horizontalpodautoscaler`。

`kubernetes_state.hpa.current_replicas`
: このオートスケーラーによって管理されるポッドのレプリカの現在の数。タグ: `kube_namespace` `horizontalpodautoscaler`。

`kubernetes_state.hpa.spec_target_metric`
: このオートスケーラーが望ましいレプリカ数を計算するときに使用するメトリクス仕様。タグ: `kube_namespace` `horizontalpodautoscaler` `metric_name` `metric_target_type`。

`kubernetes_state.hpa.status_target_metric`
: このオートスケーラーが望ましいレプリカ数を計算するときに使用する現在のメトリクスステータス。タグ: `kube_namespace` `horizontalpodautoscaler` `metric_name` `metric_target_type`。

`kubernetes_state.vpa.count`
: 垂直ポッドオートスケーラーの数。タグ: `kube_namespace`。

`kubernetes_state.vpa.lower_bound`
: VerticalPodAutoscaler アップデーターが削除する前にコンテナが使用できる最小リソース。タグ: `kube_namespace` `verticalpodautoscaler` `kube_container_name` `resource` `target_api_version` `target_kind` `target_name` `unit`。

`kubernetes_state.vpa.target`
: VerticalPodAutoscaler がコンテナに推奨するターゲットリソース。タグ: `kube_namespace` `verticalpodautoscaler` `kube_container_name` `resource` `target_api_version` `target_kind` `target_name` `unit`。

`kubernetes_state.vpa.uncapped_target`
: VerticalPodAutoscaler が境界を無視するコンテナに推奨するターゲットリソース。タグ: `kube_namespace` `verticalpodautoscaler` `kube_container_name` `resource` `target_api_version` `target_kind` `target_name` `unit`。

`kubernetes_state.vpa.upperbound`
: VerticalPodAutoscaler アップデーターが削除する前にコンテナが使用できる最大リソース。タグ: `kube_namespace` `verticalpodautoscaler` `kube_container_name` `resource` `target_api_version` `target_kind` `target_name` `unit`。

`kubernetes_state.vpa.update_mode`
: VerticalPodAutoscaler の更新モード。タグ: `kube_namespace` `verticalpodautoscaler` `target_api_version` `target_kind` `target_name` `update_mode`。

`kubernetes_state.vpa.spec_container_minallowed`
: 名前に一致するコンテナに VerticalPodAutoscaler が設定できる最小リソース。タグ: `kube_namespace` `verticalpodautoscaler` `kube_container_name` `resource` `target_api_version` `target_kind` `target_name` `unit`。

`kubernetes_state.vpa.spec_container_maxallowed`
: 名前に一致するコンテナに VerticalPodAutoscaler が設定できる最大リソース。タグ: `kube_namespace` `verticalpodautoscaler` `kube_container_name` `resource` `target_api_version` `target_kind` `target_name` `unit`。

`kubernetes_state.cronjob.count`
: cronjobs の数。タグ:`kube_namespace`。

`kubernetes_state.cronjob.spec_suspend`
: 一時停止フラグは、後続の実行を一時停止するようにコントローラーに指示します。タグ: `kube_namespace` `kube_cronjob` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.cronjob.duration_since_last_schedule`
: cronjob が最後にスケジュールされてからの期間。タグ: `kube_cronjob` `kube_namespace` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.job.count`
: ジョブの数。タグ: `kube_namespace` `kube_cronjob`。

`kubernetes_state.job.failed`
: フェーズ失敗に達したポッドの数。タグ: `kube_job` または `kube_cronjob` `kube_namespace` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.job.succeeded`
: フェーズ成功に達したポッドの数。タグ: `kube_job` または `kube_cronjob` `kube_namespace` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.job.completion.succeeded`
: ジョブの実行が終了しました。タグ: `kube_job` または `kube_cronjob` `kube_namespace` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.job.completion.failed`
: ジョブの実行に失敗しました。タグ: `kube_job` または `kube_cronjob` `kube_namespace` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.job.duration`
: ジョブの開始時刻から完了時刻までの経過時間、またはジョブがまだ実行中の場合は現在時刻。タグ: `kube_job` `kube_namespace` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.resourcequota.<resource>.limit`
: リソースごとのリソース割り当て制限に関する情報。タグ: `kube_namespace` `resourcequota`。

`kubernetes_state.resourcequota.<resource>.used`
: リソースごとのリソースクォータの使用に関する情報。タグ: `kube_namespace` `resourcequota`。

`kubernetes_state.limitrange.cpu.min`
: 制約による CPU 制限範囲の使用に関する情報。タグ: `kube_namespace` `limitrange` `type`。

`kubernetes_state.limitrange.cpu.max`
: 制約による CPU 制限範囲の使用に関する情報。タグ: `kube_namespace` `limitrange` `type`。

`kubernetes_state.limitrange.cpu.default`
: 制約による CPU 制限範囲の使用に関する情報。タグ: `kube_namespace` `limitrange` `type`。

`kubernetes_state.limitrange.cpu.default_request`
: 制約による CPU 制限範囲の使用に関する情報。タグ: `kube_namespace` `limitrange` `type`。

`kubernetes_state.limitrange.cpu.max_limit_request_ratio`
: 制約による CPU 制限範囲の使用に関する情報。タグ: `kube_namespace` `limitrange` `type`。

`kubernetes_state.limitrange.memory.min`
: 制約によるメモリ制限範囲の使用に関する情報。タグ: `kube_namespace` `limitrange` `type`。

`kubernetes_state.limitrange.memory.max`
: 制約によるメモリ制限範囲の使用に関する情報。タグ: `kube_namespace` `limitrange` `type`。

`kubernetes_state.limitrange.memory.default`
: 制約によるメモリ制限範囲の使用に関する情報。タグ: `kube_namespace` `limitrange` `type`。

`kubernetes_state.limitrange.memory.default_request`
: 制約によるメモリ制限範囲の使用に関する情報。タグ: `kube_namespace` `limitrange` `type`。

`kubernetes_state.limitrange.memory.max_limit_request_ratio`
: 制約によるメモリ制限範囲の使用に関する情報。タグ: `kube_namespace` `limitrange` `type`。

`kubernetes_state.service.count`
: サービスの数。タグ: `kube_namespace` `type`。

`kubernetes_state.service.type`
: サービスのタイプ。タグ: `kube_namespace` `kube_service` `type`。

`kubernetes_state.ingress.count`
: Ingress の数。タグ:`kube_namespace`。

`kubernetes_state.ingress.path`
: Ingress パスに関する情報。タグ: `kube_namespace` `kube_ingress_path` `kube_ingress` `kube_service` `kube_service_port` `kube_ingress_host`。

**注:** Kubernetes オブジェクトで [Datadog 標準ラベル][5]を構成して、`env` `service` `version` タグを取得できます。

### イベント

Kubernetes State Metrics Core チェックには、イベントは含まれません。

### サービスチェック

`kubernetes_state.cronjob.complete`
: cronjob の最後のジョブが失敗したかどうか。タグ:`kube_cronjob` `kube_namespace` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.cronjob.on_schedule_check`
: cronjob の次のスケジュールが過去である場合に警告します。タグ: `kube_cronjob` `kube_namespace` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.job.complete`
: ジョブが失敗したかどうか。タグ: `kube_job` または `kube_cronjob` `kube_namespace` (標準ラベルの `env` `service` `version`)。

`kubernetes_state.node.ready`
: ノードの準備ができているかどうか。タグ: `node` `condition` `status`。

`kubernetes_state.node.out_of_disk`
: ノードがディスク容量不足であるかどうか。タグ: `node` `condition` `status`。

`kubernetes_state.node.disk_pressure`
: ノードにディスクプレッシャーがかかっているかどうか。タグ: `node` `condition` `status`。

`kubernetes_state.node.network_unavailable`
: ノードネットワークが利用できないかどうか。タグ: `node` `condition` `status`。

`kubernetes_state.node.memory_pressure`
: ノードネットワークにメモリプレッシャーがかかっているかどうか。タグ: `node` `condition` `status`。

### 検証

Cluster Agent コンテナ内で [Cluster Agent の `status` サブコマンドを実行][6]し、Checks セクションで `kubernetes_state_core` を探します。

## トラブルシューティング

### タイムアウトエラー

デフォルトでは、Kubernetes State Metrics Core チェックは、Kubernetes API サーバーからの応答を 10 秒間待ちます。大規模なクラスターでは、リクエストがタイムアウトし、メトリクスが欠落する可能性があります。

環境変数 `DD_KUBERNETES_APISERVER_CLIENT_TIMEOUT` をデフォルトの 10 秒よりも大きな値に設定することで、これを避けることができます。

{{< tabs >}}
{{% tab "Datadog Operator" %}}
以下の構成で `datadog-agent.yaml` を更新します。

```yaml
apiVersion: datadoghq.com/v2alpha1
kind: DatadogAgent
metadata:
  name: datadog
spec:
  override:
    clusterAgent:
      env:
        - name: DD_KUBERNETES_APISERVER_CLIENT_TIMEOUT
          value: <value_greater_than_10>
```

次に、新しい構成を適用します。

```shell
kubectl apply -n $DD_NAMESPACE -f datadog-agent.yaml
```

{{% /tab %}}
{{% tab "Helm" %}}
以下の構成で `datadog-values.yaml` を更新します。

```yaml
clusterAgent:
  env:
    - name: DD_KUBERNETES_APISERVER_CLIENT_TIMEOUT
      value: <value_greater_than_10>
```

次に、Helm チャートをアップグレードします。

```shell
helm upgrade -f datadog-values.yaml <RELEASE_NAME> datadog/datadog
```
{{% /tab %}}
{{< /tabs >}}

ご不明な点は、[Datadog のサポートチーム][7]までお問い合わせください。

## その他の参考資料

{{< partial name="whats-next/whats-next.html" >}}


[1]: https://kubernetes.io/blog/2021/04/13/kube-state-metrics-v-2-0/
[2]: /ja/integrations/kubernetes_state_core/#migration-from-kubernetes_state-to-kubernetes_state_core
[3]: /ja/integrations/kubernetes_state_core/#data-collected
[4]: /ja/agent/cluster_agent/
[5]: /ja/getting_started/tagging/unified_service_tagging/?tab=kubernetes#configuration-1
[6]: /ja/agent/guide/agent-commands/?tab=clusteragent#agent-status-and-information
[7]: /ja/help/
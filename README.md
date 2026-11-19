# ocp-monitoring

OpenShift Container Platform (OCP) 4.21 以上を対象に、Cluster Monitoring および User Workload Monitoring (UWM) の機能概要と構築手順をまとめたリポジトリです。

対象プラットフォーム:

- **ROSA**（Red Hat OpenShift Service on AWS。Classic / HCP いずれも含む）
- **OCI**（Oracle Cloud Infrastructure）上のセルフマネージドクラスタ

共通する機能説明・手順は [共通編](#共通編) に、環境固有の前提条件や手順はそれぞれ [ROSA編](#rosa編) / [OCI編](#oci編) にまとめています。

> 稼働している Pod（コンポーネント）の詳細な役割・構成図は [ARCHITECTURE.md](ARCHITECTURE.md) を参照してください。

## 目次

- [リポジトリ構成](#リポジトリ構成)
- [共通編](#共通編)
  - [Cluster Monitoring とは](#cluster-monitoring-とは)
  - [User Workload Monitoring (UWM) とは](#user-workload-monitoring-uwm-とは)
  - [永続化ストレージの設定](#永続化ストレージの設定)
  - [StorageClass 名の確認方法](#storageclass-名の確認方法)
  - [マニフェストの適用手順](#マニフェストの適用手順)
  - [設定適用後の PVC 拡張手順（oc patch）](#設定適用後の-pvc-拡張手順oc-patch)
  - [（任意）アプリチームへの権限委譲](#任意アプリチームへの権限委譲)
  - [アプリ監視サンプル（ServiceMonitor / PrometheusRule）](#アプリ監視サンプルservicemonitor--prometheusrule)
- [ROSA編](#rosa編)
- [OCI編](#oci編)
  - [⚠️ 事前確認: ワーカーノードのゾーンラベル](#️-事前確認-ワーカーノードのゾーンラベル)
- [トラブルシューティング](#トラブルシューティング)
- [参考リンク一覧](#参考リンク一覧)

---

## リポジトリ構成

```
ocp-monitoring/
├── README.md
├── ARCHITECTURE.md
├── cluster-monitoring/
│   ├── cluster-monitoring-config.yaml            # Cluster Monitoring 用 ConfigMap
│   └── rosa/
│       └── cluster-monitoring-config_rosa.yaml   # ROSA 向け
└── user-workload-monitoring/
    ├── user-workload-monitoring-config.yaml      # UWM 用 ConfigMap
    ├── rosa/
    │   └── user-workload-monitoring-config_rosa.yaml   # ROSA 向け
    └── sample-app/
        ├── sample-app-deployment.yaml            # サンプルアプリ
        ├── sample-app-service-monitor.yaml       # ServiceMonitor
        └── sample-app-prometheusrule.yaml        # PrometheusRule
```

---

## 共通編

### Cluster Monitoring とは

OCP に標準で組み込まれているモニタリングスタックです。`openshift-monitoring` namespace にデプロイされ、kube-apiserver / etcd / network などの **OpenShift のコアコンポーネント** を監視します。主なコンポーネントは以下の通りです。

| コンポーネント | 役割 |
| --- | --- |
| Prometheus (`prometheus-k8s`) | コアコンポーネントのメトリクス収集 |
| Alertmanager (`alertmanager-main`) | アラート通知の管理 |
| Thanos Querier | Prometheus / UWM Prometheus 横断クエリ |
| kube-state-metrics / node-exporter / openshift-state-metrics | 各種メトリクスエクスポーター |

参考: [Monitoring stack for Red Hat OpenShift 4.21](#参考リンク一覧)

### User Workload Monitoring (UWM) とは

ユーザー（アプリケーションチーム）が作成したプロジェクト（namespace）のメトリクス・アラートを監視するための追加スタックです。`openshift-user-workload-monitoring` namespace にデプロイされ、以下のコンポーネントが有効化されます。

| コンポーネント | 役割 |
| --- | --- |
| Prometheus (`prometheus-user-workload`) | ユーザー定義プロジェクトのメトリクス収集 |
| Thanos Ruler (`thanos-ruler-user-workload`) | ユーザー定義の PrometheusRule（アラート/レコーディングルール）の評価 |
| Alertmanager (`alertmanager-user-workload`) ※任意 | ユーザー定義プロジェクト専用のアラート通知（集中管理する場合は不要） |

デフォルトでは無効化されており、`cluster-monitoring-config` ConfigMap（`cluster-monitoring/cluster-monitoring-config.yaml`）の `enableUserWorkload: true` で有効化します。

> **ポイント:** `enableUserWorkload: true` だけでは、アプリチームが作成した `AlertmanagerConfig` はルーティングに反映されません。アプリチーム側でのアラート通知を機能させるには、`cluster-monitoring-config` 側で `alertmanagerMain.enableUserAlertmanagerConfig: true` も合わせて設定する必要があります（本リポジトリのマニフェストには設定済みです）。

UWM 有効化後の役割分担はおおむね以下の通りです。

| 担当 | 作業内容 | 頻度 |
| --- | --- | --- |
| 基盤チーム（cluster-admin） | `cluster-monitoring-config` の編集による UWM / アラート設定の有効化、（任意で）アプリチームへの権限委譲 | 初回のみ |
| アプリチーム | 自分たちの namespace 内で `ServiceMonitor` / `PrometheusRule` / `AlertmanagerConfig` を追加・変更・削除してメトリクス収集・アラートを自走管理 | 随時（基盤チームへの都度依頼は不要） |

参考: [Configuring user workload monitoring](#参考リンク一覧)（ROSA は別ガイドのため [ROSA編の公式手順リンク](#公式手順リンクrosa) を参照）

### 永続化ストレージの設定

デフォルトでは Prometheus / Alertmanager / Thanos Ruler は **永続化されない（emptyDir）** 状態で動作します。Pod の再起動・ノード障害・クラスタアップグレード時にメトリクス/アラート設定が失われるため、本リポジトリのマニフェストでは `volumeClaimTemplate` による永続化（PVC 払い出し）を設定しています。

- `cluster-monitoring/cluster-monitoring-config.yaml`: Alertmanager (main) / Prometheus (k8s) 用
- `user-workload-monitoring/user-workload-monitoring-config.yaml`: Alertmanager (user-workload) / Prometheus (user-workload) / Thanos Ruler (user-workload) 用

本リポジトリでは、**各環境のデフォルト StorageClass（ブロックボリューム）** を使用する想定です（ROSA: `gp3-csi` 相当 / OCI: `oci-bv` 相当。実際の名前は環境ごとに異なるため、必ず事前に確認してください）。

#### なぜ永続化が必要か

デフォルトでは Prometheus / Alertmanager のデータは Pod 内の一時領域に置かれるため、Pod の再起動・再作成で消えます。永続ボリューム (PV) を設定すると、次の効果が得られます。

- メトリクスとアラートのデータが Pod の再起動・再作成後も残る（データ損失の防止）
- Alertmanager の再起動時に、重複通知の受信やアラートのサイレンス消失を防げる

公式は「実稼働環境では永続ストレージの設定を強く推奨」しており、マルチノードクラスターでは高可用性のため次のコンポーネントで必須としています。

| 種別 | 永続化が必要なコンポーネント |
|---|---|
| Cluster Monitoring | Prometheus, Alertmanager |
| User Workload Monitoring | Prometheus, Alertmanager, Thanos Ruler |

#### サイジングに関する公式の目安

公式ドキュメント（OpenShift 4.21）には、環境規模に依存しない「固定の推奨容量」は記載されていません。
記載されているのは次のデフォルト値・例・注意点です。

| 項目 | Cluster Monitoring | User Workload Monitoring |
|---|---|---|
| Prometheus の既定の保持期間 | 15 日 | 24 時間 |
| PVC 容量の記載例 | Prometheus: 40Gi（拡張例: 100Gi） | Thanos Ruler: 10Gi（拡張例: 20Gi） |
| retention / retentionSize の記載例 | `retention: 24h` / `retentionSize: 10GB` | `retention: 24h` / `retentionSize: 10GB` |
| 容量に関する注意 | 十分な永続ストレージを確保しディスクが一杯にならないようにする | Thanos Ruler の必要容量は、評価するルール数と各ルールが生成するサンプル数による。Prometheus の保持期間を設定すると Thanos Ruler も同じ保持期間を継承する（個別設定が無い場合） |

※ 上記の容量例はあくまで設定方法を示す例であり、推奨値ではありません。

#### 本リポジトリのマニフェストの設定値

公式に推奨値がないため、検証用クラスタでの実測（[モニタリングデータ量の算出方法](#モニタリングデータ量の算出方法)）をもとに算出した以下を設定しています。PVC は拡張のみ可能で縮小できないため、運用中の実測で見直してください。

| コンポーネント | retention | retentionSize | PVC 容量（1レプリカあたり） |
|---|---|---|---|
| Prometheus (k8s) | 15d（公式既定） | 40GB | 50Gi |
| Alertmanager (main) | - | - | 5Gi |
| Prometheus (UWM) | 15d（UWM 既定は 24h から延長） | 40GB | 50Gi |
| Alertmanager (UWM) | - | - | 5Gi |
| Thanos Ruler (UWM) | 15d | - | 10Gi |

- Prometheus / Alertmanager は 2 レプリカ構成のため、PVC は**レプリカごとに**作成されます（Prometheus (k8s) の場合は合計 100Gi）。
- `retention`（日数）と `retentionSize`（容量）は**先に到達した方**で古いデータが削除されます。`retentionSize` は PVC 容量の約 8 割に設定し、満杯を防ぎます。
- 保持されるのは、Prometheus が収集したメトリクスの時系列データです（各レプリカの `/prometheus` に全データを保持）。アラートのサイレンス等は Alertmanager が保持します。

共通の注意点（公式より）:

- PVC は**拡張のみ可能で縮小はできない**。そのため、小さく始めて実測で拡張する運用が安全です。
- PVC 設定の変更時は StatefulSet が再作成され、一時的にサービスが停止します。
- データのコンパクションは 2 時間ごとのため、コンパクション前に PV が満杯になり `retentionSize` を超えることがあります。その間は `KubePersistentVolumeFillingUp` アラートが発生します。`retentionSize` は PVC 容量より余裕を持たせて設定してください。
- 規模別の目安は、公式の「Prometheus データベースのストレージ要件」（ノード数・Pod 数に応じた増加量の試算）を参照してください。
  - Cluster Monitoring: https://docs.redhat.com/ja/documentation/monitoring_stack_for_red_hat_openshift/4.21/html-single/configuring_core_platform_monitoring
  - User Workload Monitoring: https://docs.redhat.com/ja/documentation/monitoring_stack_for_red_hat_openshift/4.21/html-single/configuring_user_workload_monitoring

#### モニタリングデータ量の算出方法

PVC のサイズは、「Prometheus が 1 秒あたり何サンプルを取り込んでいるか」から 1 日あたりのデータ増加量を求め、保持期間を掛けて見積もります。以下のコマンドは、Cluster Monitoring 用（`openshift-monitoring` / `prometheus-k8s-0`）です。UWM の場合は、`-n openshift-user-workload-monitoring` と `prometheus-user-workload-0` に読み替えてください。

##### 1. 現在のサンプル取り込み速度（samples/秒）を取得する

```bash
oc -n openshift-monitoring exec -c prometheus prometheus-k8s-0 -- \
  curl -s -G "http://localhost:9090/api/v1/query" \
  --data-urlencode 'query=sum by (pod) (rate(prometheus_tsdb_head_samples_appended_total[5m]))'
```

出力例（検証用クラスタ。`type` ラベルごとに分かれた結果を `sum by (pod)` で合算したもの）:

```
{"status":"success","data":{"resultType":"vector","result":[
  {"metric":{"pod":"prometheus-k8s-0"},"value":[1795012962.755,"11355.67"]},
  {"metric":{"pod":"prometheus-k8s-1"},"value":[1795012962.755,"11356.96"]}]}}
```

出力の見方:

- `value` の 2 番目の値（文字列）が **1 秒あたりの取り込みサンプル数**です（上記は約 11,356 samples/秒）。1 番目はタイムスタンプです。
- `-0` と `-1` の 2 レプリカは同じターゲットを収集するため、ほぼ同じ値になります。**合算せず、1 レプリカ分として扱います**（PVC はレプリカごとにあるため）。2 つの値が大きく乖離する場合は、片方のスクレイプが失敗していないかを確認してください。
- `sum by (pod)` を付けない場合は、`type="float"`（通常のサンプル）と `type="histogram"`（ネイティブヒストグラム。通常は 0）に分かれて出力されます。`type="float"` の値が実質的な取り込み速度です。
- この値はその時点の瞬間値です。変動を均すには、過去 1 日の平均を取ります。

  ```bash
  oc -n openshift-monitoring exec -c prometheus prometheus-k8s-0 -- \
    curl -s -G "http://localhost:9090/api/v1/query" \
    --data-urlencode 'query=avg_over_time(sum by (pod) (rate(prometheus_tsdb_head_samples_appended_total[5m]))[1d:5m])'
  ```

##### 2. 1 サンプルあたりのディスク使用量（バイト）を実測する（任意）

目安は 1〜2 バイト/サンプルですが、実測もできます。コンパクション（2 時間ごと）後のブロックで計算されるため、稼働から数時間以上経過している必要があります。

```bash
oc -n openshift-monitoring exec -c prometheus prometheus-k8s-0 -- \
  curl -s -G "http://localhost:9090/api/v1/query" \
  --data-urlencode 'query=rate(prometheus_tsdb_compaction_chunk_size_bytes_sum[1d]) / rate(prometheus_tsdb_compaction_chunk_samples_sum[1d])'
```

`value` の 2 番目の値が**バイト/サンプル**です。この値が 1〜2 の範囲なら、下の算出式の「1〜2 バイト」を実測値に置き換えて使います。

##### 3. 実際のディスク使用量を確認する

```bash
# PV の使用量・空き容量（バイト）
oc -n openshift-monitoring exec -c prometheus prometheus-k8s-0 -- \
  curl -s -G "http://localhost:9090/api/v1/query" \
  --data-urlencode 'query=kubelet_volume_stats_used_bytes{namespace="openshift-monitoring",persistentvolumeclaim=~"prometheus-k8s-db-.*"}'

# Pod 内から直接確認する場合
oc -n openshift-monitoring exec -c prometheus prometheus-k8s-0 -- df -h /prometheus
```

- 使用量が PVC 容量に近い場合は枯渇の兆候です（PVC 枯渇時は、Observe 画面で「データポイントがありません」となることがあります）。
- 保持期間（`retention`）または `retentionSize` に達すると古いブロックが削除されるため、**定常状態では使用量の増加は止まります**。稼働開始から日数が浅い場合を除き、実測の増加量よりも、下記の算出式での見積もりを使います。

##### 算出式

| 項目 | 算出式 |
|---|---|
| 1 日のサンプル数 | 取り込み速度（samples/秒）× 86,400 |
| 1 日のデータ増加量 | 1 日のサンプル数 × バイト/サンプル（1〜2 バイト、または実測値） |
| 保持期間分のデータ量 | 1 日のデータ増加量 × `retention` の日数 |
| 必要な PVC 容量 | 保持期間分のデータ量 × 1.3（WAL・head・コンパクション中の一時領域の余裕） |
| `retentionSize` | PVC 容量 × 0.8 程度 |
| `retentionSize` で保持できる日数 | `retentionSize` ÷ 1 日のデータ増加量 |

##### 算出例（上記の出力: 11,356 samples/秒、`retention: 15d`）

| 項目 | 1.5 バイト/サンプル | 2 バイト/サンプル |
|---|---|---|
| 1 日のサンプル数 | 約 9.8 億（11,356 × 86,400） | 約 9.8 億 |
| 1 日のデータ増加量 | 約 1.5 GB | 約 2.0 GB |
| 15 日分 | 約 22 GB | 約 29 GB |
| × 1.3（必要な PVC 容量） | 約 29 GB | 約 38 GB |

いずれも PVC 50Gi（`retentionSize: 40GB`）に収まるため、本リポジトリのマニフェストは Prometheus (k8s) を 50Gi / 40GB としています。2 バイト/サンプルの場合は余裕が小さいので、クラスタの成長（ノード・Pod の増加）に合わせて定期的に再計算してください。

### StorageClass 名の確認方法

マニフェスト中の `<storage-class-name>` は、クラスタに存在する StorageClass 名で**事前に置換**する必要があります。適用前に、必ず以下のコマンドでクラスタの StorageClass を確認してください。

```bash
# StorageClass の一覧を確認（(default) と表示されているものが既定の StorageClass）
oc get storageclass

# デフォルト StorageClass の名前だけを取得する場合
oc get storageclass -o jsonpath='{.items[?(@.metadata.annotations.storageclass\.kubernetes\.io/is-default-class=="true")].metadata.name}{"\n"}'
```

確認した StorageClass 名を、`cluster-monitoring-config.yaml` および `user-workload-monitoring-config.yaml` 内のすべての `<storage-class-name>` に置き換えてください。

```bash
# 例: 確認した StorageClass 名が "example-bv" の場合
sed -i 's/<storage-class-name>/example-bv/g' cluster-monitoring/cluster-monitoring-config.yaml
sed -i 's/<storage-class-name>/example-bv/g' user-workload-monitoring/user-workload-monitoring-config.yaml
```

### マニフェストの適用手順

> 適用には `cluster-admin` ロールが必要です。
> `cluster-monitoring-config` / `user-workload-monitoring-config` ConfigMap は、他の設定（Telemeter 設定やアラートルーティング等）がすでに含まれている場合があります。本リポジトリのマニフェストをそのまま `oc apply` すると **既存の設定を上書き・消失させる** おそれがあるため、ConfigMap が既に存在する場合は、既存設定をエクスポートした上で必要な項目のみ差し込んでください。

1. （OCI の場合のみ）[ワーカーノードのゾーンラベルを事前確認](#️-事前確認-ワーカーノードのゾーンラベル) してください。
2. [StorageClass 名を確認し、マニフェストに反映](#storageclass-名の確認方法) してください。
3. `cluster-monitoring-config` ConfigMap の有無を確認します。

   ```bash
   oc get configmap cluster-monitoring-config -n openshift-monitoring
   ```

   - **存在しない場合**（新規クラスタ等）: 本リポジトリのマニフェストをそのまま適用します。

     ```bash
     oc apply -f cluster-monitoring/cluster-monitoring-config.yaml
     ```

   - **既に存在する場合**: まず現在の設定をファイルへリダイレクトして退避します。

     ```bash
     oc get configmap cluster-monitoring-config -n openshift-monitoring -o yaml > cluster-monitoring-config-current.yaml
     ```

     退避したファイルの `data.config.yaml:` 配下（YAML文字列として埋め込まれたブロック）に、`cluster-monitoring/cluster-monitoring-config.yaml` の `enableUserWorkload` / `alertmanagerMain` / `prometheusK8s` の各項目を **既存の設定を残したまま** 差し込みます（インデントのずれに注意してください）。編集後、退避したファイルを適用します。

     ```bash
     oc apply -f cluster-monitoring-config-current.yaml
     ```

4. `user-workload-monitoring-config` ConfigMap についても同様に確認・適用します。

   ```bash
   oc get configmap user-workload-monitoring-config -n openshift-user-workload-monitoring
   ```

   - **存在しない場合**:

     ```bash
     oc apply -f user-workload-monitoring/user-workload-monitoring-config.yaml
     ```

   - **既に存在する場合**: 同様に現在の設定を退避してから、`alertmanager` / `prometheus` / `thanosRuler` の永続化設定のみを差し込んで適用します。

     ```bash
     oc get configmap user-workload-monitoring-config -n openshift-user-workload-monitoring -o yaml > user-workload-monitoring-config-current.yaml
     # ファイルを編集して必要な項目を差し込んだ後
     oc apply -f user-workload-monitoring-config-current.yaml
     ```

5. 反映状況を確認します。

   ```bash
   # monitoring Cluster Operator が AVAILABLE=True / PROGRESSING=False になることを確認
   oc get co monitoring

   # Pod が Running になっていることを確認
   oc get pod -n openshift-monitoring
   oc get pod -n openshift-user-workload-monitoring

   # PVC が Bound になっていることを確認
   oc get pvc -n openshift-monitoring
   oc get pvc -n openshift-user-workload-monitoring
   ```

### 設定適用後の PVC 拡張手順（oc patch）

既に稼働しているクラスタで、ConfigMap の `volumeClaimTemplate` のサイズを変更した場合は、既存の PVC も `oc patch` で拡張します。ConfigMap を適用するだけでは、既存の PVC のサイズが変わらないことがあるためです。

> **事前確認**
> - PVC は**拡張のみ可能で、縮小はできません**。現在の容量より小さい値を指定すると失敗します。`oc get pvc` で現在の容量を確認してから実行してください。
> - StorageClass が `allowVolumeExpansion: true` であることが必要です（ROSA の `gp3-csi` は対応しています）。
> - 以下は ROSA（`gp3-csi`）の例です。

```bash
# 1. ConfigMap の適用（ファイル保存後）
oc apply -f cluster-monitoring-config.yaml
oc apply -f user-workload-monitoring-config.yaml
```

```bash
# 2. 既存の Prometheus PVC を 50Gi に拡張
oc -n openshift-monitoring patch pvc prometheus-k8s-db-prometheus-k8s-0 -p '{"spec":{"resources":{"requests":{"storage":"50Gi"}}}}'
oc -n openshift-monitoring patch pvc prometheus-k8s-db-prometheus-k8s-1 -p '{"spec":{"resources":{"requests":{"storage":"50Gi"}}}}'
```

```bash
# 3. 既存の Alertmanager PVC を 5Gi に拡張（存在する場合）
oc -n openshift-monitoring patch pvc alertmanager-main-db-alertmanager-main-0 -p '{"spec":{"resources":{"requests":{"storage":"5Gi"}}}}'
oc -n openshift-monitoring patch pvc alertmanager-main-db-alertmanager-main-1 -p '{"spec":{"resources":{"requests":{"storage":"5Gi"}}}}'
```

UWM 側の PVC も拡張する場合は、PVC 名を確認してから、同様に `oc patch` を実行してください（`openshift-user-workload-monitoring` namespace）。

```bash
oc -n openshift-user-workload-monitoring get pvc
```

拡張後の確認:

```bash
# CAPACITY が新しい値になっていること、STATUS が Bound であること
oc -n openshift-monitoring get pvc

# 拡張処理中は Conditions に FileSystemResizePending が表示されることがある
oc -n openshift-monitoring describe pvc prometheus-k8s-db-prometheus-k8s-0

# Pod が Running、ClusterOperator が Degraded でないこと
oc -n openshift-monitoring get pods
oc get co monitoring
```

### （任意）アプリチームへの権限委譲

アプリチームが基盤チームに都度依頼せず `ServiceMonitor` / `PrometheusRule` / `AlertmanagerConfig` を自分たちの namespace 内で管理できるようにする場合は、該当 namespace に対して以下の RoleBinding を付与します（`monitoring-edit` / `alert-routing-edit` クラスタロールを使用）。

```bash
# 例: app-a namespace のユーザ定義プロジェクト監視を app-team-user に委譲する場合
oc policy add-role-to-user monitoring-edit app-team-user -n app-a
oc policy add-role-to-user alert-routing-edit app-team-user -n app-a
```

付与しない場合、アプリチームのリソース作成・変更は引き続き基盤チームへの依頼が必要です。

### アプリ監視サンプル（ServiceMonitor / PrometheusRule）

アプリチームが自分のアプリケーションをモニタリング対象に追加する際のサンプル一式を `user-workload-monitoring/sample-app/` に用意しています。

| ファイル | 内容 |
| --- | --- |
| `sample-app-deployment.yaml` | `/metrics` エンドポイントを公開するサンプルアプリ（Deployment + Service） |
| `sample-app-service-monitor.yaml` | Prometheus に対して「このService からメトリクスを収集して」と登録する `ServiceMonitor` |
| `sample-app-prometheusrule.yaml` | しきい値（5xx エラー率 5%超）を満たした場合にアラートを発火させる `PrometheusRule` |

利用する際は、各ファイルの `<namespace-name>` / `my-project` をアプリの実際の namespace に置き換えてください。`ServiceMonitor` の `selector.matchLabels` / `endpoints.port` / `endpoints.path` は、対象アプリの Service のラベル・ポート名・メトリクスパスと一致させる必要があります。

```bash
# <namespace-name> は対象アプリの namespace に置き換える
oc apply -f user-workload-monitoring/sample-app/sample-app-deployment.yaml -n <namespace-name>
oc apply -f user-workload-monitoring/sample-app/sample-app-service-monitor.yaml -n <namespace-name>
oc apply -f user-workload-monitoring/sample-app/sample-app-prometheusrule.yaml -n <namespace-name>

# ServiceMonitor / PrometheusRule の登録確認
oc get servicemonitor -n <namespace-name>
oc get prometheusrule -n <namespace-name>
oc describe prometheusrule sample-app-alerts -n <namespace-name>
```

Web コンソールの `Observe > Targets` でスクレイピング状況、`Observe > Metrics` でメトリクス取得状況を確認できます。

---

## ROSA編

ROSA（Classic / HCP）はマネージドサービスのため、クラスタの一部コンポーネントは Red Hat SRE によって管理されています。適用前に以下を確認してください。

- 公式ドキュメント上は新規クラスタで UWM が有効化されているとの記載がありますが、念のため `openshift-monitoring` namespace に `cluster-monitoring-config` ConfigMap 自体が存在するかどうかを確認します。**こちらの [共通編・マニフェストの適用手順](#マニフェストの適用手順) の手順3で ConfigMap の有無を確認し、存在しない場合は新規作成、存在する場合は `oc edit`（または既存設定を退避してマージ）で `enableUserWorkload: true` を設定してください。**
- 永続化設定（`volumeClaimTemplate` による PVC）の構成方法は、ROSA（特に HCP）のバージョンによって変更が予定/実施されている場合があるため、適用前に必ず最新の公式手順を確認してください。
- デフォルトの StorageClass はブロックストレージ（AWS EBS CSI、一般的に `gp3-csi`）です。名称は [StorageClass 名の確認方法](#storageclass-名の確認方法) の手順で必ず確認してください。
- カスタム Prometheus インスタンスを別途デプロイしている場合、UWM 有効化前に削除が必要です。

### 手順

共通編の [マニフェストの適用手順](#マニフェストの適用手順) に従って、`oc apply`（または `oc edit`）で `cluster-monitoring-config` / `user-workload-monitoring-config` を適用してください。ROSA 固有の追加手順はありません。

### 公式手順リンク（ROSA）

- [Monitoring (ROSA, 単一ページ版)](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html-single/monitoring/index)
- [Enabling monitoring for user-defined projects (ROSA)](https://docs.redhat.com/ja/documentation/red_hat_openshift_service_on_aws/4/html-single/monitoring/index#enabling-monitoring-for-user-defined-projects_preparing-to-configure-the-monitoring-stack-uwm)
- [Enabling the platform Alertmanager instance for user-defined alert routing (ROSA)](https://docs.redhat.com/ja/documentation/red_hat_openshift_service_on_aws/4/html-single/monitoring/index#enabling-the-platform-alertmanager-instance-for-user-defined-alert-routing_preparing-to-configure-the-monitoring-stack-uwm)
- [Configuring user workload monitoring (永続化設定を含む) (ROSA)](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html-single/monitoring/index#configuring-user-workload-monitoring)
- （任意・権限委譲時）[Granting user permission to configure monitoring for user-defined projects (ROSA)](https://docs.redhat.com/ja/documentation/red_hat_openshift_service_on_aws/4/html-single/monitoring/index#granting-users-permission-to-configure-monitoring-for-user-defined-projects_preparing-to-configure-the-monitoring-stack-uwm)
- （任意・権限委譲時）[Granting user permission to configure alert routing for user-defined projects (ROSA)](https://docs.redhat.com/ja/documentation/red_hat_openshift_service_on_aws/4/html-single/monitoring/index#granting-users-permission-to-configure-alert-routing-for-user-defined-projects_preparing-to-configure-the-monitoring-stack-uwm)
- [About ROSA with HCP](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/about/about-hcp)
- [Using Container Storage Interface (CSI) (ROSA)](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/storage/using-container-storage-interface-csi)
- Red Hat ナレッジベース: "ROSA HCP: Upcoming Changes to Monitoring Stack Configuration"（モニタリングの永続化設定方法が変更される可能性があるため、適用前に要確認。アクセスには Red Hat サブスクリプションが必要）

---

## OCI編

OCI 上にセルフマネージド（IPI / Agent-based Installer 等でインストール）した OCP クラスタを対象とします。

### ⚠️ 事前確認: ワーカーノードのゾーンラベル

**本リポジトリの検証環境では、OCI 上のワーカーノードの一部で `topology.kubernetes.io/zone` ラベルがデフォルトで空になっている事象を確認しました。**

#### 発生した事象

- Cluster Monitoring / UWM の永続化設定（PVC）を適用したところ、`oc get co` で `monitoring` が長時間 `AVAILABLE=False` / `PROGRESSING=True`（待機）のままになる。
- `oc describe node <worker>` で確認すると、一部のワーカーノードのみ `topology.kubernetes.io/zone` および `failure-domain.beta.kubernetes.io/zone` ラベルが **空文字** になっていた（他のワーカーには `AP-TOKYO-1-AD-1` のような値が正しく設定されていた）。
- OCI CSI ドライバ (`blockvolume.csi.oraclecloud.com`) が払い出す PV には、プロビジョニング先ノードの zone 情報を元にした `nodeAffinity`（`topology.kubernetes.io/zone` 一致条件）が設定される。ゾーンラベルが空のノードはこの条件に一致しないため、スケジューラがそのノードに Pod を配置できず、該当 PVC を利用する **ThanosRuler / Alertmanager の Pod がスケジューリングできない** 状態になっていた。

#### 事前確認コマンド

マニフェスト適用（永続化設定）の **前に**、すべてのワーカーノードでゾーンラベルが空でないことを必ず確認してください。

```bash
# 全ワーカーノードのゾーンラベルを一覧表示
oc get nodes -l node-role.kubernetes.io/worker -L topology.kubernetes.io/zone,failure-domain.beta.kubernetes.io/zone
```

`TOPOLOGY.KUBERNETES.IO/ZONE` の列が空（`<none>` や空欄）のノードが存在する場合は、PV の nodeAffinity と一致せず Pod がスケジューリングできなくなるため、**先に解消してから** 永続化設定を適用してください。

#### 対処方法

暫定対処として、ゾーンラベルを手動で設定することでスケジューリング不整合を解消できます（恒久対処としては OCI Cloud Controller Manager 側でのゾーン/フォルトドメイン情報の付与が正しく行われているか確認することを推奨します）。

```bash
# <zone-name> は同一 Availability Domain 上の他ノードのラベル値と合わせる（例: AP-TOKYO-1-AD-1）
oc label node <worker-node-name> \
  topology.kubernetes.io/zone=<zone-name> \
  failure-domain.beta.kubernetes.io/zone=<zone-name> \
  --overwrite
```

ラベル付与後、PVC / Pod のスケジューリング状況と `oc get co monitoring` が `AVAILABLE=True` / `PROGRESSING=False` になることを確認してください。

### 手順

1. 上記の [ゾーンラベル事前確認](#️-事前確認-ワーカーノードのゾーンラベル) を実施する。
2. 共通編の [マニフェストの適用手順](#マニフェストの適用手順) に従って適用する。

### 公式手順リンク（OCI）

OCI 上のセルフマネージドクラスタは、通常の OCP 向けドキュメント（"Monitoring stack for Red Hat OpenShift" ガイド）を参照します。

- [Monitoring stack for Red Hat OpenShift 4.21（トップページ）](https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.21)
- [Configuring user workload monitoring (4.21)](https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.21/html-single/configuring_user_workload_monitoring/index)
- [Configuring core platform monitoring: 永続化ストレージの前提条件 (4.21)](https://docs.redhat.com/ja/documentation/monitoring_stack_for_red_hat_openshift/4.21/html-single/configuring_core_platform_monitoring/index#persistent-storage-prerequisites_storing-and-recording-data)
- [Installing on OCI (OCP 4.21)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/installing_on_oci/index)
- [Oracle Cloud Infrastructure Cloud Controller Manager (GitHub)](https://github.com/oracle/oci-cloud-controller-manager)
- [Oracle Cloud Infrastructure Container Storage Interface (CSI) driver (GitHub)](https://github.com/oracle/oci-cloud-controller-manager/blob/master/docs/csi.md)

---

## トラブルシューティング

| 症状 | 確認コマンド | 想定原因 |
| --- | --- | --- |
| `oc get co monitoring` が `PROGRESSING=True` のまま変化しない | `oc get pod -n openshift-monitoring` / `oc get pod -n openshift-user-workload-monitoring` | Pod が `Pending` のままになっていないか確認 |
| 上記 Pod が `Pending` | `oc describe pod <pod-name> -n <namespace>` | `Events` に `node(s) had volume node affinity conflict` が出ていないか確認（→ [OCI編のゾーンラベル事前確認](#️-事前確認-ワーカーノードのゾーンラベル) 参照） |
| PVC が `Pending` のまま | `oc describe pvc <pvc-name> -n <namespace>` | `storageClassName` の指定ミス（[StorageClass 名の確認方法](#storageclass-名の確認方法) 参照） |
| Observe の Dashboards / Metrics で「データポイントがありません」と表示される | `oc -n openshift-monitoring get pvc` / `oc -n openshift-monitoring exec -c prometheus prometheus-k8s-0 -- df -h /prometheus` | PVC の枯渇（Prometheus の PV が満杯）。PVC を拡張する（→ [PVC の拡張手順](#設定適用後の-pvc-拡張手順oc-patch) 参照） |
| アプリチームの `AlertmanagerConfig` によるルーティングが反映されない | `oc get cm cluster-monitoring-config -n openshift-monitoring -o yaml` | `alertmanagerMain.enableUserAlertmanagerConfig: true` が設定されているか確認 |

---

## 参考リンク一覧

### 共通（Monitoring stack for Red Hat OpenShift）

- [Monitoring stack for Red Hat OpenShift 4.21（トップページ）](https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.21)
- [Configuring user workload monitoring (4.21)](https://docs.redhat.com/en/documentation/monitoring_stack_for_red_hat_openshift/4.21/html-single/configuring_user_workload_monitoring/index)
- [Configuring core platform monitoring: 永続化ストレージの前提条件 (4.21)](https://docs.redhat.com/ja/documentation/monitoring_stack_for_red_hat_openshift/4.21/html-single/configuring_core_platform_monitoring/index#persistent-storage-prerequisites_storing-and-recording-data)
- [Accessing metrics: ターゲット詳細の確認 (4.21)](https://docs.redhat.com/ja/documentation/monitoring_stack_for_red_hat_openshift/4.21/html-single/accessing_metrics/index#getting-detailed-information-about-a-target_accessing-metrics-as-an-administrator)

### ROSA

- [Monitoring (ROSA, 単一ページ版)](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html-single/monitoring/index)
- [Enabling monitoring for user-defined projects (ROSA)](https://docs.redhat.com/ja/documentation/red_hat_openshift_service_on_aws/4/html-single/monitoring/index#enabling-monitoring-for-user-defined-projects_preparing-to-configure-the-monitoring-stack-uwm)
- [Enabling the platform Alertmanager instance for user-defined alert routing (ROSA)](https://docs.redhat.com/ja/documentation/red_hat_openshift_service_on_aws/4/html-single/monitoring/index#enabling-the-platform-alertmanager-instance-for-user-defined-alert-routing_preparing-to-configure-the-monitoring-stack-uwm)
- [Configuring user workload monitoring（永続化設定を含む）(ROSA)](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html-single/monitoring/index#configuring-user-workload-monitoring)
- [Granting user permission to configure monitoring for user-defined projects (ROSA)](https://docs.redhat.com/ja/documentation/red_hat_openshift_service_on_aws/4/html-single/monitoring/index#granting-users-permission-to-configure-monitoring-for-user-defined-projects_preparing-to-configure-the-monitoring-stack-uwm)
- [Granting user permission to configure alert routing for user-defined projects (ROSA)](https://docs.redhat.com/ja/documentation/red_hat_openshift_service_on_aws/4/html-single/monitoring/index#granting-users-permission-to-configure-alert-routing-for-user-defined-projects_preparing-to-configure-the-monitoring-stack-uwm)
- [About ROSA with HCP](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/about/about-hcp)
- [Using Container Storage Interface (CSI) (ROSA)](https://docs.redhat.com/en/documentation/red_hat_openshift_service_on_aws/4/html/storage/using-container-storage-interface-csi)

### OCI

- [Installing on OCI (OCP 4.21)](https://docs.redhat.com/en/documentation/openshift_container_platform/4.21/html/installing_on_oci/index)
- [oracle/oci-cloud-controller-manager (GitHub)](https://github.com/oracle/oci-cloud-controller-manager)

> **Note:** `docs.redhat.com` はドキュメント構成がバージョンごとに変更される場合があります。リンク先が見つからない場合は、該当ガイドのトップページから目次（Table of contents）を辿って該当セクションを探してください。

# Kubernetes によるバッチジョブ基盤と監視の検証報告

EC2 上の kind クラスタ（軽量 Kubernetes）で、現行オンプレのバッチ処理と監視を
Kubernetes でどこまで置き換えられるかを検証した結果をまとめる。

---

## 1. やったこと（全体像）

```
┌─────────────────────────────────────────────────────────────────────┐
│                    EC2 (t3.large) 上の kind クラスタ                   │
│                                                                     │
│  ┌──────────────────────────────────────────┐                      │
│  │         バッチジョブ基盤（Tekton）          │                      │
│  │                                          │                      │
│  │  main → transform → split → transfer     │                      │
│  │                                → cleanup  │                      │
│  │         ↑ CronJob が定時起動               │                      │
│  └──────────────────────────────────────────┘                      │
│                        │ メトリクス・ログ                              │
│                        ▼                                            │
│  ┌──────────────────────────────────────────┐                      │
│  │           監視基盤（PLG スタック）           │                      │
│  │                                          │                      │
│  │  Prometheus ──→ Grafana ←── Loki          │                      │
│  │       │                       ↑           │                      │
│  │  AlertManager              Promtail       │                      │
│  │  (10 アラート + 監視抑止)                   │                      │
│  └──────────────────────────────────────────┘                      │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. バッチジョブ — JP1 ジョブネットの Kubernetes 置き換え

### 2.1 現行（JP1）と Kubernetes の対応

| 現行オンプレ（JP1） | Kubernetes（Tekton） | 備考 |
|-------------------|---------------------|------|
| ジョブネット | **Pipeline** | 複数ジョブを直列・並列に連結 |
| 先行ジョブ待ち | **runAfter** | 前のジョブが成功したら次へ |
| 異常終了ハンドラー | **finally** | 成功/失敗に関わらず後処理 |
| JP1 スケジューラ | **CronJob** | cron 式で定時起動 |
| ジョブの再実行 | **retries** | 自動リトライ（冪等性が前提） |

### 2.2 検証した 5 段 Pipeline

```
batch-main ──→ batch-transform ──→ batch-split ──→ batch-transfer ──→ batch-cleanup
 データ取得     フォーマット変換     ファイル分割     転送（模擬）      中間ファイル削除
                                                                    ↓ 常に実行
                                                              error-handler
```

| ジョブ | 現行オンプレの対応処理 | 検証結果 |
|--------|---------------------|---------|
| batch-main | 実績連動ログ転送処理 | 成功 |
| batch-transform | 加工処理 | 成功 |
| batch-split | 分割処理 | 成功 |
| batch-transfer | ホストデータ伝送処理 | 成功（模擬） |
| batch-cleanup | 削除処理 | 成功（証跡は残す） |

**検証結果:** CronJob から 5 段 Pipeline を起動 → 全タスク順次成功を確認。

### 2.3 CronJob による定期実行

```
CronJob (schedule: 0 11 * * 1-5 = JST 20:00 平日)
  → initContainer: RUN_ID 採番（YYYYMMDD-HHMMSS-UUID8桁）
  → kubectl create: PipelineRun 作成
    → batch-pipeline: 5 タスク順次実行
```

**JP1 スケジューラの代替として CronJob が機能することを確認。**

### 2.4 方式比較（4 パターン検証）

| # | パターン | 方式 | ROSA 対応 | 推奨 |
|---|---------|------|-----------|------|
| P1 | Argo Workflows DAG | Workflow CRD | 要追加導入 | |
| P2 | Native Job + InitContainers | 1 Pod 内で直列 | 標準対応 | |
| P3 | Shell Orchestrator | kubectl wait | 標準対応 | |
| P4 | **Tekton Pipelines** | Pipeline CRD + runAfter | **ROSA 標準搭載** | **推奨** |

**Tekton 推奨の理由:** ROSA に標準搭載されており、Red Hat の公式サポートが付く。追加導入不要。

---

## 3. 監視 — JP1/PFM の Kubernetes 置き換え

### 3.1 監視スタック構成

```
┌───────────────────────────────────────────────────────┐
│                    監視スタック                         │
│                                                       │
│  kube-state-metrics ──→ Prometheus ──→ AlertManager   │
│  node-exporter     ──→    │              │            │
│  cAdvisor          ──→    │         通知（Webhook）     │
│                           ▼                           │
│                        Grafana ←── Loki ←── Promtail  │
│                     (ダッシュボード)  (ログ)   (収集)    │
└───────────────────────────────────────────────────────┘
```

### 3.2 アラートルール（10 ルール、全て Prometheus で認識確認済み）

**バッチ監視（3 ルール）**

| アラート名 | 条件 | 重要度 | 検証 |
|-----------|------|--------|------|
| KubeJobFailed | Job が失敗 | critical | **発火確認済み** |
| KubeJobSlowExecution | 実行時間 > 5 分 | warning | ルール認識済み |
| KubeJobOOMKilled | OOMKill 検出 | critical | ルール認識済み |

![KubeJobFailed アラート詳細](../grafana/alert-batch-kubejobfailed.png)
*KubeJobFailed: Job 失敗を即座に検出。PromQL 式と対処コマンドが定義されている*

**インフラ監視（4 ルール）**

| アラート名 | 条件 | 重要度 |
|-----------|------|--------|
| NodeHighCPU | CPU 使用率 > 95%（25分） | warning |
| NodeHighDisk | ディスク残量 < 5%（25分） | critical |
| NodeLowMemory | メモリ残量 < 15%（10分） | warning |
| NodeNotReady | ノード NotReady（5分） | critical |

![インフラ監視 4 ルール一覧](../grafana/alert-infra-rules.png)
*インフラ監視: 4 ルールが全て Normal 状態で稼働中*

**Kubernetes 監視（3 ルール）**

| アラート名 | 条件 | 重要度 |
|-----------|------|--------|
| PodHighRestartRate | 1h で 5 回以上再起動 | warning |
| PodCrashLoopBackOff | CrashLoopBackOff 5 分継続 | critical |
| CronJobMissedSchedule | スケジュール遅延 > 1h | warning |

![PodHighRestartRate が Firing 状態](../grafana/alert-k8s-podhighrestartrate-firing.png)
*PodHighRestartRate: Grafana Pod の再起動を検知して Firing 状態に。アラートが実際に動作している証拠*

### 3.3 アラート発火テスト

SIMULATE_FAILURE=true で意図的にジョブを失敗させて、KubeJobFailed アラートが **firing** になることを確認。

```
検証手順:
1. SIMULATE_FAILURE=true の Job を作成
2. Job が Failed 状態に → Prometheus が kube_job_status_failed=1 を検出
3. KubeJobFailed アラートが firing に遷移
4. AlertManager が通知（現在はダミー Webhook、ROSA で Slack 等に差し替え）
```

### 3.4 監視抑止（開局/閉局）

現行の JP1 監視抑止に相当する機能を AlertManager の mute_time_intervals で実現。

| 抑止名 | 時間帯（JST） | 曜日 |
|--------|-------------|------|
| weekday-night | 20:00〜翌 08:00 | 月〜金 |
| saturday-night | 18:00〜翌 08:00 | 土 |
| sunday-all-day | 終日 | 日 |

**検証結果:** AlertManager の running config に mute_time_intervals が正しく反映されていることを確認。

### 3.5 ログ収集（Loki）

```
Batch Pod ─(JSON stdout)─→ ノードのログファイル ─(tail)─→ Promtail ─(push)─→ Loki ─(LogQL)─→ Grafana
```

- バッチジョブが JSON 形式でログを stdout に出力
- Promtail（DaemonSet）がログを自動収集
- Loki に格納され、LogQL でクエリ可能

**検証結果:** argo namespace のバッチログが Loki に格納され、API 経由で検索できることを確認。

![Loki でバッチログを検索](../grafana/explore-loki-argo-logs.png)
*Grafana Explore: Loki で argo namespace のバッチログを検索。JSON 形式の run_id, job, level, msg が確認できる*

### 3.6 現行監視ツールの読み替え

| 現行（JP1 系） | Kubernetes | 対応 |
|--------------|------------|------|
| JP1/PFM（プロセス監視） | Pod 状態 / Probe / kube-state-metrics | 設計済み |
| JP1/PFM（性能データ） | Prometheus + node-exporter | **実装済み** |
| JP1/PFM（閾値監視） | PrometheusRule（10 ルール） | **実装済み** |
| JP1/AJS3（スケジューラ） | CronJob | **実装済み** |
| JP1/AJS3（監視抑止） | mute_time_intervals | **実装済み** |
| JP1/AJS3（異常通知） | AlertManager | **実装済み** |
| JP1/IM（ログ監視） | Promtail + Loki + LogQL | **実装済み** |
| SLB ヘルスチェック | Readiness/Liveness Probe | 設計済み |

---

### 3.7 Grafana 統合画面

![Prometheus でジョブ成功数を確認](../grafana/explore-prometheus-job-succeeded-detail.png)
*Grafana Explore: Prometheus で kube_job_status_succeeded を PromQL でクエリ。バッチジョブの成功数をリアルタイムで確認できる*

![データソース一覧](../grafana/datasources.png)
*Grafana Data sources: Alertmanager / Loki / Prometheus が統合されており、1 つの画面でメトリクス・ログ・アラートを横断的に確認できる*

---

## 4. なぜ PLG スタック（Prometheus + Loki + Grafana）を選んだか

| 観点 | PLG（採用） | EFK（Elasticsearch + Fluentd + Kibana） |
|------|------------|----------------------------------------|
| メモリ使用量 | ~500MB（軽い） | ~2GB（重い） |
| Grafana 統合 | メトリクスとログを **同じ画面** で見られる | Kibana は別画面 |
| ROSA との親和性 | OpenShift Logging が Loki ベース | 非標準 |
| 学習コスト | Prometheus を知っていれば低い | Elasticsearch の知識が別途必要 |

バッチログは JSON 形式で構造化済み → 全文検索（EFK の強み）は不要。
ROSA との親和性と軽量さで PLG を採用。

---

## 5. 検証環境と結果サマリ

| 項目 | 内容 |
|------|------|
| 環境 | EC2 t3.large (2vCPU/8GB) 上の kind クラスタ |
| Kubernetes バージョン | v1.32.2 |
| 検証日 | 2026-03-11 |

| 検証項目 | 結果 |
|---------|------|
| 5 段 Pipeline 実行 | 全タスク成功 |
| CronJob からの自動実行 | 成功 |
| Prometheus アラートルール（10 本） | 全ルール認識 |
| KubeJobFailed アラート発火 | 発火確認 |
| 監視抑止（mute_time_intervals） | 設定反映確認 |
| Loki ログ収集・検索 | 動作確認 |

---

## 6. ROSA への移植方針

| 項目 | kind（検証済み） | ROSA（移植後） |
|------|----------------|---------------|
| Tekton | kubectl apply でインストール | ROSA 標準搭載（追加作業なし） |
| Prometheus ストレージ | なし（メモリのみ） | gp3-csi 10Gi |
| Loki ストレージ | ローカル | S3 バックエンド |
| Loki レプリカ | 1 | 3（可用性確保） |
| AlertManager 通知先 | ダミー Webhook | Slack / PagerDuty |
| Grafana 認証 | admin（平文） | Secret 管理 |

**Kustomize の overlays で kind/ROSA を切り替え可能な設計にしてあるため、移植時の変更は最小限。**

---

## 7. 次のステップ

```
Phase 1（完了）: kind + 4 パターン比較検証
    ↓
Phase 2（完了）: 5 段 Pipeline + 監視 10 ルール + 監視抑止 + CronJob 定期実行
    ↓
Phase 3（次）: ROSA 環境で Tekton Pipeline と監視基盤を構築・動作確認
    ↓
Phase 4: JOBHOLD 解除、業務監視、DB / レスポンス / セキュリティ監視を具体化
```

| Phase 3 でやること | 詳細 |
|------------------|------|
| Tekton Pipeline 移植 | ROSA 標準搭載のため apply のみ |
| 監視スタック移植 | Helm values の overlay 切り替え |
| Loki の S3 バックエンド化 | 長期ログ保管 |
| AlertManager の通知先設定 | Slack / PagerDuty 接続 |
| JOBHOLD 解除の実装 | 転送先ホスト仕様の確定後 |

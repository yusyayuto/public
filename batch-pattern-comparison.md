# バッチフローパターン方式比較書

## 目次

1. [はじめに](#1-はじめに)
2. [各パターン詳細](#2-各パターン詳細)
3. [JP1/AJS3 機能マッピング表](#3-jp1ajs3-機能マッピング表)
4. [方式比較マトリクス](#4-方式比較マトリクス)
5. [推奨案と判断基準](#5-推奨案と判断基準)

---

## 1. はじめに

### 本書の目的

JP1/AJS3 のジョブネットで構築された直列バッチフロー（実行・異常終了処理）を Kubernetes / OpenShift に移行するにあたり、候補となる6つの方式を比較し、案件に最適な方式を選定するための判断材料を提供する。

### 検証対象シナリオ

```
ジョブA（バッチ本体）→ ジョブB（ログ加工）→ ジョブC（転送）
                                                   ↓ 失敗時
                                               エラー処理
```

### 前提条件

- 本番環境: ROSA（Red Hat OpenShift on AWS）/ 閉域ネットワーク
- 検証環境: EC2 上の kind クラスタ（Kubernetes v1.32）
- バッチイメージ: batch-main / batch-transform / batch-transfer（全パターン共通）
- ジョブ間のファイル共有: /work ディレクトリ（PVC または emptyDir）

---

## 2. 各パターン詳細

### Pattern 1: Argo Workflows DAG（実装済み・ベースライン）

#### 概要

Argo Workflows の DAG テンプレートでジョブ連鎖を定義する方式。
Kubernetes のカスタムリソース（Workflow CRD）としてパイプラインを管理する。

#### 構成図

![P1: Argo Workflows DAG フロー](diagrams/pattern1-argo-dag.png)

#### 実装ファイル

| ファイル | 内容 |
|----------|------|
| `k8s/argo/workflow.yaml` | DAG パイプライン定義 |
| `k8s/argo/rbac.yaml` | workflowtaskresults 用の権限 |
| `scripts/run-workflow.sh` | 実行スクリプト |

#### メリット

- DAG で複雑な依存関係（並列・合流・条件分岐）を表現できる
- Web UI で実行状況をリアルタイム確認できる
- onExit / retryStrategy / timeout など運用機能が豊富
- volumeClaimTemplate で PVC のライフサイクルを自動管理

#### デメリット

- Argo Workflows のインストール・運用が必要（Helm or kubectl apply）
- CRD が増える（Workflow, WorkflowTemplate, CronWorkflow 等）
- ROSA には Argo Workflows が標準搭載されていない（別途インストールが必要）
- emissary executor はレジストリから ENTRYPOINT を取得するため、クラスタ内にしかないイメージ（kind load 等）では command: の明示指定が必要

#### ROSA 互換性

- **インストール**: Operator Hub に Argo Workflows Operator はあるが、Red Hat 公式サポート外
- **SCC（Security Context Constraints）**: emissary executor が特権を要求する場合がある → restricted SCC で動作確認が必要
- **閉域環境**: Argo のコンテナイメージを事前に ECR に push する必要あり

---

### Pattern 2: Native Job + InitContainers（実装済み）

#### 概要

Kubernetes 標準の Job リソースだけを使い、1つの Pod 内で initContainers → メインコンテナの順にジョブを直列実行する方式。追加ツール不要で最もシンプル。

#### 構成図

![P2: Native Job + InitContainers フロー](diagrams/pattern2-native-job.png)

#### 実装ファイル

| ファイル | 内容 |
|----------|------|
| `k8s/native-job/job-chain-initcontainer.yaml` | InitContainers 方式の Job 定義 |
| `k8s/native-job/cronjob-chain.yaml` | CronJob でスケジュール実行 |
| `scripts/run-native-job.sh` | 実行スクリプト |

#### メリット

- **追加ツール一切不要**（Kubernetes 標準機能のみ）
- YAML 1枚で完結、学習コスト最小
- CronJob と組み合わせてスケジュール実行も可能
- emptyDir で PVC 不要（ボリューム管理の手間なし）
- ROSA/OpenShift でも問題なく動作

#### デメリット

- **個別ジョブの再実行ができない**（Pod 全体をやり直す必要がある）
- 並列実行に対応できない（initContainers は常に直列）
- リソース（CPU/メモリ）は Pod 単位でしか設定できない
- ジョブの実行状況が Pod のステータスでしか分からない（Web UI なし）
- エラーハンドリングが限定的（initContainer が失敗 → Pod 全体が Failed）

#### ROSA 互換性

- **完全互換**: Kubernetes 標準機能のみ使用。追加インストール不要
- **SCC**: restricted SCC で問題なし
- **閉域環境**: バッチイメージを ECR に push するだけで動作

---

### Pattern 3: Shell Orchestrator + kubectl wait（実装済み）

#### 概要

3つの独立した Job を、シェルスクリプト内の `kubectl wait` で順番に実行する方式。
スクリプトが JP1/AJS3 のジョブネット定義の役割を果たす。

#### 構成図

![P3: Shell Orchestrator フロー](diagrams/pattern3-shell-orchestrator.png)

#### 実装ファイル

| ファイル | 内容 |
|----------|------|
| `k8s/shell-orchestrator/pvc.yaml` | 共有 PVC 定義 |
| `k8s/shell-orchestrator/job-main.yaml` | Job A 定義 |
| `k8s/shell-orchestrator/job-transform.yaml` | Job B 定義 |
| `k8s/shell-orchestrator/job-transfer.yaml` | Job C 定義 |
| `scripts/run-shell-orchestrator.sh` | オーケストレーションスクリプト |

#### メリット

- **個別ジョブの再実行が可能**（失敗したジョブだけを手動で create できる）
- シェルスクリプトなので既存の運用スクリプトと親和性が高い
- JP1 から移行する際の思考モデルが近い（スクリプト = ジョブネット）
- 追加ツール不要（kubectl だけで動く）

#### デメリット

- **スクリプトの実行元が止まるとパイプラインも止まる**（スクリプトがオーケストレーター）
- スクリプトの品質がそのまま運用品質になる（テストしにくい）
- 並列実行は可能だがスクリプトが複雑化する
- 宣言的（YAML で状態を定義）ではなく手続き的（スクリプトで手順を記述）
- PVC のライフサイクル管理が手動（作成・削除をスクリプトで行う）

#### ROSA 互換性

- **完全互換**: Kubernetes 標準機能 + シェルスクリプト。追加インストール不要
- **注意点**: スクリプトの実行元（CI/CD サーバーや踏み台）の可用性を確保する必要がある
- **閉域環境**: kubectl が API サーバーに到達できれば動作

---

### Pattern 4: Tekton Pipelines（実装済み）

#### 概要

Tekton Pipelines の Task / Pipeline CRD でジョブ連鎖を定義する方式。
ROSA/OpenShift では「OpenShift Pipelines」として標準搭載されている。

#### 構成図

![P4: Tekton Pipelines フロー](diagrams/pattern4-tekton.png)

#### 実装ファイル

| ファイル | 内容 |
|----------|------|
| `k8s/tekton/install-tekton.sh` | kind 用インストールスクリプト |
| `k8s/tekton/task-batch-main.yaml` | Task A 定義 |
| `k8s/tekton/task-batch-transform.yaml` | Task B 定義 |
| `k8s/tekton/task-batch-transfer.yaml` | Task C 定義 |
| `k8s/tekton/pipeline.yaml` | Pipeline 定義（Task 連鎖 + finally） |
| `k8s/tekton/pipelinerun.yaml` | PipelineRun テンプレート |
| `scripts/run-tekton-pipeline.sh` | 実行スクリプト |

#### メリット

- **ROSA/OpenShift に標準搭載**（OpenShift Pipelines = Tekton）→ Red Hat 公式サポート付き
- Task を再利用できる（Pipeline を組み替えるだけ）
- finally で確実なエラーハンドリング
- Kubernetes ネイティブ（CRD ベース、宣言的管理）
- Tekton Hub でコミュニティ Task が利用可能

#### デメリット

- Tekton のインストールが必要（ROSA 以外の環境では手動インストール）
- Argo Workflows に比べて Web UI が貧弱（Tekton Dashboard は別途インストール）
- CRD が多い（Task, Pipeline, PipelineRun, TaskRun 等）
- 並列実行時の DAG 表現は Argo ほど柔軟ではない
- 学習コストがやや高い（params / workspaces / results の理解が必要）

#### ROSA 互換性

- **最高の互換性**: OpenShift Pipelines Operator で自動インストール・自動更新
- **SCC**: OpenShift Pipelines は restricted SCC で動作するよう設計されている
- **閉域環境**: Operator のミラーリングが必要だが、Red Hat の公式手順がある
- **サポート**: Red Hat サブスクリプションに含まれる

---

### Pattern 5: OpenShift Indexed Job（比較書のみ・実装なし）

#### 概要

Kubernetes 1.21+ の Indexed Job を使い、1つの Job 内で複数のインデックス付きタスクを並列実行する方式。completionMode: Indexed を指定すると、各 Pod に JOB_COMPLETION_INDEX 環境変数が自動付与される。

#### 構成図

```
[Job Controller]
    ↓ Indexed Job
[Pod index=0] ──┐
[Pod index=1] ──┼── 並列実行（各 Pod に異なるインデックスが付く）
[Pod index=2] ──┘
```

#### 実装しない理由

- 今回の移行対象となる JP1 ジョブネットは直列実行のフロー。Indexed Job は複数 Pod を並列起動し、各 Pod に異なるインデックス（JOB_COMPLETION_INDEX）を与えて同じ処理を分散実行する仕組みで、Pod 間の依存順序は指定できない
- batch-main → batch-transform → batch-transfer のような順序制御が必要なフローには不向き
- ユースケースが異なる（大量データの分割並列処理向き）

#### こんな場面で有効

- 1000万件のレコードを10分割して並列処理したい
- 同じ処理を異なるパラメータで一斉実行したい
- MapReduce 的な大量データ処理

---

### Pattern 6: AWS Step Functions + ECS（比較書のみ・実装なし）

#### 概要

AWS Step Functions でワークフローを定義し、ECS Fargate でバッチコンテナを実行する方式。Kubernetes を使わず、AWS マネージドサービスだけで完結する。

#### 構成図

```
[Step Functions]
    ↓ ステートマシン定義（ASL）
[ECS Task: batch-main]
    ↓ 成功時
[ECS Task: batch-transform]
    ↓ 成功時
[ECS Task: batch-transfer]
    ↓ 失敗時
[Catch → SNS 通知]
```

#### 実装しない理由

- Kubernetes/OpenShift を使わない方式のため、kind での検証対象外
- 会社環境が ROSA（OpenShift）指定のため、ECS への移行は選択肢にならない

#### こんな場面で有効

- Kubernetes を使わない AWS ネイティブ環境
- サーバー管理なし（Fargate）で運用負荷を最小化したい
- Step Functions の豊富な制御フロー（Choice, Parallel, Map, Wait）を活用したい
- CloudWatch / EventBridge との統合が容易

---

## 3. JP1/AJS3 機能マッピング表

JP1/AJS3 の主要機能が各パターンでどのように実現されるかを対比する。

| JP1/AJS3 の機能 | Pattern 1: Argo DAG | Pattern 2: InitContainers | Pattern 3: Shell + wait | Pattern 4: Tekton |
|-----------------|--------------------|--------------------------|-----------------------|-------------------|
| **ジョブネット（直列実行）** | DAG dependencies | initContainers の順序 | kubectl wait で制御 | runAfter で制御 |
| **並列実行** | DAG で表現可 | 不可 | スクリプトで可能（複雑化） | Pipeline tasks で可能 |
| **条件分岐** | when 式で可能 | 不可 | スクリプトの if で可能 | when 式で可能 |
| **異常終了ハンドラー** | onExit テンプレート | Pod Failed のみ | trap ERR | finally タスク |
| **リトライ** | retryStrategy で指定 | backoffLimit で指定 | スクリプトでループ | retries で指定 |
| **タイムアウト** | activeDeadlineSeconds | activeDeadlineSeconds | timeout オプション | timeout で指定 |
| **スケジュール実行** | CronWorkflow | CronJob | cron + スクリプト | Tekton Triggers / CronJob |
| **ジョブ個別再実行** | テンプレート単位で可 | 不可（Pod 全体やり直し） | Job 単位で可能 | TaskRun 単位で可能 |
| **実行履歴・ログ** | Argo UI | kubectl logs | kubectl logs | Tekton Dashboard |
| **パラメータ渡し** | arguments.parameters | 環境変数 | 環境変数 | params |
| **ファイル共有** | volumeClaimTemplate | emptyDir | PVC（手動管理） | workspaces |
| **保留（HOLD）** | suspend テンプレート | 不可 | スクリプトで read 待ち | PipelineRun の中断 |
| **カレンダー制御** | CronWorkflow + 外部 | CronJob + 外部 | cron + 外部 | Triggers + 外部 |
| **通知（メール等）** | exit-handler で実装 | 別途仕組みが必要 | スクリプトで実装 | finally で実装 |

---

## 4. 方式比較マトリクス

5段階評価（5=最も優れている、1=最も劣っている）

| 評価項目 | P1: Argo DAG | P2: InitContainers | P3: Shell + wait | P4: Tekton | P5: Indexed Job | P6: Step Functions |
|---------|-------------|-------------------|-----------------|-----------|-----------------|-------------------|
| **導入容易さ** | 2 | 5 | 4 | 3 | 4 | 2 |
| **学習コスト** | 3 | 5 | 4 | 3 | 4 | 3 |
| **柔軟性（複雑なフロー）** | 5 | 1 | 3 | 4 | 1 | 5 |
| **エラー処理** | 5 | 2 | 3 | 4 | 2 | 5 |
| **個別再実行** | 4 | 1 | 5 | 4 | 1 | 4 |
| **スケジュール実行** | 4 | 4 | 3 | 3 | 4 | 5 |
| **監視・可視化** | 5 | 2 | 2 | 3 | 2 | 5 |
| **ROSA 対応** | 2 | 5 | 5 | 5 | 5 | 1 |
| **Red Hat サポート** | 1 | 5 | 5 | 5 | 5 | 1 |
| **本番運用向き** | 4 | 2 | 2 | 4 | 2 | 5 |
| **合計** | **35** | **32** | **36** | **38** | **30** | **36** |

### 評価基準の説明とスコアリング根拠

各項目について、5点・3点・1点の条件を定義し、採点を客観的にする。

| 評価項目 | 5点（優） | 3点（中） | 1点（劣） |
|---------|----------|----------|----------|
| **導入容易さ** | K8s 標準機能のみ。追加インストール不要 | Helm/Operator で追加インストール可能。設定は少ない | 複数の前提条件・複雑な設定が必要 |
| **学習コスト** | YAML 1枚で完結。K8s 基本知識だけで使える | 専用の概念（CRD 等）を3〜5個学ぶ必要がある | 専用の DSL/CLI/概念が多い。習熟に1週間以上 |
| **柔軟性** | 並列・条件分岐・ループ・サブワークフローすべて対応 | 直列+条件分岐に対応。並列は限定的 | 直列のみ。条件分岐・並列は不可 |
| **エラー処理** | 専用のエラーハンドラー+リトライ+タイムアウトを宣言的に設定可 | リトライ+タイムアウトは設定可。エラー通知は別途構築 | エラーは Pod/Job の Failed のみ。通知は自前実装 |
| **個別再実行** | 失敗したタスクだけを即座に再実行可能（GUI/CLI） | 条件付きで再実行可能（部分的な手動操作が必要） | Pod/Job 全体のやり直しが必要 |
| **スケジュール実行** | 組み込みのスケジュール機能+外部トリガー連携 | CronJob と組み合わせで対応可能 | 外部の cron に依存。組み込みスケジュールなし |
| **監視・可視化** | 専用 Web UI でリアルタイム表示。ログ・メトリクス統合 | Web UI あり、ただし機能が限定的 or 別途インストール | kubectl のみ。専用 UI なし |
| **ROSA 対応** | ROSA 標準搭載。互換性保証済み | K8s 標準機能のため問題なし。ただし ROSA 最適化はされていない | 動作するが追加インストールが必要。SCC 互換性の確認が必要 |
| **Red Hat サポート** | Red Hat サブスクリプションに含まれる | K8s 標準機能として間接的にサポートされる | サポート対象外。コミュニティサポートのみ |
| **本番運用向き** | 監査ログ・RBAC・マルチテナント・HA すべて対応 | 基本的な運用機能（リトライ・ログ）はある | PoC 向き。監査・HA は自前実装が必要 |

#### 各パターンの採点根拠

**P1: Argo Workflows DAG** — 合計35点
- 導入容易さ(2): Helm+CRD のインストールが必要。ROSA では Operator Hub だが非公式
- 学習コスト(3): Workflow/Template/DAG 等の専用概念があるが、ドキュメントが充実
- 柔軟性(5): DAG で並列・条件分岐・ループすべて対応
- エラー処理(5): onExit+retryStrategy+activeDeadlineSeconds を宣言的に設定
- 個別再実行(4): テンプレート単位で再実行可。ただし Argo CLI が必要
- スケジュール(4): CronWorkflow で組み込み対応
- 監視(5): Argo UI でリアルタイム表示、ログ統合
- ROSA対応(2): SCC との互換性確認が必要、非標準コンポーネント
- Red Hatサポート(1): サポート対象外
- 本番運用(4): 監査ログ・RBAC 対応。HA は追加構成が必要

**P2: Native Job + InitContainers** — 合計32点
- 導入容易さ(5): K8s 標準機能のみ
- 学習コスト(5): YAML 1枚。initContainers は K8s の基本
- 柔軟性(1): 直列のみ。並列・条件分岐は不可
- エラー処理(2): Pod Failed のみ。通知は自前
- 個別再実行(1): Pod 全体をやり直す必要がある
- スケジュール(4): CronJob で対応可能
- 監視(2): kubectl のみ
- ROSA対応(5): 標準機能、互換性保証
- Red Hatサポート(5): K8s 標準
- 本番運用(2): 監査・HA は自前

**P3: Shell Orchestrator + kubectl wait** — 合計36点
- 導入容易さ(4): kubectl のみ。ただしスクリプトの作成が必要
- 学習コスト(4): シェルスクリプト+kubectl。既存知識で対応可
- 柔軟性(3): スクリプトで条件分岐・並列は可能だが複雑化
- エラー処理(3): trap ERR+リトライはスクリプトで実装。宣言的ではない
- 個別再実行(5): Job 単位で create するだけ。最も簡単
- スケジュール(3): 外部 cron に依存
- 監視(2): kubectl のみ
- ROSA対応(5): 標準機能
- Red Hatサポート(5): K8s 標準
- 本番運用(2): スクリプトの品質=運用品質。テストしにくい

**P4: Tekton Pipelines** — 合計38点
- 導入容易さ(3): Helm インストールが必要。ただし ROSA では標準搭載
- 学習コスト(3): Task/Pipeline/params/workspaces 等の概念が必要
- 柔軟性(4): 直列+並列+条件分岐。ただし DAG はない
- エラー処理(4): finally+retries+timeout。Argo の onExit ほど柔軟ではない
- 個別再実行(4): TaskRun 単位で再実行可
- スケジュール(3): Tekton Triggers+CronJob で対応
- 監視(3): Tekton Dashboard は別途インストール。tkn CLI は充実
- ROSA対応(5): OpenShift Pipelines として標準搭載
- Red Hatサポート(5): サブスクリプションに含まれる
- 本番運用(4): CRD ベースで宣言的。GitOps 対応。RBAC 統合

---

## 5. 推奨案と判断基準

### 結論: Pattern 4（Tekton Pipelines）を推奨

#### 推奨理由

1. **ROSA との親和性が最も高い**
   - OpenShift Pipelines として標準搭載されている
   - Red Hat の公式サポートが受けられる
   - SCC（Security Context Constraints）との互換性が保証されている

2. **JP1 からの移行に必要十分な機能を持つ**
   - 直列実行（runAfter）、パラメータ渡し（params）、エラー処理（finally）
   - 個別ジョブの再実行（TaskRun 単位）
   - ファイル共有（workspaces）

3. **Kubernetes ネイティブな設計**
   - CRD ベースで宣言的に管理できる（GitOps との相性が良い）
   - kubectl で操作できる（特別な CLI が必須ではない）

4. **将来の拡張性**
   - Tekton Triggers で Webhook 連携が可能
   - Tekton Hub のコミュニティ Task を活用できる
   - CI/CD パイプラインとしても利用できる（ジョブ管理と CI/CD を統合）

#### 判断フローチャート

```
Q1: ROSA/OpenShift を使うか？
  ├─ YES → Q2 へ
  └─ NO  → Q3 へ

Q2: Red Hat の公式サポートが必要か？
  ├─ YES → Pattern 4（Tekton）を推奨
  └─ NO  → Q3 へ

Q3: 複雑なフロー（並列・条件分岐）が必要か？
  ├─ YES → Pattern 1（Argo Workflows）を推奨
  └─ NO  → Q4 へ

Q4: ジョブの個別再実行が必要か？
  ├─ YES → Pattern 3（Shell Orchestrator）で十分
  └─ NO  → Pattern 2（Native Job）で十分
```

#### パターン別の推奨シーン

| パターン | 推奨シーン |
|---------|-----------|
| **Pattern 1: Argo DAG** | 複雑なワークフロー（並列・合流・ループ）が多い環境。UI での監視が重要。Argo が既に導入済みの場合 |
| **Pattern 2: InitContainers** | 最もシンプルな直列実行で十分な場合。PoC や一時的なバッチ処理 |
| **Pattern 3: Shell + wait** | 既存のシェルスクリプト運用が確立されている環境。段階的な移行の第一歩 |
| **Pattern 4: Tekton** | ROSA/OpenShift 環境。Red Hat サポートが必要。JP1 からの本格移行 |
| **Pattern 5: Indexed Job** | 大量データの分割並列処理。直列実行のジョブ連鎖には不向き |
| **Pattern 6: Step Functions** | Kubernetes を使わない AWS ネイティブ環境。サーバー管理を完全に排除したい場合 |

### 移行ロードマップ（案）

```
Phase 1（完了）: kind + 4パターン検証（Argo/Native Job/Shell/Tekton）
    ↓
Phase 2: ROSA 環境で Tekton Pipeline を構築・動作確認
    ↓
Phase 3: JP1 ジョブネットを Tekton Pipeline に順次移行
    ↓
Phase 4: Tekton Triggers でスケジュール実行・Webhook 連携を追加
```

---

## 変更履歴

| 日付 | 内容 |
|------|------|
| 2026-03-05 | 初版作成（6パターン比較、Pattern 1〜4 実装済み） |
| 2026-03-06 | スコアリング基準（5点/3点/1点の条件）と各パターンの採点根拠を追加 |
| 2026-03-10 | Phase 1 記述を更新（「kind + Argo Workflows」→「kind + 4パターン検証」） |

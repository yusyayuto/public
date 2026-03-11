# JP1 ジョブネット K8s 移行 検証レポート（エグゼクティブサマリー）

## 結論

**Tekton Pipelines（Pattern 4）を推奨する。** ROSA に標準搭載されており、Red Hat 公式サポートが受けられる。JP1 からの移行に必要な機能（直列実行・エラー処理・個別再実行・スケジュール実行）をすべて備えている。

---

## 背景

金融系案件で稼働中の JP1/AJS3 ジョブネットを、ROSA（Red Hat OpenShift on AWS）上の Kubernetes に移行する。現行システムはオンプレ基盤（JP1/AJS3 / JP1/PFM 等）で構成されており、移行対象の核心は「実績連動ログ転送処理」— 業務実績データの加工→分割→転送→JOBHOLD 解除→後続バッチ起動の一連のフローである（詳細は [現行システムアーキテクチャ](current-system-architecture.md) を参照）。ROSA のコスト制限があるため、個人 AWS 環境の EC2 + kind クラスタで事前検証を実施した。

### 検証したバッチフロー（5 段 Pipeline、2026-03-11 検証済み）

```
batch-main → batch-transform → batch-split → batch-transfer → batch-cleanup
 データ取得     フォーマット変換    ファイル分割     転送（模擬）     中間ファイル削除
                                                                  ↓ 常に実行
                                                            error-handler
```

---

## 検証結果

4つの方式を実装・検証し、全パターンで正常動作を確認した（2026-03-05）。

| # | 方式 | 結果 | ROSA 対応 | Red Hat サポート |
|---|------|------|-----------|-----------------|
| P1 | Argo Workflows DAG | 成功 | 要追加導入 | なし |
| P2 | Native Job + InitContainers | 成功 | 標準対応 | あり（K8s標準） |
| P3 | Shell Orchestrator | 成功 | 標準対応 | あり（K8s標準） |
| **P4** | **Tekton Pipelines** | **成功** | **標準搭載** | **あり（OpenShift Pipelines）** |

追加で以下の検証も完了（2026-03-06〜03-11）:

| 検証項目 | 結果 | 日付 |
|---------|------|------|
| 非root コンテナ + セキュリティ制限 | 成功（OpenShift の SCC 要件を満たす） | 03-06 |
| リソース制限（CPU/メモリ） | 成功（全コンテナに設定済み） | 03-06 |
| NetworkPolicy（通信制限） | 成功（Calico CNI で検証） | 03-06 |
| **5 段 Pipeline（split + cleanup 追加）** | **成功** | **03-11** |
| **CronJob 定期実行** | **成功（CronJob → PipelineRun 自動作成）** | **03-11** |
| **Prometheus アラート 10 ルール** | **成功（バッチ 3 + インフラ 4 + K8s 3）** | **03-11** |
| **アラート発火テスト（KubeJobFailed）** | **成功（firing 確認）** | **03-11** |
| **監視抑止（mute_time_intervals）** | **成功（開局/閉局設定反映）** | **03-11** |
| **Loki ログ収集・検索** | **成功（LogQL クエリ確認）** | **03-11** |

詳細は [検証報告（技術説明用）](k8s-batch-monitoring-overview.md) を参照。

---

## なぜ Tekton を推奨するか

| 判断軸 | Tekton の優位性 |
|--------|----------------|
| **ROSA 親和性** | OpenShift Pipelines として標準搭載。追加インストール不要 |
| **サポート** | Red Hat サブスクリプションに含まれる。障害時にベンダーサポートを受けられる |
| **JP1 機能カバー** | 直列実行（runAfter）、エラー処理（finally）、パラメータ渡し（params）、個別再実行（TaskRun） |
| **運用品質** | CRD ベースで宣言的管理。GitOps との相性が良い |
| **将来拡張** | Tekton Triggers で Webhook 連携、Tekton Hub でコミュニティ Task を活用可能 |

合計スコアでも Tekton が最高（38点/50点満点）。詳細は[方式比較書](batch-pattern-comparison.md)を参照。

---

## 残課題

| # | 課題 | 重要度 | 対応時期 |
|---|------|--------|---------|
| 1 | ROSA 環境での動作確認 | 高 | Phase 2（4月〜） |
| 2 | 閉域ネットワークでのオフラインインストール | 高 | Phase 2 |
| 3 | AlertManager の通知先決定（Slack / Datadog / PagerDuty） | 中 | Phase 2 |
| 4 | バッチイメージの ECR 移行 | 中 | Phase 2 |
| 5 | JP1 ジョブネットの段階的移行計画 | 中 | Phase 3（本番移行時） |

---

## コスト

| 期間 | 環境 | 月額（USD） |
|------|------|------------|
| 3月（現在） | 個人 AWS（EC2 + kind） | ~$56 |
| 4月〜 | ROSA 検証環境 | ~$890 |

Reserved Instances 適用で ROSA 環境は約 20% 削減可能（~$710/月）。
詳細は[コスト試算書](cost-estimation.md)を参照。

---

## 移行ロードマップ

```
Phase 1（完了）: EC2 + kind で 4 パターン比較検証
    ↓
Phase 2（完了）: 5 段 Pipeline + 監視 10 ルール + 監視抑止 + CronJob + Loki ← 今ここ
    ↓
Phase 3（4月〜）: ROSA 環境で Tekton Pipeline と監視基盤を構築・動作確認
    ↓
Phase 3: JP1 ジョブネットを Tekton Pipeline に順次移行
    ↓
Phase 4: Tekton Triggers でスケジュール実行・Webhook 連携を追加
```

---

## 次のアクション

1. **本レポートの承認** → Phase 2 の ROSA 環境構築に着手
2. ROSA クラスタの作成・Tekton Pipeline の動作確認
3. 閉域ネットワーク移植手順の実地検証
4. JP1 ジョブネットの移行対象リストアップと優先順位付け

---

## 関連ドキュメント

| ドキュメント | 内容 |
|-------------|------|
| [設計書](design.md) | AWS 構成・Terraform・K8s 検証構成・セキュリティ設計 |
| [方式比較書](batch-pattern-comparison.md) | 6パターンの詳細比較・JP1 機能マッピング・スコアリング |
| [移植ガイド](migration.md) | 閉域ネットワーク環境への移植手順 |
| [コスト試算書](cost-estimation.md) | 個人環境・ROSA 環境の費用見積もり |
| [監視・ログ設計ガイド](observability-guide.md) | Prometheus / Grafana / Loki の構成・アラート設計 |
| [**検証報告（技術説明用）**](k8s-batch-monitoring-overview.md) | **バッチ+監視の検証結果。JP1 との対応表付き** |

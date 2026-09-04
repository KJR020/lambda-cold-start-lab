# Roadmap

## Overview

AWS Lambdaのコールドスタートを題材として、PythonとGoの実行開始までの処理を再現可能な条件で比較する。
検証コード、Infrastructure as Code（IaC）、生データ、集計結果、グラフ、記事原稿を同じGitリポジトリで管理し、第三者が測定条件と結論を追跡できる状態を目指す。

## Approach Decision

- **Chosen**: TerraformでZIP形式のLambda関数を構築し、Lambda APIへの直接invokeとCloudWatch Logsの`INIT_START`・`REPORT`を使って測定する。
- **Why**: API Gatewayなどの外部要因を除外でき、手元にあるTerraform、AWS CLI、Dockerを使って環境構築から後片付けまで再現できるため。
- **Rejected alternatives**:
  - AWS SAM中心の構成: SAM CLIが未導入であり、ローカル実行の起動時間はAWS管理下の実行環境を直接表さない。
  - Serverless FrameworkまたはAWS CDK: 今回の小規模な比較には追加の言語ランタイムと抽象化が増え、測定条件の説明が複雑になる。
  - コンテナイメージ配布: ZIP配布との違いが新しい変数になるため、初回検証の対象外とする。

## Scope

- **In**:
  - PythonとGoで同じ応答を返すLambda関数
  - 依存関係の量を段階的に変えた代表的なシナリオ
  - 同一条件でのコールドスタートとウォームスタートの測定
  - 測定条件、AWSランタイムの版、成果物サイズ、実行ログの保存
  - p50、p95、p99、最小値、最大値、平均値、標準偏差の集計
  - 集計結果のグラフ化と、一次情報・外部統計を併記した技術記事
- **Out**:
  - API GatewayやVPCを含むエンドツーエンド遅延の比較
  - Provisioned Concurrency、SnapStart、Lambda Managed Instancesの比較
  - コンテナイメージとZIP配布の比較
  - PythonとGo以外の言語
  - 本番トラフィックを使った測定
  - すべてのサーバーレス用途に対する言語選定の一般化

## Constraints

- AWSプロファイルは`kjr020_private`、リージョンは`ap-northeast-1`を候補とし、リソース作成前にアカウントIDを確認する。
- 初回比較は`arm64`、512 MB、ZIP配布、VPCなし、Provisioned Concurrencyなし、SnapStartなし、Lambda直接invokeで条件をそろえる。
- GoはOS-only runtimeの`provided.al2023`、Pythonは測定時点でAWS LambdaがサポートするPython 3.14を使用する。
- Terraform AWS ProviderはPython 3.14を扱える6.21.0以上に固定する。
- Goの成果物は`GOOS=linux GOARCH=arm64 CGO_ENABLED=0`でビルドし、ZIP直下へ実行可能な`bootstrap`を置く。Pythonのネイティブ依存もLinux arm64向けに構築する。
- 生データは追記専用として扱い、加工済みデータとグラフは生データから再生成できるようにする。
- 各測定にはGit commit、日時、リージョン、アーキテクチャ、メモリ、関数バージョン、ランタイム版とruntime version ARN、依存シナリオ、成果物サイズを記録する。
- コールドスタート標本は`INIT_START`を確認できたinvokeだけを採用し、関数エラーやsuppressed initの疑いがある標本は失敗理由とともに分離する。
- CloudWatch Logsの反映遅延を前提にポーリングとページネーションを実装し、同期invokeの末尾4 KBログだけに依存しない。
- AWS上の実験用リソースは測定終了後に削除できるようTerraformで管理する。
- 記事では「Goが速い」と先に結論づけず、測定値の分布、制約、年代を明記した外部データから判断する。

## Boundary Strategy

- **Why this split**: 測定条件、AWS環境、データ処理、記事を分けることで、条件変更による影響範囲とレビュー対象を明確にする。
- **Shared seams to watch**: シナリオID、測定レコードのフィールド名、関数名、Terraform出力、記事から参照する集計ファイルのパスを各仕様で一致させる。

## Specs (dependency order)

- [ ] benchmark-contract -- 比較条件、シナリオ識別子、測定レコード、再現性メタデータの形式を定義する。Dependencies: none
- [ ] lambda-benchmark-environment -- Python/Go関数、ビルド、TerraformによるAWS実験環境を構築する。Dependencies: benchmark-contract
- [ ] measurement-analysis -- コールドスタート生成、invoke、ログ取得、検証、統計集計、グラフ生成を自動化する。Dependencies: benchmark-contract, lambda-benchmark-environment
- [ ] technical-article -- 一次情報、外部統計、実測結果を結び付けた日本語の技術記事を作成する。Dependencies: measurement-analysis

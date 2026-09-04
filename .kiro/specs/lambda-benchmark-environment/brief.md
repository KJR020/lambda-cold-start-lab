# Brief: lambda-benchmark-environment

## Problem

実測値を比較するには、PythonとGoの関数を同じAWS条件で繰り返し構築でき、言語以外の差を説明できる環境が必要になる。

## Current State

GitHubリポジトリは空であり、関数コード、依存関係の固定、ビルド手順、Terraform、IAM、ログ設定が存在しない。

## Desired Outcome

1つの手順でPython/Goの成果物をビルドしてAWSへ配置でき、関数名やARNなどを測定処理へ機械的に渡せる。

## Approach

PythonはAWS管理ランタイム、Goは`provided.al2023`を使い、どちらもZIPで配布する。
TerraformでIAMロール、CloudWatch Logs、複数のLambdaシナリオを管理し、関数名と設定を出力する。

## Scope

- **In**: Python/Go関数、依存関係固定、Linux arm64向けビルド、ZIP生成、Terraform、最小権限IAM、ログ保持期間、出力値
- **Out**: invokeの反復、ログ解析、統計集計、外部統計の調査、記事本文

## Boundary Candidates

- 言語別の関数とビルド成果物
- AWSリソースとIAM
- 測定処理へ渡すTerraform出力

## Out of Boundary

- API Gateway、VPC、Provisioned Concurrency、SnapStart
- コンテナイメージ配布
- 実験用リソース以外のAWS環境

## Upstream / Downstream

- **Upstream**: `benchmark-contract`
- **Downstream**: `measurement-analysis`

## Existing Spec Touchpoints

- **Extends**: なし
- **Adjacent**: `benchmark-contract`が定義するシナリオIDと固定条件を変更しない

## Constraints

初回条件は`ap-northeast-1`、`arm64`、512 MB、VPCなしとする。
AWSへの変更はTerraformで追跡し、`terraform destroy`で実験用リソースを削除できるようにする。

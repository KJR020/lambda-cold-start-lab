# Brief: measurement-analysis

## Problem

手作業のinvokeとログ転記では、コールドスタートの保証、測定順、欠測、メタデータ、統計計算に人為的な違いが入り、記事の根拠として弱くなる。

## Current State

CloudWatch Logsの`INIT_START`と`REPORT`から取得したい値は決まっているが、コールドスタートを生成する手順、データ採取、検証、集計、グラフ作成が自動化されていない。

## Desired Outcome

コマンド1つで指定したシナリオを測定し、検証済みの生データ、集計CSV、記事で利用できるグラフを再現できる。

## Approach

関数設定の測定用nonceを変更して新しいLambdaバージョンを発行し、そのバージョンの初回invokeで`INIT_START`を確認できた場合だけコールドスタート標本として採取する。
続けて同じバージョンをinvokeした値をウォームスタート標本とし、CloudWatch Logsとinvoke結果を突合する。

## Scope

- **In**: 測定順の生成、Lambda更新待機、バージョン発行、直接invoke、ログ取得、JSON Lines保存、スキーマ検証、統計量、CSV、グラフ
- **Out**: Lambda関数とTerraformの所有、外部統計の調査、記事本文の編集

## Boundary Candidates

- AWSに副作用を与える測定ランナー
- 生ログから測定レコードへの変換
- 検証済みデータから統計とグラフへの変換

## Out of Boundary

- 待機時間だけに依存してコールドスタートを推定する方法
- エラー標本を黙って除外する処理
- 同期invokeで返る末尾4 KBのログだけを完全な記録として扱うこと
- 事前に有利な結果だけを選ぶ処理

## Upstream / Downstream

- **Upstream**: `benchmark-contract`、`lambda-benchmark-environment`
- **Downstream**: `technical-article`

## Existing Spec Touchpoints

- **Extends**: なし
- **Adjacent**: AWSリソース定義は`lambda-benchmark-environment`、レコード定義は`benchmark-contract`が所有する

## Constraints

シナリオの実行順を無作為化または交互化し、時間帯やランタイム更新の影響を一方の言語だけが受けにくくする。
初回試行と記事用試行を分け、最終サンプル数を測定結果を見て恣意的に変更しない。
CloudWatch Logsの反映遅延に備えて上限付きポーリングとページネーションを行う。
関数エラーとsuppressed initの疑いがある標本は成功標本から分離し、runtime version ARNを記録する。

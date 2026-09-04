# Brief: benchmark-contract

## Problem

PythonとGoのLambdaコールドスタートを比較するとき、条件や記録項目が途中で変わると、測定結果を再現したり公平に解釈したりできない。

## Current State

比較したい観点と大まかな実験案はあるが、シナリオ識別子、固定条件、生データの形式、欠測や失敗の扱いがファイルとして定義されていない。

## Desired Outcome

実験環境、採取処理、集計処理、記事が同じ定義を参照し、測定レコードだけを見ても条件と来歴を追跡できる。

## Approach

この文書での「ベンチマーク契約」は、比較対象、固定条件、シナリオID、JSON Lines形式の測定レコード、検証規則をまとめた内部仕様を指す。
機械検証できるスキーマと、人が読める実験方針を一緒に管理する。

## Scope

- **In**: シナリオ一覧、固定条件、測定レコード、再現性メタデータ、欠測・失敗レコード、検証規則
- **Out**: Lambda関数の実装、AWSリソース、実際のinvoke、統計計算、記事本文

## Boundary Candidates

- 実験の意味と比較条件
- ファイル間で共有するデータ形式
- 測定結果を採用または除外する規則

## Out of Boundary

- 特定の測定結果を良い・悪いと評価する処理
- AWSアカウントやIAMの具体的な構築
- 記事向けの文章表現

## Upstream / Downstream

- **Upstream**: AWS Lambda公式ドキュメント、検証目的
- **Downstream**: `lambda-benchmark-environment`、`measurement-analysis`、`technical-article`

## Existing Spec Touchpoints

- **Extends**: なし
- **Adjacent**: なし（新規リポジトリ）

## Constraints

言語間で同じ意味を持つフィールド名を使い、生データを後から上書きしない。
測定時点のランタイム版とGit commitを必須にする。

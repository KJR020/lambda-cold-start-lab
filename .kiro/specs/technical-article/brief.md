# Brief: technical-article

## Problem

単純な速度比較だけでは、PythonとGoの実行モデル、Lambdaの初期化処理、採用状況、測定の限界を読者が区別できない。

## Current State

会話上の説明と記事構成案はあるが、一次情報への参照、実測データ、再現手順、外部統計の年代を結び付けた原稿は存在しない。

## Desired Outcome

読者が「何を測ったか」「何が分かったか」「何までは言えないか」を確認でき、リポジトリからデータとグラフを再生成できる日本語記事が完成する。

## Approach

LambdaのライフサイクルとPython/Goの実行準備を一次情報で説明し、その後に実験方法、結果、考察、採用統計、判断基準を示す。
Goの優位性を前提にせず、仮説が支持されなかった結果もそのまま扱う。

## Scope

- **In**: AWS・Python・Goの一次情報、サーバーレスの言語利用統計、実験方法、結果、グラフ、考察、制約、再現手順へのリンク
- **Out**: 新しい実験条件の追加、計測コードの修正、特定言語の全面的な推奨

## Boundary Candidates

- 技術的な仕組みの説明
- 実測結果と不確実性の解釈
- 外部統計と企業事例の年代付き整理

## Out of Boundary

- 出典のない採用率や性能値
- 異なる年代の統計を同じ母集団として合算すること
- 今回の実験条件をすべてのワークロードへ一般化すること

## Upstream / Downstream

- **Upstream**: `measurement-analysis`の検証済み集計、AWS/Python/Go公式情報、公開された利用統計
- **Downstream**: 外部ブログまたは技術媒体への公開

## Existing Spec Touchpoints

- **Extends**: なし
- **Adjacent**: 集計値やグラフは`measurement-analysis`の出力を参照し、記事側で手修正しない

## Constraints

原稿は日本語で作成する。
数値には測定日、条件、サンプル数を併記し、外部統計には公開年と母集団を明記する。

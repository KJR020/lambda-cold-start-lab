# Research & Design Decisions

## Summary

- **Feature**: `benchmark-contract`
- **Discovery Scope**: New Feature
- **Key Findings**:
  - 言語に依存しないJSON Schema Draft 2020-12をデータ契約の正本にすると、Terraform、Python、Go、記事生成処理から同じ定義を参照できる。
  - Pythonの`jsonschema`はDraft 2020-12、スキーマ自体の検証、全違反の列挙に対応している。
  - 測定記録はUTF-8のJSON Linesとして一試行ずつ追記すると、既存記録を上書きせず、行単位で検証できる。
  - Lambdaのランタイム版とARNは、新しい実行環境の作成時に出る`INIT_START`から取得する必要がある。
  - AWSの生ログや応答本文をそのまま公開する方式は、再現性に不要な識別情報を混入させやすいため採用しない。

## Research Log

### 言語に依存しないデータ契約

- **Context**: Lambda関数はPythonとGoで実装し、測定と分析はPythonで行うため、特定言語のクラスを契約の正本にすると下流がその言語へ依存する。
- **Sources Consulted**: [JSON Schema Core Draft 2020-12](https://json-schema.org/draft/2020-12/json-schema-core)、[JSON Schema Validation Draft 2020-12](https://json-schema.org/draft/2020-12/json-schema-validation)
- **Findings**:
  - JSON SchemaはJSON文書の構造を記述し、検証キーワードによって制約への適合を判定できる。
  - `$id`はスキーマ資源の正規URIを定め、`$ref`は分割したスキーマを参照できる。
  - URIは識別子であり、検証時にネットワークから取得できる必要はない。
  - `required`、`enum`、数値範囲などを利用して、必須項目と許容値を実装言語から独立して表現できる。
- **Implications**:
  - Draft 2020-12のJSON Schemaを正本にする。
  - `$id`にはリポジトリ固有のHTTPS URIを割り当てるが、検証時はローカルの登録表で解決する。
  - 未定義フィールドを黙って受け入れないため、各オブジェクトへ`additionalProperties: false`を設定する。

### Python検証ライブラリ

- **Context**: 検証CLIはスキーマ違反を一件目で終了せず、入力中の全違反を返す必要がある。
- **Sources Consulted**: [`jsonschema`の検証API](https://python-jsonschema.readthedocs.io/en/stable/validate/)
- **Findings**:
  - `Draft202012Validator.check_schema`でスキーマ自体をメタスキーマに照らして検証できる。
  - `iter_errors`は入力に対する違反を遅延列挙できる。
  - `format`は既定では注釈として扱われ、日時形式を検査するには`FormatChecker`を明示する必要がある。
  - 旧`RefResolver`は非推奨であり、参照解決には`referencing`の登録表を使う。
- **Implications**:
  - `Draft202012Validator`、`FormatChecker`、ローカル`Registry`を組み合わせる。
  - 起動区分や比較グループの整合性など、JSON Schemaだけで読みづらくなる規則は意味検証としてPythonに置く。
  - 検証結果はJSON Pointer形式の場所、機械判定用コード、人が読める説明を含む。

### 追記型の測定記録

- **Context**: すべての試行を残し、既存の生データを上書きせず、標本単位で追跡する必要がある。
- **Sources Consulted**: [JSON Lines](https://jsonlines.org/)
- **Findings**:
  - JSON Linesは一行を一つのJSON値として扱い、記録を一件ずつ処理できる。
  - UTF-8、各行が有効なJSON値、改行区切りという小さな規約で構成される。
  - `.jsonl`は慣用的な拡張子だが、MIME型は標準化されていない。
- **Implications**:
  - 測定記録の保存形式をUTF-8の`.jsonl`とする。
  - 空行と同じ`measurement_id`の重複を不合格にする。
  - CLIはファイル全体をメモリへ展開せず一行ずつ検証する。

### Lambdaの初期化根拠とランタイム版

- **Context**: コールドスタートの判定と実行基盤の版を、推測ではなく観測情報から記録する必要がある。
- **Sources Consulted**: [AWS Lambdaのランタイム版の識別](https://docs.aws.amazon.com/lambda/latest/dg/runtime-management-identify.html)、[AWS Lambda実行環境](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html)
- **Findings**:
  - Lambdaは新しい実行環境を作るとき、CloudWatch Logsへ`INIT_START`を出力する。
  - `INIT_START`にはランタイム版とランタイム版ARNが記録される。
  - 同じ実行環境を再利用する呼び出しでは`INIT_START`は出ない。
  - 実行失敗後の次回呼び出しでは、レポート上のDurationへ再初期化時間が含まれる場合があり、単純なDuration差分だけではコールドスタートと断定できない。
- **Implications**:
  - `INIT_START`をコールドスタートの直接根拠として記録する。
  - 実行環境再利用の識別子と`INIT_START`不在が両方そろう場合だけウォームスタートとする。
  - 根拠が欠ける場合と矛盾する場合は`unknown`とし、推測で補正しない。
  - ランタイム版とARNを取得できた場合は両方を保存する。

### 公開データの安全性

- **Context**: 公開リポジトリで再現性を確保しつつ、認証情報、AWSアカウントID、利用者固有名、無関係な入力を残してはならない。
- **Sources Consulted**: [AWSのアクセスキー管理](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)、要件7
- **Findings**:
  - 生のCloudWatchログや関数応答本文には、契約で許可していない値が混入し得る。
  - 計測に必要なのは解析済みの時間、起動根拠、環境属性、応答の一致確認であり、本文全体ではない。
  - Lambdaのランタイム版ARNはAWS管理のARNで、例示上はアカウントID部分が空である。
- **Implications**:
  - 公開測定記録は許可した構造化項目だけを持つ。
  - 応答は本文ではなくSHA-256ダイジェストで同一性を確認する。
  - 関数名、ロググループ名、アカウントIDは保存せず、契約内の別名へ変換する。
  - 禁止キー名とAWS ARNのアカウントID部分を公開前の意味検証で検査する。

## Architecture Pattern Evaluation

| Option | Description | Strengths | Risks / Limitations | Notes |
|---|---|---|---|---|
| JSON Schema正本と薄い検証CLI | スキーマが構造を定め、Pythonが意味規則と入出力を担当する | 言語非依存、差分レビューが容易、下流から再利用できる | 構造規則と意味規則が二か所に分かれる | 採用 |
| Pydanticモデル正本 | PythonモデルからJSON Schemaを生成する | Python実装が簡潔、型変換が容易 | GoとTerraformがPython生成物へ依存し、正本が見えにくい | 不採用 |
| Python独自検証 | Pythonコードだけで全規則を実装する | 自由度が高い | 契約の可搬性が低く、規則一覧を監査しにくい | 不採用 |
| 包括的なドメイン層 | エンティティ、リポジトリ、サービスを多層化する | 将来の拡張箇所を作りやすい | 現時点のファイル検証には層が過剰 | 不採用 |

## Design Decisions

### Decision: JSON Schema Draft 2020-12を正本にする

- **Context**: Python、Go、Terraform、記事生成処理が同じ契約改訂を参照する必要がある。
- **Alternatives Considered**:
  1. JSON Schema Draft 2020-12を直接管理する。
  2. PydanticモデルからJSON Schemaを生成する。
  3. Pythonの独自検証コードだけを提供する。
- **Selected Approach**: `scenario-set`、`experiment-plan`、`measurement-record`、`validation-report`をDraft 2020-12で定義する。
- **Rationale**: 言語に依存せず、スキーマの差分がそのまま契約変更としてレビューできる。
- **Trade-offs**: 複数文書をまたぐ整合性はPythonの意味検証へ分ける必要がある。
- **Follow-up**: 実装時にすべてのスキーマを`check_schema`へ通す。

### Decision: `contract/`を独立したuvプロジェクトにする

- **Context**: 後続の測定分析もPythonを使うが、ベンチマーク契約と依存関係や変更責任を混ぜたくない。
- **Alternatives Considered**:
  1. リポジトリ直下に一つのPythonプロジェクトを作る。
  2. `contract/`へ専用のPythonプロジェクトを作る。
  3. インストール不要の単一スクリプトにする。
- **Selected Approach**: `contract/pyproject.toml`と`contract/uv.lock`を持つ独立プロジェクトにする。
- **Rationale**: 後続仕様がルート設定を奪い合わず、契約の検証環境だけを固定できる。
- **Trade-offs**: 開発者は`uv run --project contract`を付けて実行する必要がある。
- **Follow-up**: CLI例と検証コマンドを`contract/README.md`へ記載する。

### Decision: 生ログではなく解析済みの測定記録を公開する

- **Context**: 起動根拠を保持しながら、AWS固有の識別情報や無関係な応答を公開データから除く必要がある。
- **Alternatives Considered**:
  1. CloudWatchログをそのまま保存する。
  2. ログを秘匿化して保存する。
  3. 許可項目だけを抽出した測定記録を保存する。
- **Selected Approach**: 計測処理がCloudWatchログを解析し、契約で許可した項目だけを`MeasurementRecord`へ保存する。
- **Rationale**: 公開データの項目を列挙でき、漏えい検査を決定的に実行できる。
- **Trade-offs**: パーサー不具合の調査に生ログが必要な場合、公開リポジトリ外の一時データを別途確認する必要がある。
- **Follow-up**: 後続の測定仕様で、原ログを公開ディレクトリへ書かないことを設計する。

### Decision: 構造検証と意味検証を順に実行する

- **Context**: JSON Schemaで表現しやすい規則と、シナリオ一覧や実験計画を参照する規則が混在する。
- **Alternatives Considered**:
  1. すべてをJSON Schemaへ押し込む。
  2. すべてをPythonへ置く。
  3. JSON Schemaの構造検証後にPythonの意味検証を行う。
- **Selected Approach**: 構文解析、契約改訂確認、構造検証、意味検証、公開安全検査の順に処理する。
- **Rationale**: 規則の置き場所を役割で分けながら、最終結果を一つの`ValidationReport`へ集約できる。
- **Trade-offs**: 同じ概念を二重実装しないよう、どちらが正本かを明確に保つ必要がある。
- **Follow-up**: 実装時に各違反コードの責任モジュールを固定する。

### Decision: 契約改訂は明示的な文字列で固定する

- **Context**: 未対応の契約を黙って受け入れず、測定と派生物が参照した定義を特定する必要がある。
- **Alternatives Considered**:
  1. ファイル名だけで版を表す。
  2. Gitコミットだけで版を表す。
  3. 各文書に`contract_revision`を持たせる。
- **Selected Approach**: 初回改訂を`1.0.0`とし、すべての契約文書と測定記録に`contract_revision`を必須とする。
- **Rationale**: 単一ファイルだけを受け取っても互換性を判定でき、Gitコミットは内容の来歴として別に保持できる。
- **Trade-offs**: 互換性の判断と改訂番号の更新をレビューで管理する必要がある。
- **Follow-up**: スキーマ変更時は依存仕様を再検証する。

## Risks & Mitigations

- JSON Schemaと意味検証で同じ規則を重複させる危険があるため、単一文書内の形はスキーマ、文書間の関係と公開安全条件はPythonに限定する。
- `format`を設定しただけで日時検証が動くと誤認する危険があるため、`FormatChecker`を必須にして不正日時のテストを置く。
- スキーマの`$ref`解決がネットワークへ依存する危険があるため、全スキーマをローカル登録表へ事前登録する。
- 生ログを保存しないことで調査材料が減るため、測定記録へ判定根拠と抽出元のイベント種別を明示する。
- JSON Linesは追記に向くが、形式だけでは既存行の変更を検出できないため、基準ファイルが現行ファイルの完全な接頭辞であることをCLIで検査する。

## References

- [JSON Schema Core Draft 2020-12](https://json-schema.org/draft/2020-12/json-schema-core) — スキーマ識別、参照、JSONデータモデルの正本。
- [JSON Schema Validation Draft 2020-12](https://json-schema.org/draft/2020-12/json-schema-validation) — 構造検証キーワードの正本。
- [`jsonschema` Schema Validation](https://python-jsonschema.readthedocs.io/en/stable/validate/) — Python検証APIと`format`検査の仕様。
- [uvのプロジェクト構造](https://docs.astral.sh/uv/concepts/projects/layout/) — `pyproject.toml`、プロジェクト環境、ロックファイルの管理方法。
- [JSON Lines](https://jsonlines.org/) — 行単位のJSON記録形式。
- [AWS Lambdaのランタイム版の識別](https://docs.aws.amazon.com/lambda/latest/dg/runtime-management-identify.html) — `INIT_START`とランタイム版情報。
- [AWS Lambda実行環境](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtime-environment.html) — 初期化、再利用、抑制された初期化の挙動。
- [AWS IAMのベストプラクティス](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html) — 長期認証情報を含む公開禁止情報の背景。

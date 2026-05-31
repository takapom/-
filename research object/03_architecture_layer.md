# アーキテクチャレイヤー方針

## 基本方針

採用するアーキテクチャは、LLM を飛行制御ループに入れない。LLM は、自然言語をミッション仕様へ変換し、失敗ログを解釈し、修復案を出す層に限定する。

低レベルの姿勢制御、安定化、安全停止は ArduPilot / Pixhawk 側に残す。AI ハーネスは、その上位で「実行してよいミッション候補だけを通す」役割を持つ。

## レイヤー構成

```text
Human Intent
  ↓
Natural Language Interface
  ↓
Intent Normalizer
  ↓
Mission IR Generator
  ↓
Constraint Binder
  ↓
Static Validator
  ↓
Code / Mission Emitter
  ↓
ArduPilot SITL Gate
  ↓
Runtime Monitor
  ↓
Real Vehicle Gate
  ↓
MAVLink / ArduPilot / Pixhawk
  ↓
Telemetry Logger
  ↓
Observability / Evaluation Store
  ↓
Failure Analyzer and Repair Loop
```

## 各レイヤーの責務

### 1. Natural Language Interface

人間の自由文指示を受け取る。日本語と英語のどちらも扱ってよいが、初期実験では日本語を主対象にして、必要に応じて英語対訳を作る。

責務:

- 指示本文の保存
- ユーザー、時刻、前回ミッションIDとの紐づけ
- 追加確認が必要な曖昧表現の検出候補を渡す

### 2. Intent Normalizer

自由文を、タスク目的、変更対象、制約、禁止条件、緊急条件に分ける。

例:

```yaml
intent:
  task: fly_square
  modification:
    altitude_max_m: 1.5
  constraints:
    no_fly_zones: [zone_a]
    speed_max_mps: 0.8
  safety:
    on_violation: land
```

ここではまだ実行コードを生成しない。曖昧性が高い場合は、人間確認または保守的な拒否に進める。

### 3. Mission IR Generator

研究上の最重要レイヤーである。LLM の第一出力を Python コードではなく、型付きの Mission IR にする。

Mission IR の最小要素:

- mission_id
- frame: local_ned / global
- takeoff altitude
- waypoints
- actions: takeoff, goto, hover, yaw, land, return_home
- constraints: altitude, speed, geofence, timeout, battery
- emergency policy
- previous_mission_reference

Mission IR は、JSON Schema または Pydantic で検証可能にする。内部実行座標は `NED` に統一し、高度は人間向けの `altitude_m` と実行向けの `position_ned_m[2]` を混同しない。Pythonコード生成を直接評価したい場合も、提案方式の中心には置かない。

### 4. Constraint Binder

自然言語の制約を、実行前に検査できる形式へ束縛する。

扱う制約:

- 高度上限・下限
- 速度上限
- 飛行可能範囲
- 進入禁止領域
- 最大飛行時間
- 最小バッテリー条件
- フェイルセーフ条件
- 実機検証に進めるための安全ゲート

将来的には STL / LTL / CBF などの形式手法へ接続できるが、最初のPoCではルールベース制約でよい。重要なのは、LLM の出力をそのまま信じず、決定的な制約検査層を置くことである。

### 5. Static Validator

Mission IR を実行前に検査する。

検査項目:

- スキーマ妥当性
- 単位の明示
- 座標系の明示
- 許可アクションのみ使用
- 高度・速度・ジオフェンス制約
- ミッション長とタイムアウト
- 危険な組み合わせ
- 未解決の曖昧性

Validator が落とした候補は、実行せず Failure Analyzer へ送る。

### 6. Code / Mission Emitter

検証済み Mission IR を ArduPilot SITL や MAVLink 実行用の具体形式へ変換する。

ここは可能な限り決定的にする。LLM で毎回コード全体を書かせるのではなく、テンプレートまたは固定変換器で Mission IR から実行コードを作る。

### 7. ArduPilot SITL Gate

実機前の必須ゲートである。

責務:

- SITLで実行可能か確認
- テレメトリを収集
- 制約違反を検出
- タスク成功・失敗を判定
- 実機検証へ進める候補だけを通す

実機検証は、SITL成功、静的検証通過、危険フラグなし、人間承認ありの場合に限定する。

### 8. Runtime Monitor

実行中の監視層である。SITL と実機の両方で使う。

監視項目:

- altitude
- velocity
- position
- distance to no-fly zone
- mode
- battery
- heartbeat
- timeout
- deviation from planned path

違反時は、LLMに判断させず、事前に決めた停止・着陸・return-to-homeへ移る。

### 9. Telemetry Logger

研究評価のために、すべての試行を再現可能なログとして残す。

最低限保存するもの:

- trace_id
- trial_id
- condition_id
- task_id
- prompt_version
- schema_version
- validator_version
- apply_policy_version
- natural_language_input
- normalized_intent
- mission_ir
- mission_patch
- validator_result
- emitted_code_or_mission
- sitl_result
- telemetry_trace
- failure_label
- repair_iteration
- repair_prompt
- repaired_mission_ir
- human_intervention

### 10. Observability / Evaluation Store

LLMOps / LLM observability は、ハーネスの安全判定層ではなく、数値化フェーズの観測・評価・再現性レイヤーとして置く。

責務:

- LLM 呼び出し、prompt、structured output、validator 結果、修復履歴を trial 単位の trace として保存する
- SITL / telemetry log と LLM trace を `trace_id` / `trial_id` で接続する
- 固定タスクセット上で prompt、schema、validator、apply policy の変更前後を比較する
- 論文用の集計表、失敗分類、回帰検出、再実行条件を保存する

初期候補:

| 役割 | 候補 |
| --- | --- |
| LLM trace / prompt version / evaluation record | Langfuse |
| Offline eval / regression / red teaming | promptfoo |
| OSS / OpenTelemetry寄りの trace 代替 | Phoenix, OpenTelemetry / OpenLLMetry |
| 実験 artifact / dataset versioning | MLflow, DVC |
| 学術寄り eval harness | Inspect AI |
| SITL / telemetry 時系列ログ | ArduPilot log, MCAP, Rerun |

これらは研究の測定基盤であり、提案方式そのものではない。安全制約の最終判定は Static Validator、Runtime Monitor、SITL Gate が担当する。

### 11. Failure Analyzer and Repair Loop

失敗を分類し、次の生成に戻す。

LLM に任せてよいこと:

- エラー文、制約違反、テレメトリ要約から原因候補を説明する
- Mission IR の修正案を出す
- 曖昧な指示への確認質問を作る

LLM に任せないこと:

- 安全制約の最終判定
- 実機実行可否の最終判定
- 飛行中の緊急判断

## 採用するシステム境界

| 領域 | 採用方針 |
| --- | --- |
| LLM | 高レベル仕様生成、失敗解釈、修復提案に限定 |
| Mission IR | 研究の中心。型付き、検証可能、ログ可能にする |
| Validator | 決定的。LLM出力の安全性を外部から検査する |
| SITL | 実機前ゲート。ほぼ全反復はここで行う |
| Observability / Evaluation Store | 数値化と再現性のための記録基盤。安全判定は担当しない |
| Pixhawk / ArduPilot | 低レベル制御、安全停止、フェイルセーフを担当 |
| Raspberry Pi | 実機側の伴走計算機。MAVLink通信とログ収集を担当 |
| Real Drone | 最終段階の限定検証。研究の中心実験はSITLで成立させる |

## アーキテクチャ上の重要な判断

### 判断1: 直接コード生成を主方式にしない

直接コード生成は比較ベースラインとして残すが、提案方式にはしない。理由は、コード生成では構文、API、単位、座標系、安全制約が同時に壊れるため、失敗原因の切り分けが難しいからである。

### 判断2: Mission IR を第一成果物にする

Mission IR を置くことで、自然言語理解、制約検査、実行変換、ログ評価を分離できる。これにより、研究として各層の寄与を評価できる。

### 判断3: 実機よりSITLを研究中心に置く

実機は必要だが、主張の中心にはしない。実機は、SITLで得た設計原理が限定条件下でも破綻しないかを見る確認に使う。

### 判断4: モデル比較を主張の中心にしない

Codex と Claude Code の比較は有用だが、モデルは更新される。研究の主張は、特定モデルの優劣ではなく、ハーネス構造が信頼性を改善するかに置く。

### 判断5: 失敗を成果物にする

成功デモだけでは研究として弱い。どの指示で、どの層が、どの種類の失敗を防げたかを示すことで、アーキテクチャ設計原理としての価値が出る。

## 次に決めるべき実装前仕様

実装に入る前に、次を固定する。

1. Mission IR の最小スキーマ
2. 許可するアクション集合
3. 最初の20個の自然言語タスク
4. 安全制約の初期値
5. Validator の判定項目
6. SITLログの保存形式
7. 失敗分類ラベル
8. 比較条件 B0-B4 のどこまでを最初のPoCに含めるか

この8項目が固まれば、実装方針に進める。

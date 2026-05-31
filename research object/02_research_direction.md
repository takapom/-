# 研究方針

## 推奨タイトル

暫定タイトル:

> 状態保持型 AI ハーネスによる自然言語UAVミッション変更の安全制約付き実行変換

英語タイトル案:

> A Stateful AI Harness for Safety-Constrained Translation of Natural-Language UAV Mission Modifications

## 研究対象

対象は、自然言語で与えられるUAVミッションの初期生成および変更指示である。特に、次のような指示を扱う。

- 「高度を1.5m以下に抑えて四角形経路を飛ぶ」
- 「同じ経路で速度だけ下げる」
- 「この領域には入らずに目的地へ向かう」
- 「前回失敗した離陸後のふらつきを避けて再実行する」
- 「異常があれば停止または着陸する」

研究の対象外を明確にする。

- LLM に低レベル姿勢制御を直接行わせない
- 高周波制御器やフライトコントローラ自体を新規開発しない
- 新規VLAモデルの学習を主貢献にしない
- 実機での長距離・屋外・複雑環境飛行を主実験にしない

## 中心仮説

H1: 自然言語から直接コードを生成する方式より、型付きミッション中間表現を介して生成・検証・実行する方式の方が、安全制約違反と実行時エラーを減らす。

H2: 状態保持と実行ログ解釈を含む閉ループハーネスは、単発生成方式より、同一または類似指示に対する出力分散、修復回数、人間介入回数を減らす。

H3: LLM を高レベルの仕様生成・失敗解釈・修復提案に限定し、低レベル制御と安全停止を ArduPilot / Pixhawk 側に分離することで、実機移行時の危険な失敗を抑えられる。

## 研究問い

RQ1: 自然言語UAVミッション変更において、直接コード生成と中間ミッション仕様生成では、エラー率、安全制約違反、タスク成功率にどの程度の差が出るか。

RQ2: 状態保持、制約検査、SITL評価、ログ解釈、修復ループの各要素は、信頼性指標にどの程度寄与するか。

RQ3: どの種類の自然言語指示が、曖昧性、制約衝突、座標系誤り、API誤用、過剰な自由度を引き起こしやすいか。

RQ4: シミュレーションで安全と判定されたミッションは、限定条件下の実機検証でどの程度同じ失敗傾向・軌道傾向を示すか。

## 貢献の形

この研究で狙う貢献は、性能SOTAではなく、設計原理と評価可能なシステム化である。

1. 自然言語UAVミッション変更のための状態保持型 AI ハーネスアーキテクチャ
2. 自然言語指示、ミッション仕様、制約、実行ログ、修復履歴を含む評価プロトコル
3. 直接コード生成、プロンプト制約、中間表現、閉ループ修復を比較するアブレーション
4. LLM 由来のUAVミッション失敗分類
5. SITLから限定実機検証へ進むための実行ゲート設計

LLMOps / LLM observability は、この貢献の中心には置かない。役割は、非決定的な LLM 出力を trial 単位で記録し、prompt / schema / validator / apply policy の変更前後を同じタスクセットで比較し、失敗分類と改善量を再現可能にすることである。したがって、Langfuse や promptfoo などのツールは研究対象ではなく、評価プロトコルを支える測定基盤として扱う。

数値化フェーズで最低限追跡するもの:

- `trace_id`, `trial_id`, `condition_id`, `task_id`
- `model_version`, `prompt_version`, `schema_version`, `validator_version`, `apply_policy_version`
- Mission IR / Mission IR patch
- validator result, SITL result, runtime monitor result
- failure label, repair iteration, human intervention
- latency, token usage, cost, output variance

## 比較条件

比較対象はモデル名ではなく、ハーネス構成に置く。

| 条件 | 内容 | 目的 |
| --- | --- | --- |
| B0: Direct Code | 自然言語からMAVLink/DroneKit/Pythonコードを直接生成 | 最も危険な素朴ベースライン |
| B1: Prompted Code | プロンプトで制約を与えた上でコード生成 | プロンプト制約だけで足りるかを見る |
| B2: Mission IR | 自然言語から型付きミッション中間表現を生成し、決定的にコード化 | 中間表現の効果を見る |
| B3: Mission IR + Validator | 中間表現に静的制約検査を加える | 安全違反の事前排除効果を見る |
| B4: Stateful Harness | 状態保持、SITL評価、ログ解釈、修復を加える | 提案方式 |

Codex と Claude Code は、各条件の中で使用する生成器として比較してよい。ただし結論は「どちらのモデルが優秀か」ではなく、「モデルが変わっても有効なハーネス条件は何か」に置く。

## タスクセット

最初のベンチマークは、過度に複雑にしない。小さくても、失敗が分類できるタスクセットにする。

### Primitive Tasks

- arm / disarm
- takeoff to fixed altitude
- hover for fixed duration
- move forward / backward / left / right
- yaw rotation
- land

### Geometric Mission Tasks

- square path
- triangle path
- waypoint sequence
- return-to-home
- loiter at waypoint

### Mission Modification Tasks

- altitude cap modification
- speed cap modification
- route shape modification
- no-fly zone insertion
- return condition insertion
- emergency stop / land condition insertion

### Ambiguous or Conflicting Tasks

- 「なるべく低く速く飛ぶ」
- 「障害物を避けつつ最短で戻る」
- 「さっきより安全に同じことをする」
- 「高度1m以下で障害物を越える」

このカテゴリは成功率だけでなく、ハーネスが曖昧性確認、制約衝突、拒否、人間確認へ適切に移れるかを見る。

## 評価指標

### 生成・仕様レベル

- JSON / YAML / DSL 構文妥当率
- ミッションスキーマ妥当率
- 単位、座標系、フレーム指定の正確性
- 禁止API呼び出し数
- 許可アクション外の出力数
- 同一指示を複数回与えたときの出力分散

### シミュレーション実行レベル

- SITL 実行成功率
- タスク完了率
- 到達誤差
- 最大軌道逸脱量
- 高度上限・速度上限・ジオフェンス違反数
- 実行時エラー率
- 修復ループ回数
- 失敗原因分類の正解率

### 実機限定検証レベル

- 人間介入回数
- フェイルセーフ発火回数
- 低高度・短時間タスク成功率
- SITL と実機の軌道差
- 実行前ゲートで止められた危険候補数

## 失敗分類

失敗分類は、研究の重要な成果物にする。

| 分類 | 例 |
| --- | --- |
| Syntax Failure | 生成物がJSON/DSL/Pythonとして壊れている |
| API Misuse | MAVLinkやライブラリ呼び出しが誤っている |
| Unit/Frame Error | m/cm、NED/ENU、相対/絶対座標を取り違える |
| Constraint Violation | 高度、速度、ジオフェンス、安全停止条件を破る |
| Ambiguity Failure | 指示が曖昧なのに確認せず実行候補を作る |
| State Loss | 前回の経路、失敗、制約を忘れる |
| Over-Repair | 修復により元タスクの意味が失われる |
| Runtime Instability | シミュレーション上で姿勢・高度・経路が不安定になる |

## 180日PoCの研究マイルストーン

| 期間 | 到達点 |
| --- | --- |
| 0-30日 | Mission IR、制約スキーマ、タスクセット、評価ログ形式を固定 |
| 31-60日 | Direct Code / Prompted Code / Mission IR のSITL比較 |
| 61-90日 | Validator、状態保持、ログ解釈、修復ループの追加 |
| 91-120日 | アブレーション実験、失敗分類、出力分散評価 |
| 121-150日 | Raspberry Pi + Pixhawk 接続、プロペラなしまたは固定状態で通信確認 |
| 151-180日 | 限定条件下の実機検証、SITLとの差分分析、設計原理の整理 |

## 最初の2週間のパイロット

2週間で見るべき信号は、実機ではなく SITL と Mission IR である。

- 20個の自然言語指示を作る
- Mission IR の最小スキーマを作る
- Direct Code と Mission IR の2条件だけ比較する
- ArduPilot SITLで takeoff、square path、altitude cap、no-fly zone の4系統を試す
- 生成失敗、制約違反、SITL失敗、修復可能性を記録する

この段階で、Mission IR 方式が Direct Code より明確に安定しない場合、研究方針を再検討する。

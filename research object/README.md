# Research Object: Natural-Language UAV Control Harness

このディレクトリは、自然言語によるドローンタスク変更を安全制約付きの実行可能ミッションへ変換する研究案を整理するための作業領域である。

## 読む順序

1. [00_agent_research_sot.md](00_agent_research_sot.md): AIエージェント向けの研究SoT、参照すべきskills path、作業上の注意
2. [hogehoge.md](hogehoge.md): 既存の研究目的・研究方法・AI利活用妥当性のドラフト
3. [01_as_is_to_be.md](01_as_is_to_be.md): 現状案の as-is / to-be と、研究として弱い点・強くする方向
4. [02_research_direction.md](02_research_direction.md): 研究方針、研究問い、仮説、貢献、評価設計
5. [03_architecture_layer.md](03_architecture_layer.md): 実装前に固定すべきアーキテクチャ層と採用方針
6. [04_realtime_mission_modification.md](04_realtime_mission_modification.md): 実行中ミッション変更とリアルタイム性を研究に入れる意味、問い、実装境界
7. [05_evaluation_parameters.md](05_evaluation_parameters.md): 数値評価の基準、主指標、補助指標、初期PoCで採用すべきパラメータ
8. [06_experimental_design.md](06_experimental_design.md): 比較条件、タスクセット、正解ラベル、アブレーション、統計処理、ヒアリング事項

## 現時点の結論

この研究は「LLMでドローンを飛ばす」ことを主題にしない。主題は、自然言語で与えられた目的変更を、状態保持・制約検査・シミュレーション評価・ログに基づく修復を通じて、安全制約付きのUAVミッションへ変換する AI ハーネスの設計条件を明らかにすることである。

Codex と Claude Code の比較は中心的貢献ではなく、ハーネス内で使う生成器の違いが信頼性・再現性・修復回数にどう影響するかを調べる実験条件として扱う。

追加の重要論点として、初期ミッション生成だけでなく、飛行中の自然言語によるミッション変更を扱う。これにより、状態保持、オンライン再計画、安全ゲート、実行中の介入判断を研究対象にできる。ただし、LLM を高周波制御ループへ入れるのではなく、ミッション変更の解釈・検証・再計画に限定する。

評価は、飛行性能の最高値ではなく、自然言語由来のミッション生成・変更を安全に処理できたかに置く。初期PoCでは、安全制約違反率、危険指示の誤受理率、変更処理成功率、意思決定遅延、状態整合率を主指標にする。

LLMOps / LLM observability は主貢献ではなく、数値化フェーズのための観測・評価・再現性レイヤーとして扱う。Langfuse は trial 単位の trace、prompt version、Mission IR / patch、validator 結果、修復履歴の保存に使う候補とし、promptfoo は固定タスクセットに対する offline eval / regression / red teaming に使う候補とする。追加候補として Phoenix、OpenTelemetry / OpenLLMetry、MLflow、DVC、Inspect AI、MCAP / Rerun を検討してよいが、安全判定の最終責任は常に deterministic Validator / Runtime Monitor / SITL Gate に置く。

実験設計では、提案方式をひとまとめに評価しない。Mission IR、Validator、State Store、Mission Patch、Repair Loop を段階的に足すアブレーションにし、どの構成要素が安全性・状態保持・実行中変更処理に効いたかを分離して示す。

# ArduPilot SITL 起動・使用メモ

このディレクトリは、自然言語UAVミッション変更研究で使う ArduPilot SITL 環境のローカルメモである。

## ローカル構成

```text
ardupilot/
├── README.md                 # このファイル
├── drone-ai-harness/         # Python venv と研究側ハーネス用環境
│   ├── .venv/
│   └── requirements.lock.txt
└── tools/
    └── ardupilot/            # ArduPilot 本体
        ├── ArduCopter/
        ├── Tools/autotest/sim_vehicle.py
        ├── build/sitl/bin/arducopter
        └── waf
```

ArduPilot 本体のローカルHEAD:

```bash
git -C /Users/takagiyuuki/AI-Research-SKILLs/ardupilot/tools/ardupilot rev-parse --short HEAD
# c979b1c8d9
```

## 基本方針

- 対象は `ArduCopter SITL`
- 操作は原則としてシミュレーション内で行う
- 実機検証は現段階では対象外
- 将来の実機移行を考え、MAVLink / ArduPilot / Pixhawk に移しやすい境界を保つ
- 内部実行座標は `NED`
- 自然言語や研究ログでは `altitude_m` を使い、実行時に `z = -altitude_m` へ変換する

## 事前確認

このローカル環境では、`ardupilot/drone-ai-harness/.venv` に `MAVProxy`、`pymavlink`、`pexpect` などが入っている。

まず venv を有効化する。

```bash
cd /Users/takagiyuuki/AI-Research-SKILLs/ardupilot/drone-ai-harness
source .venv/bin/activate
```

確認:

```bash
python -c 'import pexpect, pymavlink; print("ok")'
```

`sim_vehicle.py` を素のPythonで実行すると `pexpect` 不足で落ちる場合がある。その場合は必ず上記venvを有効化してから実行する。

## 初回ビルド

ArduPilot 本体へ移動する。

```bash
cd /Users/takagiyuuki/AI-Research-SKILLs/ardupilot/tools/ardupilot
```

SITL向けにconfigureする。

```bash
./waf configure --board sitl
```

ArduCopterをビルドする。

```bash
./waf copter
```

ビルド済みバイナリは以下に生成される。

```text
/Users/takagiyuuki/AI-Research-SKILLs/ardupilot/tools/ardupilot/build/sitl/bin/arducopter
```

この環境では、上記バイナリは既に存在している。

## 最短起動

推奨は `ArduCopter/` から起動すること。SITLの `eeprom.bin` やログが作業ディレクトリに作られるため、車種ディレクトリごとに状態を分けやすい。

```bash
cd /Users/takagiyuuki/AI-Research-SKILLs/ardupilot/drone-ai-harness
source .venv/bin/activate

cd /Users/takagiyuuki/AI-Research-SKILLs/ardupilot/tools/ardupilot/ArduCopter
../Tools/autotest/sim_vehicle.py -v ArduCopter -f quad --no-rebuild -w
```

意味:

- `-v ArduCopter`: ArduCopterを起動
- `-f quad`: quad frame
- `--no-rebuild`: 既存ビルドを使う
- `-w`: EEPROMを初期化してデフォルトパラメータを読み直す

初回やコード変更後にビルドも含めて起動したい場合は `--no-rebuild` を外す。

```bash
../Tools/autotest/sim_vehicle.py -v ArduCopter -f quad -w
```

## 研究用の推奨起動

外部ハーネスからMAVLink接続しやすいように、UDP出力を明示する。

```bash
cd /Users/takagiyuuki/AI-Research-SKILLs/ardupilot/drone-ai-harness
source .venv/bin/activate

cd /Users/takagiyuuki/AI-Research-SKILLs/ardupilot/tools/ardupilot/ArduCopter
../Tools/autotest/sim_vehicle.py \
  -v ArduCopter \
  -f quad \
  --no-rebuild \
  -w \
  --out=udp:127.0.0.1:14550
```

研究側ハーネスは、基本的に `udp:127.0.0.1:14550` を読む。

必要に応じて別ポートも追加できる。

```bash
../Tools/autotest/sim_vehicle.py \
  -v ArduCopter \
  -f quad \
  --no-rebuild \
  -w \
  --out=udp:127.0.0.1:14550 \
  --out=udp:127.0.0.1:14551
```

## MAVProxyでの基本操作

`sim_vehicle.py` は通常 MAVProxy を起動する。起動後、MAVProxyプロンプトで以下を使う。

状態確認:

```text
status
mode
param show ARMING_CHECK
```

GUIDEDに変更:

```text
mode GUIDED
```

arm:

```text
arm throttle
```

離陸:

```text
takeoff 1.5
```

停止・待機系:

```text
mode LOITER
```

着陸:

```text
mode LAND
```

disarm:

```text
disarm
```

終了:

```text
exit
```

研究では、手動MAVProxy操作は動作確認用に限定する。評価実験では、Mission IR / Mission Patch から固定Emitterを通してMAVLink命令を出す。

## GUI・地図を使う場合

GUI依存が入っている場合のみ、`--console` と `--map` を付ける。

```bash
../Tools/autotest/sim_vehicle.py -v ArduCopter -f quad --no-rebuild -w --console --map
```

GUI依存が不足している場合は、headless起動を使う。

```bash
../Tools/autotest/sim_vehicle.py -v ArduCopter -f quad --no-rebuild -w
```

## ログと状態ファイル

SITLは起動ディレクトリに状態ファイルやログを作る。

代表例:

```text
eeprom.bin
mav.parm
mav.tlog
mav.tlog.raw
```

研究評価では、試行ごとにログを分ける。`--use-dir` を使うとSITL状態と出力先を分けやすい。

```bash
../Tools/autotest/sim_vehicle.py \
  -v ArduCopter \
  -f quad \
  --no-rebuild \
  -w \
  --use-dir /Users/takagiyuuki/AI-Research-SKILLs/research/sitl_runs/trial_0001 \
  --out=udp:127.0.0.1:14550
```

評価ログ側には、少なくとも以下を保存する。

```yaml
trial_id: trial_0001
sitl_use_dir: /Users/takagiyuuki/AI-Research-SKILLs/research/sitl_runs/trial_0001
mavlink_out: udp:127.0.0.1:14550
vehicle: ArduCopter
frame: quad
local_frame: NED
wipe_eeprom: true
```

## よく使うオプション

| オプション | 用途 |
| --- | --- |
| `-v ArduCopter` | ArduCopterを起動 |
| `-f quad` | quad frameで起動 |
| `--no-rebuild` | 既存ビルドを使って高速起動 |
| `-w` | EEPROM初期化。試行条件を揃えたいときに使う |
| `--out=udp:127.0.0.1:14550` | 外部ハーネス用MAVLink出力 |
| `--use-dir <dir>` | 試行ごとに状態・ログを分離 |
| `--speedup <n>` | シミュレーション速度を変更 |
| `--location <name>` | `Tools/autotest/locations.txt` の地点を使う |
| `--custom-location lat,lon,alt,heading` | 任意の開始地点を使う |
| `--console --map` | MAVProxy console/mapを表示 |

## トラブルシュート

### `ModuleNotFoundError: No module named 'pexpect'`

venvが有効化されていない可能性が高い。

```bash
cd /Users/takagiyuuki/AI-Research-SKILLs/ardupilot/drone-ai-harness
source .venv/bin/activate
python -c 'import pexpect; print("pexpect ok")'
```

それでも失敗する場合は、ArduPilot公式のMac向け前提パッケージを確認する。

```bash
cd /Users/takagiyuuki/AI-Research-SKILLs/ardupilot/tools/ardupilot
less Tools/environment_install/install-prereqs-mac.sh
```

### `build/sitl/bin/arducopter` がない

SITLビルドを実行する。

```bash
cd /Users/takagiyuuki/AI-Research-SKILLs/ardupilot/tools/ardupilot
./waf configure --board sitl
./waf copter
```

### 起動状態が前回試行に引きずられる

`-w` を付けて EEPROM を初期化する。

```bash
../Tools/autotest/sim_vehicle.py -v ArduCopter -f quad --no-rebuild -w
```

また、研究試行では `--use-dir` で試行ごとに状態を分ける。

### ポートが競合する

既存のSITLやMAVProxyを終了する。複数起動したい場合は `-I` でinstanceを分ける。

```bash
../Tools/autotest/sim_vehicle.py -v ArduCopter -f quad --no-rebuild -I 1 --out=udp:127.0.0.1:14560
```

## 研究ハーネスとの接続方針

この研究では、SITLを直接「自然言語で動かす」のではなく、次の順序で接続する。

```text
日本語指示
  ↓
Mission IR / Mission Patch
  ↓
Validator
  ↓
Emitter
  ↓
MAVLink command
  ↓
ArduCopter SITL
  ↓
Telemetry log
```

SITLは評価対象の物理実行環境であり、LLMを低レベル制御ループには入れない。

## 次に作るもの

- `research/sitl_runs/` の試行ログ保存ルール
- Mission IRからMAVLinkへ変換する最小Emitter
- `udp:127.0.0.1:14550` を読むテレメトリロガー
- 50タスクの日本語ベンチマーク

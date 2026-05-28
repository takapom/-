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
- UI確認は `QGroundControl` を前提にする
- 操作は原則としてシミュレーション内で行う
- 実機検証は現段階では対象外
- 将来の実機移行を考え、MAVLink / ArduPilot / Pixhawk に移しやすい境界を保つ
- 内部実行座標は `NED`
- 自然言語や研究ログでは `altitude_m` を使い、実行時に `z = -altitude_m` へ変換する

## QGroundControl 前提の接続構成

この研究では、QGroundControlを機体状態のUI確認に使う。研究ハーネスはQGroundControl経由ではなく、SITL/MAVProxyから別UDPポートで直接MAVLinkを受ける。

推奨ポート:

| 用途 | UDPポート | 接続先 |
| --- | --- | --- |
| QGroundControl UI | `14550` | `udp:127.0.0.1:14550` |
| 研究ハーネス / telemetry logger | `14551` | `udp:127.0.0.1:14551` |

この分離により、QGroundControlで状態を見ながら、研究ハーネス側でMission IR / Mission Patchの実行とログ収集を行える。

注意:

- QGroundControlは監視UIとして使う
- 評価実験では、QGroundControlからミッション編集・アップロード・手動コマンド送信をしない
- 手動のarm/takeoff/landは動作確認時だけに限定する
- 研究本番では、Mission IR / Mission Patchから固定Emitter経由でMAVLink命令を出す

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

## 起動方法

この環境では、QGroundControlでUI確認しながらSITLを使う。通常は「QGroundControl用の推奨起動」を使う。Codexなどの非対話セッションから起動する場合は「非対話セッション用の起動」を使う。

### 最短起動

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

### QGroundControl用の推奨起動

QGroundControlと研究ハーネスを同時に使えるように、UDP出力を2つ明示する。

```bash
cd /Users/takagiyuuki/AI-Research-SKILLs/ardupilot/drone-ai-harness
source .venv/bin/activate

cd /Users/takagiyuuki/AI-Research-SKILLs/ardupilot/tools/ardupilot/ArduCopter
../Tools/autotest/sim_vehicle.py \
  -v ArduCopter \
  -f quad \
  --no-rebuild \
  -w \
  --out=udp:127.0.0.1:14550 \
  --out=udp:127.0.0.1:14551
```

起動後、QGroundControlを開く。通常はUDP `14550` を自動検出して接続される。

自動接続されない場合は、QGroundControl側で手動UDP Linkを作る。

```text
Application Settings
  → Comm Links
  → Add
  → Type: UDP
  → Listening Port: 14550
  → Connect
```

画面やメニュー名はQGroundControlのバージョンで多少変わることがある。

研究側ハーネスは、基本的に `udp:127.0.0.1:14551` を読む。

QGroundControlだけ確認したい場合は、14550だけでもよい。

```bash
../Tools/autotest/sim_vehicle.py \
  -v ArduCopter \
  -f quad \
  --no-rebuild \
  -w \
  --out=udp:127.0.0.1:14550
```

### 非対話セッション用の起動

通常のターミナルでは、上の「QGroundControl 用の推奨起動」を使えばよい。

Codexや非対話セッションから起動する場合は、venvのPythonとPATHを明示する。この形式なら、`pexpect` と `mavproxy.py` の解決漏れを避けやすい。

```bash
cd /Users/takagiyuuki/AI-Research-SKILLs/ardupilot/tools/ardupilot/ArduCopter

env PATH=/Users/takagiyuuki/AI-Research-SKILLs/ardupilot/drone-ai-harness/.venv/bin:/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin \
  /Users/takagiyuuki/AI-Research-SKILLs/ardupilot/drone-ai-harness/.venv/bin/python \
  ../Tools/autotest/sim_vehicle.py \
  -v ArduCopter \
  -f quad \
  --no-rebuild \
  -w \
  --out=udp:127.0.0.1:14550 \
  --out=udp:127.0.0.1:14551
```

今回の起動確認では、以下の状態まで確認した。

```text
ArduCopter SITL 起動
MAVProxy 接続
heartbeat 受信
mode: STABILIZE
QGroundControl 側からの MAVLink request ACK 受信
```

## QGroundControlで確認するもの

QGroundControlでは、主に以下を確認する。

- Vehicleが接続済みになっているか
- modeが `STABILIZE` / `GUIDED` / `LOITER` / `LAND` などに変化するか
- armed / disarmed 状態
- altitude
- groundspeed
- attitude
- GPS / EKF status
- MAVLink messages
- trajectory / map上の位置

研究ログと照合するため、QGroundControl上の表示値は参考表示として扱い、正式な評価値はtelemetry logger側のMAVLinkログから計算する。

## QGroundControlでの手動確認

動作確認時だけ、QGroundControlから以下を確認してよい。

- 機体が接続される
- arm/disarmできる
- GUIDEDまたはLOITERへmode変更できる
- takeoff後に高度が上がる
- LANDで着陸状態へ移る

評価実験では、QGroundControlから手動操作を入れるとログが汚れるため、UI確認に限定する。

## MAVProxyの位置づけ

`sim_vehicle.py` は通常 MAVProxy を起動する。起動後、MAVProxyプロンプトで以下を使う。

この研究では、MAVProxyは主UIではなく、SITLとQGroundControl/研究ハーネスをつなぐ中継・デバッグ用として扱う。

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

## 停止方法

今回の環境では、MAVProxyプロンプトで `exit` を入力しても終了コマンドとして扱われなかった。SITLを止めるときは、起動したターミナルで `Ctrl-C` を送る。

```text
Ctrl-C
```

停止後、残存プロセスがないか確認する。

```bash
pgrep -fl 'arducopter|mavproxy|sim_vehicle' || true
```

何も表示されなければ停止済みである。

## MAVProxy GUI・地図を使う場合

QGroundControlを使う場合、通常はMAVProxyの `--console` / `--map` は不要である。MAVProxy側でもGUIを出したい場合のみ、`--console` と `--map` を付ける。

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
  --out=udp:127.0.0.1:14550 \
  --out=udp:127.0.0.1:14551
```

評価ログ側には、少なくとも以下を保存する。

```yaml
trial_id: trial_0001
sitl_use_dir: /Users/takagiyuuki/AI-Research-SKILLs/research/sitl_runs/trial_0001
qgroundcontrol_link: udp:127.0.0.1:14550
harness_link: udp:127.0.0.1:14551
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
| `--out=udp:127.0.0.1:14550` | QGroundControl用MAVLink出力 |
| `--out=udp:127.0.0.1:14551` | 研究ハーネス用MAVLink出力 |
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

### QGroundControlが接続されない

確認すること:

1. SITLが起動している
2. 起動コマンドに `--out=udp:127.0.0.1:14550` が入っている
3. QGroundControl側でUDP auto-connectが有効
4. 手動Linkを使う場合、Listening Portが `14550`
5. 既に別プロセスが `14550` を使っていない

QGroundControlと研究ハーネスを同じUDPポートに接続しない。QGroundControlは `14550`、研究ハーネスは `14551` に分ける。

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

QGroundControl UI confirmation reads the same SITL state in parallel
```

SITLは評価対象の物理実行環境であり、LLMを低レベル制御ループには入れない。
QGroundControlは評価値を計算する主体ではなく、状態監視と手動確認のUIとして使う。

## 次に作るもの

- `research/sitl_runs/` の試行ログ保存ルール
- Mission IRからMAVLinkへ変換する最小Emitter
- `udp:127.0.0.1:14551` を読むテレメトリロガー
- 50タスクの日本語ベンチマーク

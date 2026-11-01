# PAI2026 最終課題: CRANE+ V2 の模倣学習（gururi タスク）

「Physical AI講座2026 応用編1」の最終課題です。
Gazebo 上の CRANE+ V2 で、テレオペによるデモ記録 → ACT 学習 → 推論 を行いました。

講義で扱った、お辞儀をする「pekori」タスクに旋回を加えた **「gururi」タスク（お辞儀しながら旋回）** を実施し、
その過程で見つかった「推論が原点から動き出せない」問題を調査しました。

## リポジトリ構成

```
.
├── README.md
├── pai_training/          # 学習用コード（外部リポジトリ、submodule）
├── physicalai_teleop/     # 遠隔操作用 ROS 2 パッケージ（外部リポジトリ、submodule）
├── dataset/
│   └── rosbags/gururi/    # 収集したデモ（rosbag2, ep0〜ep9）
└── checkpoints/           # 学習済みモデル（git 管理外。Releases から取得）
    └── train/
        ├── crane_plus_gururi_act_steps200/
        └── crane_plus_gururi_act_steps1000/
```

### 使用した外部リポジトリ

| ディレクトリ | 元リポジトリ | 使用コミット | 変更 |
|---|---|---|---|
| `pai_training` | https://github.com/matsuolab/pai_training | `b4d6b54`（submodule `lerobot` は `768e366`） | 設定ファイルのみ（下記） |
| `physicalai_teleop` | https://github.com/matsuolab/physicalai_teleop | `0ef2a00` | なし |

### 学習済みモデル

`model.safetensors`（各約 153MB）は GitHub のファイルサイズ上限を超えるため、
[Releases](../../releases) に添付しています。ダウンロードして次の場所に置いてください。

```
checkpoints/train/crane_plus_gururi_act_steps200/{config.json, model.safetensors}
checkpoints/train/crane_plus_gururi_act_steps1000/{config.json, model.safetensors}
```

## 環境

| 項目 | 内容 |
|---|---|
| PC | Windows 11、RAM 16GB、GPU なし、Docker Desktop |
| Docker イメージ | `airobotbook/ros2-desktop-ai-robot-book-humble:latest`（2026年10月初旬に取得） |
| ソフト | ROS 2 Humble、Gazebo Classic 11.10.2、pixi 0.81.0 |
| 画面 | コンテナ内のデスクトップをブラウザ（VNC、`localhost:6080`）で操作 |
| 学習 | CPU のみ。WandB は使わない（`WANDB_MODE=offline`） |

コンテナ内では `physicalai_teleop` を `~/pai_ws/src`、`pai_training` を `~/pai_training` に置き、
データは `~/dataset`、モデルは `~/checkpoints/train` に保存しました。

## 実験手順

### 1. デモの収集

```bash
# ターミナルA: シミュレータ
ros2 launch crane_plus_gazebo crane_plus_with_table.launch.py

# ターミナルB: テレオペ（ターミナルを開き直すたびに source が必要）
source ~/pai_ws/install/setup.bash
ros2 run physicalai_teleop teleop_keyboard

# ターミナルC: 記録（エピソードごとに番号を変える）
ros2 bag record -a -o ~/dataset/rosbags/gururi/ep0
```

各エピソードで joint3 を S で 20 回押してお辞儀させ、続けて joint1 を S で 20 回押して旋回させました（戻さずに終了）。
pekori と同じく 10 エピソード（ep0〜ep9、各 17.7〜22.9 秒）を収集しました。

### 2. 変換・学習・推論

`pai_training/configs` の 3 ファイルで、タスク名とパスの `pekori` を `gururi` に置き換えました。

- `conversion_config.yaml`: `bag_dir`、`output_dir`（`target_freq: 20`）
- `crane_features.py`: `repo_id`、`root`、`task_name`
- `train_config.yaml`: `dataset_root`、`output_directory`、`training_steps`（200 / 1000）

その他の設定は元のままです（`device: cpu`、`batch_size: 2`、`chunk_size: 16`、`n_action_steps: 1`、`learning_rate: 1e-4`）。

```bash
cd ~/pai_training
pixi run convert                       # rosbag → LeRobotDataset
WANDB_MODE=offline pixi run train      # 学習
PYTHONUNBUFFERED=1 pixi run infer --model-path /home/ubuntu/checkpoints/train/crane_plus_gururi_act_steps1000
```

## 結果

| 実験 | steps | 後半の平均 loss | 推論の結果 |
|---|---|---|---|
| gururi-1 | 200 | 約 3.56（150〜199） | 浅くお辞儀して止まる |
| gururi-2 | 1000 | 約 1.39（950〜999） | Gazebo 起動直後は成功、テレオペで原点に戻した後は失敗 |

- steps=200 では loss が 100 ステップ以降横ばいに見えたが、steps=1000 では後半も直線的に下がり続けた。200 ステップの横ばいは収束ではなく途中の停滞だった。
- steps=1000 のモデルは、Gazebo 起動直後に推論するとお辞儀 → 旋回ができた。一方、テレオペの X で原点に戻してから推論すると、わずかにお辞儀して止まった（再現性あり）。
- テレオペ後に joint3 を S で 8 回以上（約 0.4 rad）押してから推論すると成功した（7 回までは失敗）。

## 考察

**起動直後とテレオペ後の違いはグリッパの値だけ** でした（腕の 4 関節の差は 1e-6 rad 未満）。
グリッパは起動直後が約 0、テレオペ後が −0.5236 で、学習データは全フレーム −0.5236 に固定されています。

- **起動直後の「成功」**: グリッパ約 0 は正規化すると約 2.6e5σ という学習範囲外の入力になり、最初の指令が深いお辞儀の姿勢（joint1=−0.41, joint3=−1.04）へ飛んだ。原点から自力で動き出したわけではない。
- **テレオペ後の失敗**: 入力は正常だが、原点付近の学習フレームの大半が「静止」なので、出力が現在の姿勢とほぼ同じになり動き出せない。
  - 各エピソードの先頭に平均約 48 フレーム（約 2.4 秒）の静止区間がある
  - 腕の追従に 0.4 秒以上の遅れがあり、動き出した後もしばらく状態が原点に残る
  - ACT は L1 損失で学習するため、原点付近の出力が多数派（動かない）に引っ張られやすい（推測）

他の仮説（偶然のばらつき、テレオペとの指令競合、開始姿勢のずれ、正規化の増幅、途中姿勢のデータ不足、指令遅れ）はいずれも測定で否定しました。

**今後の対策（未検証）**: 変換時に先頭と末尾の静止区間を削って（余白を残して）学習し直し、上の解釈が正しいか確かめる。

# G1 Whole-Body Control OSS 比較レポート

**作成日**: 2026-02-20
**目的**: G1清掃デモ (全身テレオペ) + 将来の自動化に向けたOSS選定

---

## 背景

### デモ目標
- 全身テレオペで清掃デモを実施
- 必要な動作: 階段の上り下り、道具・廃棄物のpick and place

### ハードウェア構成
- **ロボット**: Unitree G1
- **ハンド**: Amazing Hand (必須 - Unitree 3指ハンドは未所持)

### 将来の目標
- Policyを訓練して全自動化

---

## 比較対象

| OSS | リポジトリ | 開発元 |
|-----|-----------|--------|
| **GR00T-WBC** | このリポジトリ | NVIDIA |
| **TWIST2** | twist2_B3M-SC-1040-A | Yanjie Ze et al. |
| **UnifoLM-VLA** | unifolm-vla | Unitree Robotics |

---

## 制御の階層構造

```
┌─────────────────────────────────────────────────────────────┐
│  High-level Policy (意思決定・動作生成)                      │
│  ├── 「ゴミを見つけた → 拾う → 捨てる」                      │
│  ├── 視覚 + 言語 → 目標姿勢/軌道を生成                       │
│  └── UnifoLM-VLA が対応                                     │
├─────────────────────────────────────────────────────────────┤
│  Low-level Policy (動作追従・バランス維持)                   │
│  ├── 「目標姿勢に追従しながらバランスを取る」                 │
│  ├── 目標関節角度 → トルク指令                               │
│  └── GR00T-WBC / TWIST2 が対応                              │
└─────────────────────────────────────────────────────────────┘
```

**重要**: TWIST2の学習機能はLow-level Policy (motion tracking) のみ。High-level Policyの学習機能はない。

---

## 定性比較

### 基本情報

| 項目 | GR00T-WBC | TWIST2 | UnifoLM-VLA |
|------|-----------|--------|-------------|
| 開発元 | NVIDIA | Yanjie Ze et al. | Unitree Robotics |
| 主な用途 | 全身制御 | 全身テレオペ + 学習 | VLA (マニピュレーション) |
| ロボット | G1 | G1 | G1, ALOHA, Bridge等 |

---

### 今使えるPolicy

| 項目 | GR00T-WBC | TWIST2 | UnifoLM-VLA |
|------|-----------|--------|-------------|
| 事前学習済みPolicy | ✅ あり | ✅ あり | ✅ あり |
| 歩行 | ✅ walk, run, crawl | ✅ motion tracking | ❌ 対象外 |
| 階段・しゃがみ系 | ✅ squat, kneel, crawl (27+モード) | ⚠️ 訓練データ次第 | ❌ 対象外 |
| マニピュレーション | ✅ IK + テレオペ | ✅ テレオペ | ✅ 12タスク学習済み |

#### GR00T-WBC 利用可能モーション (GEAR-SONIC)

```
Locomotion: idle, slowWalk, walk, run
Ground: squat, kneelTwoLeg, kneelOneLeg, lyingFacedown, handCrawling, elbowCrawling
Boxing: idleBoxing, walkBoxing, leftJab, rightJab, randomPunches, leftHook, rightHook
Styled: happy, stealth, injured, careful, objectCarrying, crouch, happyDance, zombie, point, scared
```

---

### ハードウェア対応

| 項目 | GR00T-WBC | TWIST2 | UnifoLM-VLA |
|------|-----------|--------|-------------|
| G1本体 | ✅ | ✅ | ✅ |
| Amazing Hand | ❌ 非対応 | ✅ 対応済み | ⚠️ 汎用グリッパー想定 |
| Unitree 3指ハンド | ✅ 対応 | ⚠️ 未確認 | ⚠️ 未確認 |
| VRテレオペ (Pico) | ✅ | ✅ | ❌ サーバー側のみ |

---

### データ収集

| 項目 | GR00T-WBC | TWIST2 | UnifoLM-VLA |
|------|-----------|--------|-------------|
| エピソード記録機能 | ✅ あり | ✅ あり | ❌ なし (変換のみ) |
| 出力フォーマット | LeRobot | 独自JSON + 画像 | RLDS (入力のみ) |
| マルチモーダル | ✅ 画像 + 状態 + アクション | ✅ 画像 + 状態 + アクション | - |

---

### 学習機能

| 項目 | GR00T-WBC | TWIST2 | UnifoLM-VLA |
|------|-----------|--------|-------------|
| 学習コード | ❌ Coming Soon | ✅ あり | ✅ あり |
| 学習対象 | - | Low-level (motion tracking) | High-level (VLA) |
| フレームワーク | - | IsaacGym + RSL-RL | DeepSpeed + PyTorch |
| 必要GPU | - | 1 GPU | 8+ GPU |
| 訓練データ | - | 19,426モーション (OMOMO) | 12 G1タスク |

---

### セットアップ複雑度

| 項目 | GR00T-WBC | TWIST2 | UnifoLM-VLA |
|------|-----------|--------|-------------|
| Python環境 | 3.10 | 3.8 + 3.10 (2環境必要) | 3.10 |
| 主要依存 | MuJoCo, Unitree SDK2 | IsaacGym, Redis, Unitree SDK2 | DeepSpeed, TensorFlow, PyTorch |
| テレオペ実行 | 中程度 | 中程度 | 高い (収集機能なし) |
| 学習実行 | - | 中〜高 (IsaacGym) | 高い (8 GPU, データ変換) |

---

### カバー範囲

| 機能 | GR00T-WBC | TWIST2 | UnifoLM-VLA |
|------|-----------|--------|-------------|
| テレオペ | ✅ | ✅ | ❌ |
| データ収集 | ✅ | ✅ | ❌ |
| Low-level学習 | ❌ | ✅ | ❌ |
| High-level学習 | ❌ | ❌ | ✅ |
| デプロイ | ✅ | ✅ | ✅ |

---

## 必要な追加作業

### Amazing Hand対応

| OSS | 必要な作業 |
|-----|-----------|
| **GR00T-WBC** | AmazingHandControllerの移植、IKソルバー修正 (7DOF→8DOF)、テレオペ統合 |
| **TWIST2** | なし (対応済み) |
| **UnifoLM-VLA** | データ収集は外部ツール必要、アクション次元の設定変更 |

### 階段歩行

| OSS | 必要な作業 |
|-----|-----------|
| **GR00T-WBC** | GEAR-SONICの既存モード (squat, crawl等) で対応可能か検証 |
| **TWIST2** | 階段環境の構築、訓練データの準備、Policy学習 |
| **UnifoLM-VLA** | スコープ外 (歩行は対象としていない) |

### 将来の自動化

| OSS | 必要な作業 |
|-----|-----------|
| **GR00T-WBC** | 学習コード公開待ち |
| **TWIST2** | Low-level Policyは学習可能、High-levelは別途必要 |
| **UnifoLM-VLA** | High-level VLA学習可能、Low-levelは別途必要 |

---

## 各OSSの特徴まとめ

| OSS | 強み | 弱み |
|-----|------|------|
| **GR00T-WBC** | 豊富な事前学習モーション (27+)、NVIDIA公式、将来の学習パイプライン期待 | 学習コード未公開、Amazing Hand非対応 |
| **TWIST2** | Amazing Hand対応済み、Low-level学習可能、フルパイプライン | 2環境必要、階段用の訓練環境は自作 |
| **UnifoLM-VLA** | High-level VLA学習可能、言語指示で汎化 | 歩行非対応、8 GPU必要、データ収集機能なし |

---

## 技術詳細

### GR00T-WBC ハンド実装

**現在の実装**: `decoupled_wbc/control/envs/g1/g1_hand.py`
- Unitree 3指ハンド専用 (7 DOF)
- Unitree SDK2経由で通信
- `G1ThreeFingerHand` クラス

### TWIST2 Amazing Hand実装

**実装場所**: `deploy_real/robot_control/amazing_hand_wrapper.py`
- 16サーボモーター (8 DOF × 2手)
- シリアル通信 (SCS servo)
- `AmazingHandController` クラス
- 右手: ID 1-8、左手: ID 11-18

### UnifoLM-VLA アーキテクチャ

```
Vision-Language Interface (Qwen2.5-VL)
    ↓
    Processes visual observations + language instructions
    ↓
Action Model (DiT Transformer + Flow Matching)
    ↓
    Predicts action chunks (25 frames for G1)
```

---

## 参考リンク

- **GR00T-WBC ドキュメント**: https://nvlabs.github.io/GR00T-WholeBodyControl/
- **GEAR-SONIC モデル**: https://huggingface.co/nvidia/GEAR-SONIC
- **UnifoLM-VLA モデル**: https://huggingface.co/unitreerobotics/Unifolm-VLA-Base
- **G1データセット**: https://huggingface.co/collections/unitreerobotics/g1-dex1-datasets-68bae98bf0a26d617f9983ab

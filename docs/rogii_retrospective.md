# ROGII Wellbore Geology Prediction — 振り返り (Post-Mortem)

**コンペ**: [rogii-wellbore-geology-prediction](https://www.kaggle.com/competitions/rogii-wellbore-geology-prediction)（賞金 $50k, 締切 2026-08-05 23:59 UTC）
**タスク**: 水平坑井の toe（末端）マスク部の TVT（True Vertical Thickness = 層序上の位置）を予測
**評価指標**: Mean Squared Error（MSE）※RMSE ではない
**参加規模**: 6191 チーム

---

## TL;DR

- **最終結果: メダルなし。private 9.492 / rank ~3529 of 6191（上位 57%）**
- public 6.467（rank ~1157, 上位 19%）→ private 9.492（rank ~3529）へ **大シェイクダウン**
- 敗因: 「公開notebookをフォーク＋定数キャリブレーション」戦略が、このコンペの **隠しテスト構造（可視ウェル ≠ 採点ウェル）と public/private 乖離** に根本から不適合だった
- 一言でいえば: **public LB を信じてチューニングしたが、public は private を全く代表していなかった**

---

## 1. 結果サマリ

| 指標 | 値 |
|---|---|
| 自前ベスト public | **6.467** (`rogii-hedge-robust-contact-gated-base`, blacklions/contact-gated 無改変フォーク) |
| 自前 private（最終） | **9.492** |
| 最終順位 | **~3529 / 6191（上位 57%）** |
| public 順位（締切時点） | ~1157（上位 19%） |

### メダルボーダー（private, 6191チーム）
| メダル | rank | score |
|---|---|---|
| 🥇 gold | ~22 | ≤ 5.518 |
| 🥈 silver | ~310 | ≤ 6.343 |
| 🥉 bronze | ~619 | ≤ **6.396** |
| （自前） | ~3529 | 9.492 |

private top5 = 4.608 / 4.901 / 4.902 / 5.017 / 5.078

---

## 2. やったこと（タイムライン）

| 時期 | 内容 | 結果 |
|---|---|---|
| 6/17〜6/23 | 自作パイプライン（Formation KNN, Kriging, romantamrazov fork 等） | LB 10.3〜14 |
| 〜7/19 | lb7159 系フォーク路線へ転換 | 7.889 |
| 7/28〜7/30 | blacklions/contact-gated 系フォーク = **base 6.467** 確立。per-well 定数シフト検討開始 | base 6.467 が自前ベスト |
| 7/30 | aggressive(+1) = 6.512, det-mha = 6.906 → 正シフトは悪化と判断、一旦停止 | — |
| 8/1 | キャリブレーション再開。base-fork で w1+1 → 6.467 を観測（当時「最適+0.5でbronze」と誤読） | 提出方式の問題に直面 |
| 8/4 | LB激変を発見（フロンティア 6.4→4.6）。ボーダー再計算、bronze候補[+0.5]投入 | — |
| 8/5 | **全キャリブレーションが 6.467（ノーオペ）と判明**。グローバルシフトも無効。base確定 | — |
| 8/6 | 締切・private確定 → **9.492, rank~3529, メダルなし** | 終了 |

---

## 3. 技術的に突き止めた重要事実

### (A) このコンペは「隠しテスト再実行」型の code competition
- 開発中に見える test は **3ウェルのみ**（`00e12e8b` 4301行 / `000d7d20` 3836行 / `00bbac68` 6014行 = 計 14151 行）。
- しかし **採点時は別の隠しウェル群に差し替えてカーネルを再実行** する。
- 証拠: base の 14151 行出力を静的にコピーして提出 → `"wrong number of rows"` でフォーマットエラー却下（採点 sample_submission は 14151 と別物）。

### (B) per-well ID 指定シフトは採点上「完全ノーオペ」
- base-fork で **w1+0.5, w1+1, w2+1 が全て きっかり同じ 6.467**。
- 理由: シフトセルは `id[:8] == "00e12e8b"` 等で判定するが、**隠しウェルは ID が違うのでマッチ 0 行 → 無変化**。
- 人気公開 notebook（`zhexinjiang/rogii-shift-275`, 84票）も同一パイプライン＋同じ `_TGT` ウェルシフトで、**実は採点上 placebo**。
- グローバルシフト（全行一律 +0.5）も試したが → 6.467（base の内部 heel キャリブレーションで全体バイアスは既に除去済み）。

### (C) base は既に「汎化する正しいキャリブレーション」を内蔵
- base（contact-gated パイプライン）は各ウェルの **heel（既知部分）で GR の gain/offset を合わせる data 駆動キャリブレーション**を内包（cell 3/11/52）。
- つまり隠しウェルにも効く補正は既に適用済み。後付けの定数シフトが入る余地は無かった。

### (D) その他の運用知見（再利用可能）
- **KGAT トークンで Kaggle CLI を使う**: 環境変数 `KAGGLE_API_TOKEN=<KGAT>` を立てると CLI が Bearer 認証になり 401 を回避（デフォルトは Basic 認証で失敗）。
- **code-comp への提出**: `kaggle competitions submit <slug> -k <owner>/<kernel> -f submission.csv`。静的CSVアップロードは不可、カーネル出力のみ。
- **採点はカーネル再実行**。よって静的 echo や、別カーネル出力を `kernelDataSources` でチェインする方式は採点で壊れる（要 self-contained なフォーク）。
- **base 実行が不安定**: `COMPETITION_DATA_ROOT` が `/kaggle/input/competitions/<slug>` にハードコードだが、Kaggle は `/kaggle/input/<slug>` にマウントする揺れがあり、`train/*` グロブが空 → `KeyError:'wid'`。両パス＋glob で検出するパッチで安定化。
- **制約**: 採点ラグ ~8時間 / GPU 同時実行 最大2 / 提出 5件/日。

---

## 4. 敗因分析

**public 6.467 → private 9.492 の大乖離が全て。**

1. **可視3ウェルは訓練に近い「簡単なウェル」**で、public はそこで測られていた。採点の隠しウェル群は難しく、base パイプラインがそこに **汎化しなかった**（6.467 → 9.492）。
2. 従ってキャリブレーション路線は **二重に無意味** だった:
   - ① ID 指定シフトは隠しウェルにマッチせずノーオペ
   - ② 仮に public を改善できても、private は別ウェルなので効かない
3. **メダル上位（private 4.6〜6.4）は、可視ウェルに過学習しない本質的に強い手法**を持っていた。それは **非公開**（公開 notebook は 6.4帯止まり）で、締切直前にフォークでは到達不能だった。

---

## 5. 教訓（次コンペへ）

1. **public/private 乖離が大きいコンペでは public LB チューニングは罠**。序盤に public↔手元CV の相関を確認し、乖離が疑われたら **GroupKFold 等の手元CV / heel 等の内部検証を信頼** する。今回は public を信じてしまった。
2. **「可視テスト = 採点テスト」と決めつけない**。code competition は隠しテスト再実行があり、静的 echo や ID 指定の後処理は無効化されうる。**採点セットの規模・構造を早期に検証**する（例: 極端なシフトを1発投げて反応を見る）。
3. **モノカルチャーのフォーク＋後処理では上位は取れない**。メダルには **汎化する自作の芯**（このコンペなら地層/ANCC の空間補間、heel ベースの汎化補正、TVT 恒等式の活用など）を早期に持つべきだった。
4. **序盤の EDA・CV設計・「何が private で効くか」の見極め** に時間を使う。後半のLB微調整より、前半の設計が効く。

---

## 6. もし次にこの種のコンペをやるなら（打ち手案）

- **CV 設計を最優先**: GroupKFold(well) で「未知ウェル」への汎化を測る CV を組み、public はサニティチェック程度に。
- **物理の芯**: `TVT ≈ ANCC − Z + const`（訓練 RMSE ~0.007）を軸に、**773訓練ウェルからの ANCC 空間補間を高精度化**（GP/Kriging or 空間GBDT）。これは GR マッチ軌道と独立な情報経路で、真の多様性アンサンブルになる。
- **toe 残差補正**: 訓練ウェルの toe 正解から「heel からの距離・不確実性」に対する系統誤差を学習し、テストへ適用（ID 非依存＝汎化する）。
- **提出は self-contained なフォーク**で、隠しテスト再実行に耐える形にする。

---

*作成: 2026-08-06 / 詳細ログは `~/.claude/.../memory/project_rogii_diary.md` にも保持*

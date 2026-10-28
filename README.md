# 新規環境の構築用ファイル (CLAUDE.md / SKILLS.md)

新しい研究プロジェクトを始めるとき、本プロジェクトと同じフォルダ構造と表画像の様式を再現するための指示書です。
Claude Code に読み込ませることで、フォルダ構成の作成と、総括表の作成を同じ規則で行えます。

| ファイル | 内容 | 読むタイミング |
|---|---|---|
| [CLAUDE.md](../../CLAUDE.md) | フォルダ構造、各フォルダの役割、環境構築の手順、作業・返答のルール | セッション開始時に自動で読み込まれる |
| [SKILLS.md](../../SKILLS.md) | 表画像 (PNG) と数値CSVの作成手順、描画コード | 表を作るときに、CLAUDE.md の案内から参照される |

## 再現されるフォルダ構造

プロジェクト固有のフォルダは `<package_name>/` のみで、それ以外はどのプロジェクトでも同じ名前にします。

```
<project_root>/
  CLAUDE.md
  SKILLS.md
  README.md
  requirements.txt
  <package_name>/      プロジェクト固有のライブラリ (本プロジェクトでは anticloning_mask)
  configs/             実験設定 (YAML)
  docs/                詳細ドキュメント・図
  outputs/             実験結果・表・ログ
  scripts/             実験・評価・表作成の実行スクリプト
  tools/               補助ツール
```

各フォルダの役割、`outputs/` の内訳、`scripts/` の命名規則は [CLAUDE.md](../../CLAUDE.md) に書いてあります。

## 使い方

1. 新しいプロジェクトのルートに、CLAUDE.md と SKILLS.md をコピーします。
2. ルートで Claude Code を起動し、次のように依頼します。

   ```
   CLAUDE.md に従って、パッケージ名 <package_name> でフォルダ構造を作成して
   ```

3. CLAUDE.md の「環境構築の手順」に沿って、次が作られます。
   - フォルダ: `<package_name>/`, `configs/`, `docs/figures/`, `outputs/tables/`, `outputs/logs/`, `scripts/`, `tools/`
   - ファイル: `<package_name>/__init__.py`、`README.md`、`requirements.txt`、`outputs/README.md`、`configs/default.yaml`
4. 表を作るときは「SKILLS.md に従って表を作って」と依頼します。表の内容 (列・行・グループ分け) とタイトルを確認してから、`scripts/make_summary_tables.py` に関数を追加して実行します。

手動で作る場合は、CLAUDE.md の「環境構築の手順」にある `mkdir` コマンドをそのまま実行できます。

## 表画像の規則 (SKILLS.md の要点)

- 画像に入れるのは、簡潔なタイトルと表本体だけです。注釈、凡例、脚注、条件の説明は入れません。必要な説明は `docs/` や `outputs/README.md` に書きます。
- 出力先は `outputs/tables/` で、`NN_<名前>.png` と `NN_<名前>.csv` を対で保存します (`NN` は2桁の通し番号)。
- 配色はグレースケールで、同じ値が連続するセルは縦に結合し、グループの境界には太罫線を引きます。
- 描画に必要なパッケージは `matplotlib`, `numpy`, `pandas`, `japanize-matplotlib` です。
- 画像の見た目は Claude Code に撮影・確認させず、利用者が確認します。

## 本プロジェクトとの差異

- `tools/` は本プロジェクトには存在しません。補助物は `demo_site/` に置かれています。新しい環境では `tools/` に統一する前提で CLAUDE.md を書いています。
- 既存の表 (`outputs/tables/` の `02_speaker_summary.png` など) は、凡例や長いタイトルを含む旧様式です。SKILLS.md の規則で作られるのは新しい表からで、既存の表は変更していません。
- SKILLS.md は Claude Code が自動で呼び出すスキル (`.claude/skills/<名前>/SKILL.md`) の形式ではありません。CLAUDE.md の案内を通じて参照される、通常の Markdown です。

## 更新するときの注意

- フォルダ構造を変えるときは、CLAUDE.md の「フォルダ構造」「各フォルダの役割」「環境構築の手順」を同時に直します。ずれると、新しい環境で構造が再現されません。
- 表の様式を変えるときは、SKILLS.md の「見た目の仕様」と、埋め込みの `draw_table` のコードを同時に直します。
- CLAUDE.md の「返答に関するルール」「作業についてのルール」は、利用者の個人設定に基づく内容です。共同研究者へ渡すときは、この2節を削除または差し替えてください。

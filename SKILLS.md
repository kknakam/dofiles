# SKILLS.md

## 表画像の作成

実験結果のCSVから、`outputs/tables/` に「結合セル付き・グレースケールの表画像 (PNG)」と「数値CSV」を作る手順。
既存の表 (`outputs/tables/` の `02_speaker_summary.png` など) と同じ見た目にそろえる。

### 画像に入れるもの

| 入れる | 入れない |
|---|---|
| タイトル (簡潔に) | 注釈・凡例・脚注 (`note`) |
| 表本体 (ヘッダー行 + データ行) | 条件の説明文 (発話数、ステップ数、ターゲット話者など) |
| | 記号の意味の説明 (`( )` は白箱、など) |

- 説明が必要な内容は、画像ではなく `docs/` や `outputs/README.md` に書く。
- 凡例が必要になる記号は使わない。列名とセルの値だけで意味が分かるようにする。
- タイトルは名詞句で、全角20文字程度まで。括弧書きの補足や「〜ほど良い」といった読み方の説明は付けない。
  - 良い例: `損失アブレーション` / `保護の強さと音質`
  - 悪い例: `エンコーダ多様性: 保護音声の音質と、判定器ごとの cos (保護音声↔元音声, 低いほど話者がずれた)`

### 出力ファイル

| 項目 | 規則 |
|---|---|
| 保存先 | `outputs/tables/` |
| ファイル名 | `NN_<名前>.png` と `NN_<名前>.csv`。`NN` は2桁の通し番号 (既存の最大番号 + 1)。分割する場合は `NN_<名前>_<k>_<区分>.{csv,png}` |
| CSV | `encoding="utf-8-sig"`, `float_format="%.4f"`, `index=False`。列名は画像のヘッダーと対応させる |
| 画像 | `dpi=160`, `bbox_inches="tight"` |

### 見た目の仕様

| 要素 | 仕様 |
|---|---|
| 配色 | グレースケールのみ。ヘッダー背景 `#333333` (白文字・太字)、データ行は白、グループを区別する行は `#F0F0F0` |
| セル枠 | 色 `#999999`、線幅 0.6 |
| グループ境界 | 黒の太罫線 (線幅 2.0)。グループ = 手法・条件など意味のまとまり |
| 細罫線 | グループ内の区切り。色 `#666666`、線幅 1.0 |
| 結合セル | `merge_cols` に指定した列の、同じグループ内で連続する同一値を縦に結合する |
| 欠損値 | `—` |
| タイトル | 14pt・太字 |
| 本文 | 11pt。ヘッダーは 10pt。ヘッダーの改行は `\n` で行う |
| 列幅 | `widths` に相対値で指定する。文字数に応じて 0.6 〜 2.4 程度 |
| 日本語 | `japanize_matplotlib` を import する |

数値の桁数 (`num_fmt`): 列名に `%` を含む列は小数1桁、`PESQ` / `UTMOS` / `STOI` は小数3桁、`cos` を含む列は小数4桁、`SNR` は小数1桁。指標が増えたら `num_fmt` に足す。

### 実装

`scripts/make_summary_tables.py` に置く。ルート `ROOT` は `outputs/`。

```python
from pathlib import Path

import japanize_matplotlib  # noqa: F401
import matplotlib.pyplot as plt
import numpy as np
import pandas as pd

ROOT = Path(__file__).resolve().parent.parent / "outputs"
OUT_DIR = ROOT / "tables"
OUT_DIR.mkdir(parents=True, exist_ok=True)


def read(path):
    return pd.read_csv(ROOT / path, encoding="utf-8-sig")


def compute_spans(vals):
    """連続する同一値のスパン (start, end_inclusive) を返す"""
    spans, i = [], 0
    while i < len(vals):
        j = i + 1
        while j < len(vals) and (vals[j] == vals[i] or
                                 (isinstance(vals[i], float) and np.isnan(vals[i]) and
                                  isinstance(vals[j], float) and np.isnan(vals[j]))):
            j += 1
        spans.append((i, j - 1))
        i = j
    return spans


def draw_table(df, cols, headers, widths, title, path, merge_cols=(), groups=None,
               row_bg=None, fmt=None, bold=None, thin_lines=(), fontsize=11):
    """df[cols] を結合セル付きの表画像にする。画像に入れるのはタイトルと表本体だけ。

    merge_cols: 連続する同一値を縦に結合する列。groups: [(start, end), ...] 結合はこの範囲を越えない
    (グループ境界には太罫線)。row_bg(i) -> 背景色。fmt(val, col) -> 文字列。
    bold(val, col) -> 太字にするか。thin_lines: 細罫線を引く行番号 (その行の上)。
    """
    n_rows, n_cols = len(df), len(cols)
    groups = groups or [(0, n_rows - 1)]
    row_bg = row_bg or (lambda i: "white")
    fmt = fmt or (lambda v, c: "—" if isinstance(v, float) and np.isnan(v) else str(v))
    bold = bold or (lambda v, c: False)

    col_spans = {}
    for c in cols:
        if c in merge_cols:
            spans = []
            for g0, g1 in groups:
                for s, e in compute_spans(df[c].iloc[g0:g1 + 1].tolist()):
                    spans.append((g0 + s, g0 + e))
            col_spans[c] = spans
        else:
            col_spans[c] = [(i, i) for i in range(n_rows)]

    total_w = sum(widths)
    col_w = [w / total_w for w in widths]
    xs = np.cumsum([0.0] + col_w[:-1])
    row_h = 1.0 / (n_rows + 1.8)
    fig_w = total_w * 0.90 + 0.3
    fig_h = (n_rows + 1) * 0.55 + 0.6

    fig, ax = plt.subplots(figsize=(fig_w, fig_h))
    ax.set_xlim(0, 1)
    ax.set_ylim(0, 1)
    ax.axis("off")
    fig.suptitle(title, fontsize=14, fontweight="bold", y=0.995)

    def cell(x, y, w, h, text, bg="white", fg="black", is_bold=False, size=fontsize):
        ax.add_patch(plt.Rectangle((x, y), w, h, facecolor=bg, edgecolor="#999999",
                                   linewidth=0.6, transform=ax.transAxes, clip_on=False))
        ax.text(x + w / 2, y + h / 2, text, ha="center", va="center", fontsize=size, color=fg,
                fontweight="bold" if is_bold else "normal", transform=ax.transAxes,
                clip_on=False, multialignment="center")

    for lbl, x, w in zip(headers, xs, col_w):
        cell(x, 1.0 - row_h, w, row_h, lbl, bg="#333333", fg="white", is_bold=True,
             size=fontsize - 1)

    for ci, c in enumerate(cols):
        for s, e in col_spans[c]:
            val = df.iloc[s][c]
            cell(xs[ci], 1.0 - row_h * (e + 2), col_w[ci], row_h * (e - s + 1),
                 fmt(val, c), bg=row_bg(s), is_bold=bold(val, c))

    for g0, _ in groups[1:]:
        y = 1.0 - row_h * (g0 + 1)
        ax.plot([0, 1], [y, y], color="black", linewidth=2.0, transform=ax.transAxes,
                clip_on=False, zorder=10)
    for r in thin_lines:
        y = 1.0 - row_h * (r + 1)
        ax.plot([0, 1], [y, y], color="#666666", linewidth=1.0, transform=ax.transAxes,
                clip_on=False, zorder=9)

    plt.tight_layout(rect=[0, 0.01, 1, 0.98])
    fig.savefig(path, dpi=160, bbox_inches="tight")
    plt.close(fig)
    print(f"画像 保存: {path}")


def num_fmt(v, c):
    if isinstance(v, float) and np.isnan(v):
        return "—"
    if isinstance(v, (int, float, np.floating, np.integer)):
        if "%" in c:
            return f"{float(v):.1f}"
        if c in ("PESQ", "UTMOS", "STOI"):
            return f"{float(v):.3f}"
        if "cos" in c:
            return f"{float(v):.4f}"
        if "SNR" in c:
            return f"{float(v):.1f}"
        return f"{float(v):g}"
    return str(v)
```

### 表ごとの関数の書き方

1つの表を1つの関数にする。CSV保存 → `draw_table` の順で、`main` から順に呼ぶ。

```python
def loss_ablation():
    df = read("main/<実験群>/results/<結果>.csv")          # 1. 元のCSVを読む
    df = df[["条件", "手法", "PESQ", "STOI", "cos"]]         # 2. 載せる列だけに絞る (列名 = 画像のヘッダー)
    df.to_csv(OUT_DIR / "03_loss_ablation.csv", index=False,
              encoding="utf-8-sig", float_format="%.4f")    # 3. 数値CSVを保存
    draw_table(                                             # 4. 画像を保存
        df, list(df.columns),
        ["条件", "手法", "PESQ", "STOI", "cos\n(保護↔元)"],
        [1.9, 1.6, 0.9, 0.9, 1.2],
        "損失アブレーション",                                  # 簡潔なタイトル。note は渡さない
        OUT_DIR / "03_loss_ablation.png",
        merge_cols=("条件",),                                # 連続する同じ条件を縦に結合
        groups=[(0, 5), (6, 11)],                           # グループ境界 (太罫線)
        row_bg=lambda i: "white" if i < 6 else "#F0F0F0",   # グレースケールの行背景
        fmt=num_fmt,
    )


if __name__ == "__main__":
    loss_ablation()
```

### 作業の進め方

1. 表の内容 (列・行・グループ分け) を決め、通し番号 `NN` と簡潔なタイトルをユーザーに示す。
2. `scripts/make_summary_tables.py` に関数を追加し、`python scripts/make_summary_tables.py` で実行する。
3. 出力された CSV の行数・列数・欠損の有無をコマンドで確認する。
4. 画像の見た目は自分で撮影・確認せず、ユーザーに確認を依頼する。
5. 画像に注釈を入れたくなった場合は、その内容を `docs/` か `outputs/README.md` に書く。

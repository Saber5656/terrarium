# terrarium 設計書 (v1)

- Repository: `github.com/Saber5656/terrarium`
- Tagline: "A terminal pet terrarium where creatures grow when tests pass."
- License: MIT（前提）/ クラウド送信なし・完全ローカル / 個人 OSS・最小実装で早期リリース
- 作成日: 2026-07-05

---

## 1. コンセプトと既存 OSS との位置づけ

terrarium は、ターミナルの中に小さな水槽を飼う TUI トイである。
`terrarium run -- npm test` のように**普段のテストコマンドをラップするだけ**で、テストが通る（exit code 0）たびに餌が落ち、
生き物が育ち、水草が増える。世話のための専用コマンドを打たせるのではなく、
TDD の red-green ループそのものを餌やりに変換する。「テストを回すこと自体が正の体験になる」ことだけに集中する。

**既存の terminal pet 系 OSS との差別化（車輪の再発明にならない位置づけ）**

| 既存 | 育つトリガー | terrarium との違い |
|---|---|---|
| [usik/tamagotchi](https://github.com/usik/tamagotchi) | 時間経過 + 手動ケア（本家再現: 病気・poop・しつけ） | 世話が「別の作業」として増える。terrarium は**普段のテスト実行そのもの**が世話になり、専用の世話コマンドを持たない |
| [terminal-pet](https://github.com/apoorvgarg31/terminal-pet) | git commit | commit は成果の後追い。terrarium は red-green の**ループ最小単位（テスト成否）**に連動し、コミット前の試行錯誤が全部餌になる |
| [codachi](https://github.com/vincent-k2026/codachi) | Claude Code statusline 常駐、エディタイベント（test/build/commit） | 特定ツール（Claude Code）専用。terrarium はエディタ・言語・AI 非依存で、exit code 1 個だけを見る |
| [terminalgotchi](https://crates.io/crates/terminalgotchi) | `cargo test` / `git commit` で XP | Rust 開発者特化。terrarium は任意コマンドラップで言語非依存。個体でなく「水槽 = 環境ごと育つ」表現 |
| [asciiquarium](https://github.com/cmatsuoka/asciiquarium) | なし（鑑賞のみ） | 見た目の先祖。terrarium は自分の開発行動が水槽の豊かさに反映される |

つまり「**テスト結果と連動して育つ、言語非依存のローカル TDD のおとも**」がまだ空いているポジションであり、それが存在理由。
既存 tamagotchi 系が持つ重い育成要素（死・病気・義務としての世話）は意図的に輸入しない（§5 参照）。

---

## 2. v1 スコープ

| 区分 | 項目 | 備考 / 理由 |
|---|---|---|
| 入れる | `terrarium run -- <cmd>` | プロダクトの核。任意コマンドを実行し **exit code 0/非 0 だけ**を見る。成功で給餌ミニ演出 |
| 入れる | `terrarium`（水槽 TUI） | alternate screen で水槽を眺める。tick 駆動アニメ |
| 入れる | `terrarium status`（1 行サマリ） | シェルプロンプト・tmux status 組み込み用。TUI 初期化なしの高速パス |
| 入れる | 生き物 1 種（魚）× 4 成長段階 | Egg → Fry → Juvenile → Adult。ASCII/Unicode 数フレームアニメ |
| 入れる | 環境の成長（水草・岩・貝） | streak マイルストーンで水槽が豊かになる。個体だけでなく環境が育つのが terrarium の名の由来 |
| 入れる | 失敗で mood 低下（**死なない**）、24h 放置で空腹（回復可能） | §5 育成モデル参照 |
| 入れる | プロジェクト（cwd）ごとの状態分離 | JSON 保存。リポジトリごとに別の水槽 |
| 入れる | truecolor / 256 色 / 16 色フォールバック + `NO_COLOR` 尊重 | どの端末でも崩れない |
| 入れる | 非 TTY 検出 | パイプ・CI 上では演出を自動スキップし、exit code を素通しする実用ツールとしての品位 |
| 入れない | テストランナー固有のパース（JUnit XML / TAP / テスト件数） | exit code だけなら**全言語・全ランナー対応が実装ゼロ**で手に入る。パーサはランナー × バージョンの保守地獄。ただし Runner → GrowthEngine の境界にイベント型を挟み、将来 reporter を差せる口だけ残す |
| 入れない | CI 連携・GitHub 連携・バッジ | 「ローカル TDD のおとも」という核から外れる。完全ローカル原則（§7） |
| 入れない | git hook の自動設定 | ユーザーのリポジトリ設定を書き換えるのは侵襲的。README に alias / hook の自作レシピだけ書く |
| 入れない | watch モード / デーモン常駐 | v1.1 候補。都度実行で体験が成立する。常駐はバグ面積と電池消費を増やす |
| 入れない | 複数の生き物種・種の選択 | スプライト定義は複数種を許すデータ構造にするが、v1 は魚 1 種。可愛さの密度を 1 種に集中 |
| 入れない | ペット交換・共有・ランキング | ネットワーク機能は恒久的に持たない（§7） |
| 入れない | 設定ファイル | v1 はゼロコンフィグ。フラグと環境変数で足りる |
| 入れない | 効果音 | 端末ベルは事故のもと |

---

## 3. 対応プラットフォームと優先順位

| 優先度 | 環境 | 判断 | 理由 |
|---|---|---|---|
| 1 (v1) | macOS（Terminal.app / iTerm2 等） | フル対応 | 開発者の主環境。日常のドッグフードで品質を上げる |
| 1 (v1) | Linux（主要エミュレータ、tmux/screen 内含む） | フル対応 | crossterm でコストほぼゼロ。CI (ubuntu) での検証が最速。256 色フォールバックで tmux もカバー |
| 2 (v1) | Windows（**Windows Terminal のみ**） | ベストエフォート | crossterm は対応済みで CI ビルド・テストは回すが、実機での常用検証はしない。legacy conhost（旧コマンドプロンプト）は対象外と README に明記 |

TUI の描画自体は crossterm がプラットフォーム差を吸収するため 3 OS 対応はほぼタダで手に入る。
本当の OS 差リスクは `run` の**子プロセスラップ**（exit code の取り方、Ctrl-C の伝播、シグナル）にあり、§10 の P0 で検証する。

---

## 4. 技術選定

### 比較

| 候補 | 配布 | TUI アニメーション | 起動速度 | 判定 |
|---|---|---|---|---|
| **Rust + ratatui + crossterm（採用）** | 単一バイナリ。cargo / brew / Releases | immediate mode + tick 駆動が公式 recipe 化。効果ライブラリ (tachyonfx) が成立するほどエコシステムが厚い | 数十 ms 級。`status` のプロンプト組み込みに耐える | ✅ |
| Go + bubbletea | 単一バイナリ | Elm アーキテクチャ + tick。実績十分 | 速い | ◯ 成立するが、Rust 資産（開発者の主言語・既存プロジェクトとの共通化）を優先 |
| Node + ink | npm。ランタイム必須 | React 流 | 数百 ms〜 | ❌ 単一バイナリにならず、プロンプト組み込みに不向き |
| Python + textual | pip。ランタイム必須 | 高機能 | 遅い | ❌ 同上 |

### 採用スタック

| 層 | 技術 | 理由 |
|---|---|---|
| 言語 | Rust (stable) | 単一バイナリ・起動速度・クロスコンパイル |
| TUI | [ratatui](https://github.com/ratatui/ratatui) | デファクト。immediate mode で毎 tick 全再描画するだけでアニメになる |
| 端末制御 | [crossterm](https://github.com/crossterm-rs/crossterm) | ratatui 標準バックエンド。3 OS + raw mode + カラー検出 |
| CLI | clap (derive) | サブコマンド 3 つの薄い定義 |
| 状態保存 | serde + serde_json | 人間が読める・手で直せる。スキーマに version フィールド |
| パス解決 | [directories](https://crates.io/crates/directories) | XDG / Application Support / AppData を正しく解決 |
| 乱数 | fastrand | 魚の泳ぎの揺らぎ用。rand より軽量 |
| 依存方針 | ネットワーククレートを持たない | 「完全ローカル」を Cargo.toml の依存リストで検証可能にする（§7） |

### ratatui でのアニメーションの定石（調査結果）

- ratatui は immediate mode であり、**アプリ側がイベントループを持ち、tick イベントで再描画する**のが公式の定石。
  同期版（`event::poll` に tick_rate の残り時間を渡す）と async 版（`tokio::select!` で tick / render / input を分離）の両方が公式 recipe にある。
- エフェクトライブラリ [tachyonfx](https://github.com/ratatui/tachyonfx) が ratatui org 配下で成立しており、tick 駆動アニメの実績は十分。
  terrarium のアニメは 4〜8fps の低フレームで足りる（ドット絵水槽のカクカク感はむしろ味）ため、同期版イベントループで開始し、tokio は入れない。
- `run` 後の**インライン演出**（alternate screen を使わず、通常のスクロール領域に数行だけアニメを描いて残す/消す）には
  [`Viewport::Inline`](https://docs.rs/ratatui/latest/ratatui/enum.Viewport.html) が存在する。ただし端末互換性の実績が水槽 TUI ほど厚くないため、**P0 検証項目**とする（§10）。

---

## 5. アーキテクチャ

単一バイナリ・単一プロセス。常駐しない。3 つのサブコマンドが同じ StateStore を読み書きする。

```
┌────────────────────────────────────────────────────────────┐
│ terrarium (single binary)                                  │
│                                                            │
│  clap CLI                                                  │
│   ├─ terrarium             ──▶ TuiApp（水槽・alt screen）   │
│   ├─ terrarium run -- cmd  ──▶ Runner（子プロセス実行）      │
│   └─ terrarium status      ──▶ StatusLine（1 行出力）       │
│                                                            │
│  ┌──────────────┐  RunOutcome   ┌──────────────────┐       │
│  │ Runner        │─────────────▶│ GrowthEngine     │       │
│  │ (spawn child, │ (exit code   │ (餌 / mood /     │       │
│  │  stdio 素通し) │  のみ。出力は │  streak / 成長)  │       │
│  └──────────────┘  読まない)    └────────┬─────────┘       │
│                                         │ load / save     │
│  ┌──────────────┐               ┌───────▼─────────┐       │
│  │ TuiApp        │◀──────────────│ StateStore      │       │
│  │ (tick loop +  │    state     │ (JSON, atomic   │       │
│  │  FeedAnim)    │               │  write, project │       │
│  └──────────────┘               │  ごとに 1 ファイル)│       │
│                                 └─────────────────┘       │
└────────────────────────────────────────────────────────────┘
```

### 状態の保存場所

| OS | パス（[directories](https://crates.io/crates/directories) の data_dir） |
|---|---|
| Linux | `~/.local/share/terrarium/projects/<hash>.json`（XDG 準拠） |
| macOS | `~/Library/Application Support/terrarium/projects/<hash>.json` |
| Windows | `%APPDATA%\terrarium\data\projects\<hash>.json` |

- **project root の決定**: cwd から上方向に `.git` を探し、見つかればそこを root、なければ cwd。
  サブディレクトリでテストを打っても同じ水槽に餌が届く（git2 等のライブラリは使わず、ディレクトリ探索数行で済ませる）
- `<hash>` = project root 絶対パスの SHA-256 先頭 16 hex。JSON 内に元パスも保持（表示・棚卸し用）
- 書き込みは**テンポラリファイル + rename の atomic write**。2 端末同時実行は last-write-wins と割り切る（餌 1 個の消失はゲーム上許容。ロックは持たない）
- 破損時（parse 失敗）は `<hash>.json.corrupt-<ts>` へ退避して新規作成し、警告 1 行。育てた記録を黙って消さない

### 状態モデル

```rust
struct TerrariumState {
    version: u32,                    // スキーマ進化用。未知 version は触らずエラー
    project_path: String,
    creature: Creature {
        stage: Egg | Fry | Juvenile | Adult,
        growth: u32,                 // 累計成長ポイント。減らない（不可逆）
        mood: Happy | Hungry | Sulking,
        last_fed: u64,               // unix time。空腹判定・満腹クールダウンに使用
    },
    environment: { plants: u8, rocks: u8, shells: u8 },  // マイルストーン到達で増える
    stats: { total_runs, total_passes, total_fails, current_streak, best_streak },
}
```

### 育成モデル（処理フロー: `terrarium run`）

1. `--` 以降を子プロセスとして spawn。**stdin/stdout/stderr は inherit（素通し）**。色もインタラクションも壊さない
2. exit code を取得（terrarium が見るのはこの整数 1 個だけ）
3. state をロード（なければ Egg で初期化）
4. GrowthEngine が下表を適用して保存
5. TTY なら 2〜3 秒の給餌ミニ演出 + 1 行サマリ（`--quiet` で 1 行のみ）。非 TTY なら何も出さない
6. **子プロセスの exit code をそのまま伝播**して終了（`&&` 連結・スクリプト・CI に対して透明）

| イベント | 効果 |
|---|---|
| run 成功（exit 0） | 餌 1 → `growth +10`、mood 回復（→ Happy）、streak +1 |
| run 成功（ただし前回成功から 60 秒未満） | **満腹**: growth 加算なし・mood 回復のみ。`terrarium run -- true` 連打対策。terrarium はゲームであって監査ツールではないので、チート対策はこれ以上厳密化しない |
| run 失敗（非 0） | mood 1 段階低下（Happy → Sulking。最低でも Sulking 止まり）。**growth と streak 済み実績は減らない**。streak は 0 にリセット |
| シグナル/Ctrl-C による中断 | **ニュートラル**（餌なし・mood 変化なし）。テストを止めたのは失敗ではない（Windows では判別不能のため非 0 = 失敗として扱う） |
| 24h 以上 run なし | mood → Hungry（空腹表示）。成功 1 回で即回復 |
| streak マイルストーン（5 / 10 / 25 / 50…） | 水草 → 岩 → 貝 → … と環境が増える（数値は実装時にチューニング） |

**「死なない・ギスギスしない」を仕様として明記する理由**:
失敗もテストを回した証拠であり、罰を重くするとテストを回すこと自体を避けるようになる（プロダクトの目的と真逆）。
減るのは見た目の元気（mood）だけで、育った分（growth・環境）は**不可逆に積み上がる**。
red-green のリズムでは red は正常な工程なので、「しょんぼり」演出も心配そうに見守るトーンに留める。

### 成長段階と状態遷移

```
       最初の緑で即孵化    growth ≥ 250       growth ≥ 750
  Egg ─────────────▶ Fry ─────────────▶ Juvenile ─────────▶ Adult

           24h 放置              緑 1 回
  Happy ──────────▶ Hungry ──────────▶ Happy
    │ 赤（テスト失敗）                    ▲
    └─────────▶ Sulking ────────────────┘ 緑 1 回
```

---

## 6. UI/UX

### 水槽 TUI（`terrarium`、alternate screen）

```
╭─ terrarium ─ my-project ─────────────────────────────╮
│ ~   ~    ~     ~    ~    ~     ~    ~    ~    ~   ~ │
│                    °                                │
│        ><((º>      o                                │
│                    .           \|/                  │
│   ,,,,        __         ,,,   \|/      @          │
│ wwWWwwwWWWwwwww(__)wwwWWwwwwwwWWWwwwwwWWWwwwwwwWWww │
╰─ Adult · streak 12 · fed 2h ago ──────── q: quit ───╯
```

| 要素 | アニメーション | 周期 |
|---|---|---|
| 魚 | 2 フレーム（尾びれ開閉）+ ランダムウォーク。進行方向でスプライト左右反転（`><((º>` / `<º))><`）。端で反転 | tick 250ms (4fps) |
| 泡 | `.` → `o` → `°` と 1 列上昇して水面で消える | tick ごと 1 セル |
| 水草 `\|/` | 2 フレームで左右に揺れる | 1s |
| mood 表現 | Happy: 水槽全体を泳ぎ回り泡が多い / Hungry: 水面近くを往復（餌を待つ）/ Sulking: 底でほぼ静止、動きが遅い | — |

- tick 駆動の同期イベントループ（`event::poll(tick 残り時間)` 方式、§4）。キー入力は `q` / `Esc` で終了のみ
- カラー: `COLORTERM` で truecolor 検出 → 水のグラデーション。フォールバックは 256 色 → 16 色の 3 段階。`NO_COLOR` でモノクロ
- リサイズ追従。幅 40 col 未満では簡易表示（魚 + ステータス行のみ）に自動縮退
- panic hook で raw mode / alternate screen を必ず復元（端末を壊して終了しない）

### `terrarium run` の給餌演出（inline、alternate screen を使わない）

```
$ terrarium run -- cargo test
   ...（cargo test の出力がそのまま流れる。色付き・インタラクティブもそのまま）...
   Passed!  °  。
    ><((º>    ∴     ← 餌が沈み、魚が食べに来る 2〜3 秒のミニアニメ（3 行）
  (( terrarium )) growth +10 · streak 13 · Adult
$
```

- 成功: 餌が落ち魚が食べる演出 + 1 行サマリ。失敗: 演出なしで `(( terrarium )) no food this time · the fish waits patiently` の 1 行のみ（**責めない文言**）
- 非 TTY（パイプ・CI・リダイレクト）では演出もサマリも出さず、子プロセスの出力と exit code だけが流れる
- `--quiet` でアニメ抑制（1 行サマリのみ）

### `terrarium status`（プロンプト / tmux 組み込み用）

```
$ terrarium status
><((º> adult · streak 12 · fed 2h ago
```

- TUI 初期化なし・state 読むだけの高速パス。**起動〜出力 50ms 未満**を性能要件とする（§10 P1）
- state がないプロジェクトでは何も出さず exit 0（プロンプトを汚さない）

### 初回起動体験

1. どのコマンドでも初回は卵が生まれる。`terrarium` を開くと水底に卵が 1 個沈んでいて、ヒントが 1 行:
   `Run  terrarium run -- <your test command>  to hatch your egg.`（チュートリアルはこれで終わり）
2. **最初の緑で即孵化**。導入 → 最初のご褒美までを「いつものテスト 1 回」に最短化する（Egg の必要 growth は 0 → Fry）
3. 孵化演出は少しだけ豪華に（ここが SNS でシェアされる瞬間 = デモ GIF の素材）

### エッジケースの割り切り（v1）

| ケース | 挙動 |
|---|---|
| 子プロセスが TUI / watch モード / 色付き出力 | stdio inherit なのでそのまま動く。watch を Ctrl-C で抜けた場合は中断＝ニュートラル |
| `--` なしで `terrarium run npm test` | そのまま `npm test` として解釈（`--` は clap の慣例に従い任意） |
| 2 端末で同時に run | atomic write の last-write-wins（§5）。餌 1 個の消失は許容 |
| 幅が極端に狭い / 非 UTF-8 ロケール | 簡易表示 / ASCII のみのスプライトに縮退 |
| state 破損 | 退避 + 再生成 + 警告 1 行（§5） |

---

## 7. プライバシー

原則: **「exit code より内側を見ない」**。完全ローカルで、ネットワークに一切触れない。

| 項目 | 方針 |
|---|---|
| 読むもの | 子プロセスの **exit code（整数 1 個）のみ**。stdout/stderr は端末へ inherit で素通しし、terrarium はパイプも tee も**しない** — テスト出力の内容（失敗メッセージ・ファイル名・件数）は構造的に読めない設計 |
| 記録するもの | project root のパス、カウンタ（runs/passes/fails/streak）、growth、タイムスタンプのみ。**実行したコマンド文字列は保存しない**（`npm test -- --grep secret` のような引数がディスクに残らない） |
| 送信するもの | なし。テレメトリなし・アップデート確認なし・クラッシュレポートなし。依存にネットワーククレートを持たないことで `Cargo.toml` を見るだけで検証できる状態を保つ |
| 権限 | 昇格不要。書き込みは自分の data dir 1 箇所のみ。ユーザーのリポジトリ・git 設定には一切書き込まない |
| README での宣言 | Privacy 節に "terrarium reads exactly one integer from your tests: the exit code." と明記（§9） |

---

## 8. 配布方法

| 手段 | 内容 | 理由 |
|---|---|---|
| crates.io | `cargo install terrarium` — **crate 名 `terrarium` は 2026-07-05 時点で未登録であることを crates.io API で確認済み**。最初のリリースで確保する | Rust ユーザーへの最短経路。単一バイナリなので cargo install が素直に機能する |
| GitHub Releases | [cargo-dist](https://github.com/axodotdev/cargo-dist) で 3 OS（macOS universal / Linux x86_64・aarch64 / Windows）のビルド済みバイナリ + インストーラスクリプトを tag push で自動生成 | Rust 環境がない人向け。手書き Actions より運用が軽い |
| Homebrew | 自前 tap: `brew install Saber5656/tap/terrarium`（[Taps](https://docs.brew.sh/Taps)） | README の 1 行インストール用。本家 homebrew-core は知名度要件があるためまず tap |
| 後回し | AUR / scoop / winget / mise | 反応を見てから。個人 OSS の初手でパッケージマネージャを増やしすぎない |

---

## 9. README 構成案（英語）

```markdown
<バナー画像: ドット絵の水槽にロゴ。横長 PNG>

# terrarium 🐠
> A terminal pet terrarium where creatures grow when tests pass.

<デモ GIF: vhs で録画。`terrarium run -- cargo test` が緑 → 餌が落ちて魚が食べる →
 `terrarium` で水槽を眺める、の 10 秒ループ>

[badges: CI / crates.io version / license — 3 つまで]

## Install
    cargo install terrarium
    # or
    brew install Saber5656/tap/terrarium
    (or grab a binary from Releases)

## How it works
    terrarium run -- npm test     # any command works: cargo test, pytest, go test…
- Tests pass → your creature gets fed and grows. Tests fail → it just sulks a little.
- **It never dies.** Failing tests are part of TDD; terrarium never punishes you for running them.
- `terrarium` opens the tank. `terrarium status` prints a one-liner for your prompt.

## What it looks at
- Exactly one integer: the exit code. Your test output is never parsed, stored, or sent anywhere.
- Fully local. No network code at all.

## Per-project tanks
- Each repo gets its own tank (state stored under your OS data dir).

## FAQ
- "Can I cheat with `terrarium run -- true`?" — Sure, but the fish gets full quickly. It's your terrarium.
- Windows? — Works in Windows Terminal (best effort).

## Credits & License
- MIT. Inspired by oneko-era terminal toys and asciiquarium.
```

ポイント: デモ GIF とインストール 1 行がファーストビューに収まること。
GIF は [vhs](https://github.com/charmbracelet/vhs) でスクリプト録画し、リリースごとに再生成できるようにする。

---

## 10. リスクと実装前検証項目

| 優先度 | 項目 | 内容 | 検証方法 |
|---|---|---|---|
| P0 | 子プロセスラップの透明性 | stdio inherit で色付き・インタラクティブな出力が壊れないか。exit code 伝播（Unix シグナル死 = 128+n、Windows の生 code）。Ctrl-C が子に正しく渡り、terrarium が中断をニュートラル判定できるか | 捨てるプロトタイプを最初に書く（Issue #1）。cargo test / npm test / pytest / 対話コマンドで確認 |
| P0 | `Viewport::Inline` の端末互換性 | run 後のインライン演出が iTerm2 / Terminal.app / tmux 内 / Windows Terminal で崩れないか（スクロール領域・カーソル位置復元） | 同プロトタイプ。**崩れる場合は「演出は 1 行サマリのみ + 水槽 TUI で確認」へ計画的に縮退**（体験の核は失われない） |
| P1 | `status` の起動速度 | プロンプト組み込みは 50ms 未満が必要（シェルが重くなった瞬間にアンインストールされる） | hyperfine で計測。TUI 系 crate の初期化をこのパスに含めない構造を確認 |
| P1 | 名前の衝突 | crates.io の `terrarium` は**空きを確認済み**（2026-07-05、crates.io API で 404）。Homebrew formula / 有名 OSS との衝突の簡易確認をリリース前に再実施 | リリース直前に検索 + 早期に crates.io へ 0.1.0 を publish して確保 |
| P2 | 256 色 / 16 色 / 非 UTF-8 端末での見た目 | フォールバック時にレイアウトが崩れない・意味が失われないこと | 手動テストマトリクス（iTerm2 / Terminal.app / tmux / Linux console / Windows Terminal） |
| P2 | state スキーマ進化 | v2 でフィールドを足すときに v1 の水槽を壊さない | version フィールド + serde default で設計時に担保（§5） |
| P3 | チート耐性 | `terrarium run -- true` 連打 | 満腹クールダウン 60 秒（§5）で「ゲームとして」十分と割り切り済み |
| P3 | "terrarium" が一般語で発見性が低い | GitHub topic / crates.io キーワードの整備でカバー | README 整備時に対応 |

**最重要リスク**: P0 の「子プロセスラップの透明性」。
ここが壊れると「普段のテストコマンドを置き換えても何も失わない」という前提が崩れ、プロダクトが成立しない。
inline 演出（もう 1 つの P0）は縮退先があるが、素通し・伝播には代替がないため、本実装前に捨てられるプロトタイプで必ず確認する。

---

## 11. v1 Issue 分割案（8 個）

- **#1 `Spike: transparent command wrapping and inline post-run animation`** — ラベル: `spike`, `design`
  子プロセス spawn（stdio inherit）+ exit code 伝播 + Ctrl-C 伝播と、`Viewport::Inline` による数行のインライン演出を 100〜200 行のプロトタイプで検証する（P0 リスク 2 件の消化）。cargo test / npm test / pytest / 色付き・対話コマンドで macOS・Linux（Windows は CI）を確認する。
  受け入れ条件: ラップ実行で出力・色・対話性が素の実行と区別できず、exit code が一致し、非 TTY で演出が消えること。inline 演出の各端末（iTerm2 / Terminal.app / tmux / Windows Terminal）での成否と縮退判断を Issue コメントに記録。

- **#2 `Implement per-project state store with atomic JSON persistence`** — ラベル: `enhancement`
  directories によるパス解決、project root 探索（`.git` 上方探索 → cwd フォールバック）、SHA-256 ハッシュ命名、tmp + rename の atomic write、version フィールド、破損時の退避・再生成を実装する。
  受け入れ条件: 2 つのプロジェクトで状態が混ざらず、書き込み中の kill でもファイルが壊れず、破損 JSON が退避されて警告 1 行が出るユニット/結合テストが通る。

- **#3 `Implement growth engine: feeding, mood, streaks, milestones`** — ラベル: `enhancement`
  §5 の育成モデル（餌 / growth / mood / streak / 満腹クールダウン / 空腹 / 環境マイルストーン / 中断ニュートラル）を純関数として実装する。死亡状態は型として存在させない。
  受け入れ条件: §5 の表の全ケースを網羅するユニットテストが通り、growth が単調非減少であることのプロパティテストがある。

- **#4 `Build terrarium run wrapper with post-run feeding animation`** — ラベル: `enhancement`, `ux`
  #1 の知見で本実装。spawn / exit code 伝播 / 非 TTY 検出 / `--quiet` / 給餌ミニ演出（成功）と 1 行メッセージ（失敗・満腹・中断）を実装する。
  受け入れ条件: `terrarium run -- cargo test` が exit code を保存・伝播し、TTY で演出が出て、パイプ時は子の出力のみになる。演出込みのオーバーヘッドが体感を損ねない（演出スキップ時 +50ms 未満）。

- **#5 `Build aquarium TUI with tick-driven animation`** — ラベル: `enhancement`, `ux`
  ratatui の tick 駆動イベントループで水槽画面（§6 レイアウト）を実装する。魚の 2 フレームアニメ + ランダムウォーク、泡、水草、mood 別挙動、成長段階別スプライト、リサイズ・狭幅縮退、truecolor/256/16 色フォールバック、panic hook での端末復元を含む。
  受け入れ条件: 各 stage × mood の見た目が §6 の表どおりに変わり、リサイズしても崩れず、アイドル時 CPU が計測上 1% 未満で、異常終了後も端末が壊れない。

- **#6 `Add terrarium status one-liner for shell prompts`** — ラベル: `enhancement`
  TUI を初期化しない高速パスで 1 行サマリを出力する。state がなければ無出力 + exit 0。非 UTF-8 端末では ASCII 表現に縮退する。
  受け入れ条件: hyperfine で 50ms 未満、プロンプト（starship / PS1）と tmux status への組み込み例が README 断片として動作確認済み。

- **#7 `Set up release pipeline: cargo-dist, Homebrew tap, crates.io`** — ラベル: `infra`
  cargo-dist で tag push → 3 OS バイナリを GitHub Releases に自動添付。`Saber5656/homebrew-tap` に formula を追加し、crates.io へ publish して名前を確保する。CI（fmt / clippy / test を 3 OS で）もここで整える。
  受け入れ条件: タグ push だけで Releases に 3 OS の成果物が並び、`cargo install terrarium` と `brew install Saber5656/tap/terrarium` がクリーン環境で成功する。

- **#8 `Write README with banner, demo GIF, and one-line install`** — ラベル: `docs`
  §9 の構成で英語 README を作成する。vhs のスクリプトで「run が緑 → 給餌 → 水槽」のデモ GIF を録画し、バナー画像を用意する。プライバシー節（exit code しか見ない）と Windows ベストエフォートの明記を含む。
  受け入れ条件: GIF とインストール 1 行がファーストビューに収まり、vhs スクリプトがリポジトリに入っていて GIF を再生成でき、バッジが 3 個以下である。

推奨着手順: #1 → #2 → #3 → (#4, #5 並行) → #6 → (#7, #8 並行)。

---

## 参考資料（調査ソース一覧）

- 類似 OSS（位置づけ）
  - [usik/tamagotchi — A terminal Tamagotchi](https://github.com/usik/tamagotchi)
  - [apoorvgarg31/terminal-pet — feeds on your git commits](https://github.com/apoorvgarg31/terminal-pet)
  - [vincent-k2026/codachi — pet in the Claude Code statusline](https://github.com/vincent-k2026/codachi)
  - [ezeoleaf/termagotchi — terminal tamagotchi](https://github.com/ezeoleaf/termagotchi)
  - [terminalgotchi (crates.io) — cargo test / git commit で XP](https://crates.io/crates/terminalgotchi)
  - [cmatsuoka/asciiquarium — ASCII art aquarium](https://github.com/cmatsuoka/asciiquarium)
- ratatui のイベントループ / アニメーション
  - [Ratatui recipe: Terminal and EventHandler（tick 駆動の定石）](https://ratatui.rs/recipes/apps/terminal-and-event-handler/)
  - [Ratatui tutorial: Full Async Events（tokio::select! で tick/render 分離）](https://ratatui.rs/tutorials/counter-async-app/full-async-events/)
  - [ratatui/tachyonfx — effects and animation library for Ratatui](https://github.com/ratatui/tachyonfx)
  - [ratatui::Viewport（Inline viewport）— docs.rs](https://docs.rs/ratatui/latest/ratatui/enum.Viewport.html)
  - [ratatui/ratatui](https://github.com/ratatui/ratatui) / [crossterm-rs/crossterm](https://github.com/crossterm-rs/crossterm)
- 実装部品・配布
  - [directories (crates.io) — XDG / Application Support / AppData 解決](https://crates.io/crates/directories)
  - [axodotdev/cargo-dist — リリースバイナリ自動化](https://github.com/axodotdev/cargo-dist)
  - [Homebrew: Taps (Third-Party Repositories)](https://docs.brew.sh/Taps)
  - [charmbracelet/vhs — ターミナル録画によるデモ GIF 生成](https://github.com/charmbracelet/vhs)

---

## Changelog

- 2026-07-05: 初版

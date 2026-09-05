# pixelparty 設計書 (v1)

- Repository: `github.com/Saber5656/pixelparty`
- Tagline: "An event-ready collaborative canvas kit inspired by r/place."
- License: MIT（前提）/ クラウド送信なし（主催者のマシンから外に出ない self-host）/ 個人 OSS・最小実装で早期リリース
- 作成日: 2026-07-05

---

## 1. コンセプトと既存クローンとの位置づけ

pixelparty は、r/place 型の共同ピクセルキャンバスを「自分のイベントで 5 分で開催できる」ようにするキットである。
主催者がノート PC でバイナリ 1 個を起動すると、ターミナルに参加用 URL と QR コードが出る。
参加者はスマホで QR を読むだけで、登録もアプリも不要で 1 ピクセルずつ描き始められる。
イベントが終わったら Ctrl+C で完成キャンバスが PNG として残り、そのまま記念品になる。

**既存の r/place 系との差別化（車輪の再発明にならない位置づけ）**

| 既存 | 性格 | pixelparty との違い |
|---|---|---|
| r/place 本家（Reddit） | 数年に一度の超大規模お祭り。Reddit アカウント必須。参加はできても開催はできない | 「開催する側」になるための道具。アカウント不要・自分の会場で完結 |
| Pxls（pxlsspace/Pxls） | 常設コミュニティキャンバスの本格実装。Java + 設定 + モデレーション機能一式 | 常設運用しない。数時間のイベントに 1 バイナリで持ち込み、PNG を残して撤収する |
| rbxb/place（Go） | セルフホスト可能な軽量クローン。常設 Web サービス志向 | イベント動線（QR 誘導・ターミナル運用・PNG 持ち帰り）自体を主機能として持つ |
| art98 / dynastic/place 等 | MERN 等の多層構成。DB とデプロイ作業が前提 | 依存ゼロの単一バイナリ。「サーバを建てる」という作業を消す |

つまり既存クローンが向かう方向（常設・スケール・アカウント・モデレーション）を全部捨て、
**「持ち運べるイベント道具」**（立ち上げ 5 分・QR 参加・記念 PNG）に全振りすることが存在理由。
常設サービスの再現ではなく、文化祭・勉強会・オフィスパーティー・結婚式二次会のための「キット」として設計する。

---

## 2. v1 スコープ

| 区分 | 項目 | 備考 |
|---|---|---|
| 入れる | 固定サイズキャンバス（`--size`、デフォルト 256×256） | 起動時固定にすると状態管理が最も単純になる |
| 入れる | 16 色固定パレット（r/place 2017 準拠） | 色数を絞るほどドット絵の見栄えの下限が上がる |
| 入れる | クールダウン（`--cooldown`、デフォルト 5 秒、0 で無効） | 荒らし抑止と「1 ピクセルの重み」の演出。規模に応じ主催者が調整 |
| 入れる | last-write-wins 同期（スナップショット + 差分、バイナリ WS） | §5。サーバが唯一の真実 |
| 入れる | QR 誘導（ターミナル表示 + プロジェクタ用 `/qr` ページ） | 参加動線の本命。`/qr` は静的 1 ページでコスト極小 |
| 入れる | 永続化（定期スナップショット保存 + 起動時復元） | サーバ再起動・クラッシュでイベントの成果を失わない |
| 入れる | PNG エクスポート（終了時自動 + ターミナルコマンド） | 「記念品」がプロダクトの締め。拡大 nearest-neighbor で出力 |
| 入れる | ターミナルコマンド（reset / freeze / export / quit） | 管理画面を作らないための最小運用手段 |
| 入れる | 単一バイナリ配布（フロントエンドを rust-embed で同梱） | 「キット」の意味そのもの。配布物は 1 ファイル |
| 入れる | 同時 〜100 クライアント（LAN イベント規模） | §4 の試算どおり余裕。それ以上は非目標 |
| 入れない | アカウント・認証・名前入力 | 匿名参加が体験の核（§7）。イベントは物理的同室が信頼の基盤 |
| 入れない | モデレーションツール（BAN・個人履歴・ロールバック） | 「誰が置いたか」を記録しない設計（§7）と原理的に衝突。freeze + reset で足りる |
| 入れない | パブリック常設運用（TLS・リバースプロキシ・強化レート制限） | Pxls 等の領分。README で非目標と明言し issue 対応コストを断つ |
| 入れない | タイムラプス生成 | v2 の筆頭候補。スナップショット形式に世代保存の口だけ残す |
| 入れない | CRDT・オフライン編集 | 単一サーバで合流問題が存在しない（§4 に理由を明記） |
| 入れない | チャット・リアクション | スマホの画面は狭い。描くことだけに集中させる |
| 入れない | 複数ルーム / 複数キャンバス | 1 プロセス = 1 キャンバス。複数欲しければ複数起動すればよい |
| 入れない | 管理 Web UI | ターミナルで足りる。攻撃面と実装量だけ増える |
| 入れない | Docker イメージ | 常設運用の道具であり思想が違う。要望を見て v1.1 で判断 |
| 入れない | 画像アップロード・ステンシル | 荒らしの主要経路になりがち。手で描くのがイベント |

---

## 3. 対応プラットフォームと優先順位

サーバ（主催者側）とクライアント（参加者側）で分けて考える。

### サーバ（単一バイナリ）

| 優先度 | OS | 判断 | 理由 |
|---|---|---|---|
| 1 (v1) | macOS 13+（Apple Silicon / Intel） | 対応 | 開発者の環境。実機検証が最速 |
| 1 (v1) | Linux（x86_64 / arm64） | 対応 | CI で検証容易。Raspberry Pi を会場ブースに常設する使い方も想定 |
| 2 (v1) | Windows 10+（x86_64） | 対応（ビルド + CI テスト） | Rust はクロスビルドがほぼ無料なので出す。ただし実機検証は best-effort、firewall 周りは §10 P2 |
| — | Docker / クラウド | 見送り | §2 のとおり非目標 |

Rust 製サーバはクロスコンパイルの追加コストがほぼゼロのため、3 OS 同時対応が「早く出す」と両立する
（cursorpets の macOS 特化と逆の判断になるのは、GUI 統合が不要な headless CLI だから）。

### クライアント（参加者のブラウザ）

| 優先度 | 環境 | 判断 | 理由 |
|---|---|---|---|
| 1 (v1) | モバイルブラウザ（iOS Safari 16+ / Android Chrome） | 最優先で最適化 | 参加者は 100% スマホ想定。QR → ブラウザが「登録もアプリも不要」の核 |
| 2 (v1) | デスクトップブラウザ | 動作する | 主催者のプロジェクタ全景表示に使う。専用最適化はしない |
| — | ネイティブアプリ / PWA install | 作らない | インストールという摩擦を足した瞬間にイベント動線が死ぬ |

---

## 4. 技術選定

### サーバ言語・フレームワーク比較

| 候補 | 単一バイナリ | WebSocket 実績 | 静的埋め込み | 判定 |
|---|---|---|---|---|
| **Rust（axum + tokio）採用** | ◎ 数 MB | 公式 chat example が broadcast パターンそのもの | rust-embed が成熟（下記調査） | ✅ |
| Go（net/http + websocket） | ◎ | 豊富（先行例 rbxb/place が Go 実装） | `embed` 標準 | △ 成立するが、開発者の主スタックが Rust であり `cargo install` 配布も欲しい。優劣ではなく速度の問題 |
| Node.js（ws + express） | ❌ ランタイム同梱（pkg 系はサイズ・保守が不安定） | 豊富 | 可能 | ❌ 「バイナリ 1 個」が崩れる |
| Python（FastAPI + uvicorn） | ❌ PyInstaller は起動が遅く AV 誤検知も多い | あり | △ | ❌ 同上 |

### フロントエンド比較

| 候補 | 判定 | 理由 |
|---|---|---|
| **Vanilla TypeScript + Canvas 2D（採用）** | ✅ | 画面は 1 枚、状態は盤面 1 個。`drawImage` / `putImageData` で足りる。esbuild で gzip 後 ~10KB を狙う |
| React / Vue | ❌ | 盤面 1 個にコンポーネントツリーは過剰。バンドルが膨らむだけ |
| Svelte | ❌ | 軽量だが v1 の規模ではビルドチェーンを増やす理由がない |

### 採用スタック

| 層 | 技術 | 理由 |
|---|---|---|
| HTTP / WS | axum 0.8 + tokio | `axum::extract::ws` 内蔵で WS の追加依存なし。tokio エコシステムの本流 |
| ブロードキャスト | `tokio::sync::broadcast` | axum 公式 chat example の定石。`RecvError::Lagged` 検知で遅延クライアントを再同期できる |
| アセット埋め込み | rust-embed | debug は FS 読み・release は埋め込みの二面性で開発体験と単一バイナリを両立 |
| QR | qr2term | ターミナル QR 表示が 1 行で済む |
| PNG | image crate | エクスポート時のみ使用 |
| CLI | clap（derive） | `--size` `--cooldown` `--port` `--data-dir` |
| 永続化 | 独自バイナリ形式（magic + サイズ + ピクセル列） | serde すら不要の単純構造（§5） |
| フロント | TypeScript + Canvas 2D + esbuild | フレームワークなし。ランタイム依存ゼロ |

### 調査結果 1: axum での WebSocket ブロードキャスト

- axum リポジトリ同梱の chat example が「`AppState` に `broadcast::Sender` を持ち、接続ごとに `subscribe()`、
  ソケットを送受 2 task に split して受信 task が channel → WS へ転送」という本設計と同型のパターンを公式に示している。
- 規模の傍証: r/place 本家は 1000×1000・16 色 = 4bit packing で **1M ピクセルを 500KB のスナップショット**にし、
  差分を WebSocket 配信する構成で世界規模（ピーク 18 万 write/秒）を捌いた（Fastly 事例）。
  pixelparty は同型プロトコルの縮小版（64KB・100 接続・クールダウンあり）であり、構造的に余裕がある。
- 定量試算: クールダウン 5 秒・満員 100 人で最大 20 pixel/秒 → 6 バイト差分 × 100 接続 = **総送信 ~12KB/秒**。
  クールダウン無効でも 1 人 1 タップ/秒として ~60KB/秒。Wi-Fi の LAN 帯域に対して誤差レベル。

### 調査結果 2: rust-embed による埋め込み配信

- rust-embed は upstream リポジトリに axum 用 example を同梱しており、axum との組み合わせは実績のある定番。
  ラッパー crate（axum-embed）も存在するが、v1 はハンドラ 20 行程度なので直接 rust-embed を使い依存を増やさない。
- debug ビルドではファイルシステムから毎回読み、release では静的配列になる仕様のため、
  フロント開発中はブラウザリロードだけで反映され、配布時は単一バイナリになる。開発体験と配布要件が両立する。

### CRDT を使わない理由（重要な設計判断）

キャンバスの真実はサーバのメモリ上に 1 つだけ存在し、全変更はサーバを経由する。
ピアツーピアも複数サーバも存在しないため、CRDT が解決する「並行編集の合流」問題がそもそも発生しない。
同一ピクセルへの同時書き込みは**サーバ到着順の last-write-wins** で解決し、これは r/place 本家と同じ仕様であるだけでなく、
「上書き合戦」自体が r/place 系の遊びの本質である。CRDT はコード複雑度と依存だけを増やすため不採用。

---

## 5. アーキテクチャ

1 プロセス。axum の tokio runtime 上に、キャンバスを所有する single-writer task を置く。

```
                    Organizer's laptop  (single binary)
┌──────────────────────────────────────────────────────────────┐
│ pixelparty server (Rust, axum + tokio)                       │
│                                                              │
│  ┌───────────┐  GET / , /qr   ┌─────────────────────────┐    │
│  │ HTTP      │◀──────────────│ embedded frontend        │    │
│  │ (axum)    │  (rust-embed) │ (TS + Canvas, prebuilt)  │    │
│  └─────┬─────┘               └─────────────────────────┘    │
│        │ /ws upgrade                                         │
│  ┌─────▼──────────┐  PLACE(x,y,c)   ┌──────────────────┐    │
│  │ WS session     │───── mpsc ─────▶│ Canvas task       │    │
│  │ (task ×N)      │                 │ (single writer)   │    │
│  │                │◀── broadcast ───│  Vec<u8> w×h      │    │
│  └─────▲──────────┘  PIXEL diff     │  + cooldown map   │    │
│        │                            └────────┬─────────┘    │
│  ┌─────┴─────┐ reset/freeze/export           │ 10s dirty     │
│  │ Terminal  │───── mpsc ────────────────────┤               │
│  │ (stdin)   │                       ┌───────▼──────────┐    │
│  └───────────┘                       │ Persistence       │    │
│                                      │ canvas.pxp (atomic│    │
│                                      │ save) + PNG export│    │
│                                      └──────────────────┘    │
└──────────────────────────────────────────────────────────────┘
          ▲  http:// + ws://  (LAN 内のみ, 例 192.168.1.23:8080)
          │
     📱📱📱 participants' phones (QR → mobile browser, 匿名)
```

### 同期プロトコル（バイナリ WebSocket フレーム、little-endian 固定）

| ID | 方向 | 形式 | ペイロード | 用途 |
|---|---|---|---|---|
| CONFIG | S→C | text (JSON) | palette RGB 配列, cooldown_ms, frozen, size | 接続直後のメタ情報。低頻度・拡張前提なので JSON |
| `0x01` SNAPSHOT | S→C | binary | `w:u16, h:u16, pixels:u8[w×h]` | 全盤面。256×256 = **64KB + 5B**。接続時と再同期時 |
| `0x02` PIXEL | S→C 全員 | binary | `x:u16, y:u16, color:u8` | 確定した 1 ピクセル差分（**6 バイト**） |
| `0x03` PLACE | C→S | binary | `x:u16, y:u16, color:u8` | 配置要求 |
| `0x04` COOLDOWN | S→C 個別 | binary | `remaining_ms:u32` | 拒否理由の通知と UI タイマーの同期 |
| STATUS | S→C 全員 | text (JSON) | connected 数, frozen 切替, reset 通知 | 低頻度イベント |

- 配置フロー: `PLACE` → サーバ検証（範囲・色 index < 16・クールダウン・frozen）→ Canvas task が反映 →
  `PIXEL` を broadcast。**送信者も broadcast の echo で自分のピクセルを描く**（楽観描画をしない。
  LAN の往復数 ms では体感差がなく、クライアントに「未確定状態」を持たせない方が単純）。
- スナップショットは u8 = 1 ピクセル 1 バイトのまま送る。r/place のような 4bit packing で 32KB にできるが、
  64KB は LAN では誤差であり、パック処理の複雑さに見合わない（100 台一斉接続でも合計 6.4MB のバースト）。
- 再接続: クライアントは切断時に指数バックオフで自動再接続し、**再接続 = スナップショット再取得**とする。
  差分の取りこぼし追跡（シーケンス番号管理）を持たない、最も壊れにくい設計。
- `broadcast` の `RecvError::Lagged`（遅いクライアント）を検知したら、その接続にだけ SNAPSHOT を再送して復帰させる。
- keepalive: WS ping/pong 30 秒。スマホのスリープ復帰は再接続扱いで自然に回復する。

### 状態管理と永続化

- 正はメモリ上の `Vec<u8>`（w×h、パレット index）。**mpsc で単一 writer task に集約**するため
  ロック競合がなく、last-write-wins の順序が mpsc 到着順として一意に定まる。
- クールダウンは **WS 接続（session）単位**で writer task 内の map で管理。IP 単位にしない理由:
  匿名設計（§7）を守り、NAT・共有端末の誤爆を避ける。リロードで回避できる抜け道は §10 P1 で扱う。
- スナップショット保存: 10 秒ごと、dirty な場合のみ `canvas.pxp`（`PXP1` magic + u16 w/h + ピクセル列）へ
  **tmp ファイル書き込み → rename の atomic 保存**。起動時に同ファイルがあれば復元（サイズ不一致は警告して退避 + 新規）。
- 終了時（Ctrl+C / SIGTERM / `q`）: 最終スナップショット保存 + `pixelparty-<timestamp>.png` を
  nearest-neighbor 8 倍拡大（256→2048px）でエクスポート。ドット絵がそのまま SNS に貼れるサイズになる。
- タイムラプスへの口: `.pxp` を世代保存すれば実現できる構造にしておく（実装は v2。§2）。

---

## 6. UI/UX

### 主催者フロー（会場で 5 分）

1. バイナリを起動（`./pixelparty` または `--size 128 --cooldown 3` 等）
2. ターミナルに参加 URL・QR・状態が出る。プロジェクタには `/qr` ページか、全景表示用にブラウザで参加 URL を映す
3. 参加者が描いている間、ターミナルの接続数・配置数カウンタで盛り上がりを把握。荒れたら `f`（freeze）
4. 締めで `q` → PNG の保存パスが表示される → その場でプロジェクタに映して記念撮影

```
  pixelparty v1.0.0 — 256×256, 16 colors, cooldown 5s
  Serving on http://192.168.1.23:8080   (interface: en0)

  █▀▀▀▀▀█ ▀▄█▄▀ █▀▀▀▀▀█     ← 参加用 QR（qr2term）
  █ ███ █ ▄▀▄▀▄ █ ███ █
  ...

  73 connected · 4,231 pixels placed · saved 12s ago
  commands: [r]eset  [f]reeze  [e]xport png  [q]uit
```

### 参加者のスマホ画面（1 画面のみ）

```
┌────────────────────────┐
│                        │ ← 全画面 Canvas。ピンチズーム / パン
│      ▓▓▒▒▓▓            │
│     ▓[+]▒▓▓            │ ← タップでセルを選択（ハイライト表示）
│      ▓▓▓▓              │
│                        │
├────────────────────────┤
│ ■■■■■■■■■■■■■■■■ │ ← 16 色パレット（1 行で全色）
│    [ Place ]     ◔ 3s  │ ← 確定ボタン + クールダウン残り
└────────────────────────┘
```

- **タップ = 選択、Place = 確定の 2 段階配置**（r/place 本家と同じ操作系）。ズームが浅い状態での誤配置を防ぐ
- クールダウン中は Place ボタンが円形タイマーになる。残り時間はサーバの `COOLDOWN` 応答と同期
- 接続人数を小さく表示（「いま 73 人で描いてる」という一体感がイベントの燃料）
- 切断時は「Reconnecting…」オーバーレイ → 自動再接続 → スナップショット再取得で無操作復帰

### QR 誘導

- 基本はターミナル QR。加えて `/qr` ページ（URL と QR を大きく表示するだけの静的ページ）を同梱し、
  プロジェクタ・サブディスプレイに映す運用を README で推奨する。受付に紙で貼る場合も `/qr` のスクリーンショットで済む

### エッジケースの扱い

| ケース | 挙動 / 判断 |
|---|---|
| スマホがスリープ → WS 切断 | 復帰時に自動再接続 + スナップショット再取得。取りこぼし追跡はしない（§5） |
| 同一ピクセルへの同時タップ | サーバ到着順の last-write-wins。上書きされるのは仕様であり遊びの一部 |
| 塗りつぶし荒らし | クールダウン + freeze + **同室の社会的抑止**で対処。技術的な完全対策は非目標（§2）と README に明記 |
| 会場 Wi-Fi の AP isolation | 接続不能になる最大の運用リスク。§10 P0 と README の Event checklist で対処 |
| 100 台超の接続 | 拒否はしないが動作保証外。README に目安を記載 |
| iOS Safari の 100vh 問題 / ダブルタップズーム | `dvh` 単位 + `touch-action` 制御で抑止。ピンチズームはキャンバス内実装 |
| プロジェクタでの全景観戦 | 専用観戦モードは作らず、PC ブラウザで参加ページを開きズームアウトで代用 |

---

## 7. プライバシー設計

原則: **「会場の外に何も出さず、会場の中でも誰も特定しない」**

| 項目 | 方針 |
|---|---|
| ネットワーク境界 | サーバは主催者マシンで LAN 内のみ待ち受け（参加者を受け入れるため `0.0.0.0` bind、これは本プロダクトの機能そのもの）。**外向き接続はゼロ**: テレメトリなし、アップデート確認なし、CDN・Web フォント・解析スクリプトなし。全アセットを rust-embed で同梱するため、外部リクエストが存在しないことが構造的に保証される |
| 参加者の識別 | **匿名**。アカウントなし・名前入力なし・Cookie なし・localStorage 不使用。session は WS 接続そのもの（プロセスメモリ内のみ）。IP アドレスは接続管理に使うだけでログにもファイルにも書かない |
| 保存されるもの | ピクセル色の配列のみ（`canvas.pxp` と PNG）。**「誰がどこを塗ったか」は保持も記録もしない**。この結果としてモデレーション機能が原理的に作れないのは、意図した設計（§2 の「入れない」と表裏一体） |
| 成果物 | PNG は色データのみ。EXIF・作者情報・タイムスタンプ以外のメタデータなし |
| 平文通信 | LAN 内 http/ws 平文。流れるのはピクセル座標と色 index のみで個人情報を含まない。LAN IP への自己署名 TLS は証明書警告で参加動線を壊すため採用しない（README の Privacy 節に正直に記載） |
| ログ | stdout に接続数・配置数などの集計値のみ。座標ストリームを出すデバッグログは `--verbose` 明示時のみ |

---

## 8. 配布方法

| 項目 | v1 の方針 | 理由 |
|---|---|---|
| 一次配布 | GitHub Releases に 5 ターゲットの単一バイナリ + sha256（macos-arm64 / macos-x64 / linux-x64 / linux-arm64 / windows-x64） | tag push → CI 自動生成。バイナリ 1 個が「キット」の本体 |
| 1 行インストール | `curl -L <releases URL> \| tar xz` を README 最上部に | curl 取得は macOS の quarantine 属性が付かず Gatekeeper 問題を回避できる（ブラウザ DL 者向けに `xattr` 手順も併記） |
| cargo install | `cargo install pixelparty`（crates.io へ publish） | Rust ユーザー向けの正規ルート。ビルド済みフロント dist を crate に同梱する（実務は §10 P1、Issue #8 で確定） |
| Homebrew tap | v1 では見送り、v1.1 で検討 | curl 1 行で足りる。tap の保守を増やさない |
| Docker | 出さない | §2。常設運用の道具であり思想が違う |
| コード署名 / notarization | なし | GUI .app と違い CLI バイナリは curl 配布で実用上困らない。$99/年の投資は見送り |

---

## 9. README 構成案（英語）

```markdown
<バナー画像: ドット絵の群衆が巨大キャンバスを塗っている横長 PNG>

# pixelparty 🎨
> An event-ready collaborative canvas kit inspired by r/place.

<デモ GIF: ターミナル起動 → QR → スマホ 2 台がタップ → キャンバスが埋まり PNG 保存、10 秒ループ>

## Install
curl -L https://github.com/Saber5656/pixelparty/releases/latest/download/pixelparty-<os>-<arch>.tar.gz | tar xz
(or `cargo install pixelparty`, or grab a binary from Releases)

## Host a party
./pixelparty                        # 256×256, 16 colors, 5s cooldown
./pixelparty --size 128 --cooldown 0
- ターミナルに QR が出る → 参加者はスキャンして描くだけ（no accounts, no app）
- Ctrl+C でキャンバスが PNG 保存される。"That's the party favor."

## Why another r/place clone?
- Pxls などは常設コミュニティキャンバス。pixelparty は "one binary, one LAN,
  one hour, one PNG" のパーティー道具、という 2〜3 文

## Options
- --size / --cooldown / --port / --data-dir の表

## Event network checklist
- 会場 Wi-Fi の AP/client isolation で繋がらない場合がある → モバイルルータ持参 /
  スマホテザリング / 会場 NW 管理者に isolation 解除を依頼、の 3 択
- OS ファイアウォールの受信許可プロンプトに Allow すること（macOS / Windows）

## Privacy
- Everything stays on the host machine. No accounts, no analytics, no internet required.

## License
MIT
```

ポイント: デモ GIF とインストールコマンドがファーストビューに収まること。
バッジは license / release / downloads の 3 つまで。
「できないこと（常設運用・モデレーション）」を Why 節で最初から正直に書き、issue 対応コストを断つ。

---

## 10. リスクと実装前検証項目

| 優先度 | 項目 | 内容 | 検証方法 |
|---|---|---|---|
| **P0** | 実スマホ + LAN での WebSocket 同期の成立 | iOS Safari / Android Chrome 実機での `ws://`（非 TLS）接続、64KB スナップショット受信、差分反映遅延、〜100 接続時の broadcast 挙動、スリープ復帰の再接続 | 捨てる前提のプロトタイプを最初に書く（Issue #1）。実機 2 台 + 模擬 100 接続で計測 |
| **P0** | 会場 Wi-Fi の AP isolation（client isolation） | ゲスト SSID では L2 でクライアント間通信が遮断され、参加者から主催者 PC に**原理的に届かない**会場がある（Meraki 等はゲートウェイ宛以外を firewall で落とす実装）。コードでは解決不能 | 同プロトタイプで isolation 有効 SSID を再現し「繋がらないこと」を確認 → 回避策（モバイルルータ持参 / スマホテザリング / 会場管理者へ解除依頼）の有効性を検証し、README の Event checklist（Issue #9）に落とす |
| P1 | rust-embed と `cargo install` の両立 | crates.io パッケージへのビルド済み dist 同梱（`include` 指定）とフロント変更時のビルド順序、debug 時 FS 読みの開発体験 | Issue #5, #8 で構成を確定。`cargo package --list` で同梱を確認 |
| P1 | クールダウンの回避（リロードで session 更新） | session 単位クールダウンはページ再読込でリセットできる。匿名設計（§7）の意図的な代償 | イベントでは freeze + 社会的抑止で許容と README に明記。実害が出たら IP 単位オプションを v1.1 で判断 |
| P2 | ターミナル QR の可読性 | 端末のフォント・配色・行間で QR が読み取れないことがある | `/qr` ページで冗長化（Issue #6）。複数ターミナルで実機読み取り試験 |
| P2 | OS ファイアウォールの初回プロンプト | macOS「ローカルネットワーク」許可 / Windows Defender の受信許可を主催者が拒否すると誰も繋がらない | README checklist に OS 別手順を記載。spike 時に各 OS の文言を採取 |
| P3 | 名前の衝突 | crates.io の `pixelparty` および類似プロダクト・商標の簡易確認 | リリース前に `cargo search` と GitHub / Web 検索 |

**最重要リスク**: P0 の 2 件。前者が崩れると通信方式の再設計（SSE / ロングポーリング化）になり、
後者は「会場で動かない」というプロダクトの存在意義そのものに関わる。
どちらも本実装前に Issue #1 の捨てるプロトタイプで検証する。

---

## 11. v1 Issue 分割案（9 個）

- **#1 `Spike: LAN WebSocket canvas prototype with real phones`** — ラベル: `spike`, `design`
  axum + `tokio::sync::broadcast` + rust-embed の最小構成で「起動 → スマホで開く → ピクセル相互同期」の一往復を作り、P0 リスク 2 件（実スマホ/LAN での WS 同期、AP isolation）を検証する。コードは捨てる前提。
  受け入れ条件: 同一 Wi-Fi の iOS Safari / Android Chrome 実機 2 台で相手のピクセルが体感即時（<200ms）に反映される。模擬 100 接続でスナップショット配信と差分 broadcast が破綻しない。isolation 有効 SSID で繋がらないことと、テザリング等の回避策の有効性を確認する。結果（計測値）を Issue コメントに記録。

- **#2 `Implement canvas core: palette state, validation, last-write-wins`** — ラベル: `enhancement`
  キャンバス状態（`Vec<u8>` パレット index）、16 色固定パレット、`--size` フラグ、座標・色の検証、mpsc + single-writer task による last-write-wins を実装する。
  受け入れ条件: 範囲外・不正色 index の place を拒否するユニットテストが通る。同一ピクセルへの連続書き込みが到着順で解決される。`--size` で 64〜1024 の正方キャンバスを起動できる。

- **#3 `Implement WebSocket sync protocol (snapshot + diff broadcast)`** — ラベル: `enhancement`
  接続時の CONFIG（JSON）+ SNAPSHOT（バイナリ 64KB）送信、以後の PIXEL 差分（6 バイト）の `tokio::sync::broadcast` 配信、クールダウンのサーバ側強制、`Lagged` 受信者への再スナップショットを実装する。
  受け入れ条件: 新規接続がスナップショット + 差分適用で正しい盤面になる結合テストが通る。クールダウン中の PLACE が拒否され COOLDOWN 応答が返る。Lagged 発生時に接続が再スナップショットで復帰する。

- **#4 `Build mobile-first canvas frontend (TypeScript + Canvas)`** — ラベル: `enhancement`, `ux`
  フレームワークなしの TypeScript + Canvas 2D で、ピンチズーム / パン、タップ選択 → Place 確定の 2 段階配置、16 色パレットバー、クールダウン表示、指数バックオフの自動再接続を実装する。
  受け入れ条件: iOS Safari / Android Chrome 実機でズーム・パン・配置が快適に動く。誤タップで即配置されない。切断後に自動再接続して盤面が復元される。

- **#5 `Embed built frontend into the binary with rust-embed`** — ラベル: `enhancement`, `infra`
  esbuild でビルドした dist/ を rust-embed で埋め込み axum から配信する。debug ビルドは FS 読みで開発体験を保ち、release は単一バイナリで完結させる。
  受け入れ条件: release バイナリ 1 個を空ディレクトリに置いて起動し、スマホから全機能が使える。外部への HTTP リクエストが 1 本も発生しない（フォント・CDN 含む）。

- **#6 `Add organizer UX: QR code, LAN IP detection, terminal commands`** — ラベル: `enhancement`, `ux`
  起動時の LAN IP 自動検出と参加 URL + QR（qr2term）のターミナル表示、プロジェクタ用 `/qr` ページ、reset（要確認）/ freeze / export / quit のターミナルコマンド、`--port` `--cooldown` フラグを実装する。
  受け入れ条件: 起動 3 秒以内に URL と QR が表示されスマホで読み取って参加できる。freeze 中は全クライアントの配置が拒否され参加者画面にその旨が出る。`/qr` をプロジェクタに映して読み取れる。

- **#7 `Add persistence: periodic snapshots and PNG export`** — ラベル: `enhancement`
  10 秒ごとの atomic スナップショット保存（tmp + rename）、起動時復元、終了時の最終保存 + nearest-neighbor 拡大 PNG エクスポート（export コマンドでも即時出力）を実装する。
  受け入れ条件: サーバを kill → 再起動して盤面が復元される。Ctrl+C で `pixelparty-<timestamp>.png` が生成され 8 倍拡大されている。保存中クラッシュでもファイルが壊れない。

- **#8 `Set up release CI for cross-platform single binaries`** — ラベル: `infra`
  tag push で macOS（arm64/x64）/ Linux（x64/arm64）/ Windows（x64）の単一バイナリをビルドし GitHub Releases に添付する CI を整備する。`cargo install` 用のビルド済み dist 同梱方針もここで確定する。
  受け入れ条件: タグ push だけで 5 ターゲットのバイナリと sha256 が Releases に並ぶ。`cargo install pixelparty` で同一機能のバイナリが得られる。

- **#9 `Write README with banner, demo GIF, and event network guide`** — ラベル: `docs`
  §9 の構成で英語 README を作成する。バナー、デモ GIF（起動 → QR → スマホ描画 → PNG 保存の 10 秒）、1 行インストール、AP isolation を含む Event network checklist、Privacy 節を含める。
  受け入れ条件: デモ GIF とインストールコマンドがファーストビューに収まる。Event network checklist に AP isolation の説明と回避策（テザリング / モバイルルータ / 会場管理者へ依頼）がある。バッジは 3 つまで。

推奨着手順: #1 → (#2 → #3) と #4 を並行 → #5 → #6 → #7 → (#8, #9 並行)。

---

## 参考資料（技術検証の根拠）

- axum WebSocket ブロードキャスト: [axum 公式 chat example](https://github.com/tokio-rs/axum/blob/main/examples/chat/src/main.rs), [axum Discussion #1335: How can I broadcast message to all of connected websocket clients](https://github.com/tokio-rs/axum/discussions/1335)
- 静的アセットのバイナリ埋め込み: [rust-embed (crates.io)](https://crates.io/crates/rust-embed), [rust-embed 同梱の axum example (docs.rs)](https://docs.rs/crate/rust-embed/latest/source/examples/axum.rs), [axum-embed (docs.rs)](https://docs.rs/axum-embed/latest/axum_embed/)
- r/place 本家のアーキテクチャ（16 色 4bit・スナップショット + WS 差分の根拠）: [Fastly: Reddit on building & scaling r/place](https://www.fastly.com/blog/reddit-on-building-scaling-rplace)
- 先行 OSS（位置づけ表の根拠）: [pxlsspace/Pxls](https://github.com/pxlsspace/Pxls), [rbxb/place](https://github.com/rbxb/place), [dynastic/place](https://github.com/dynastic/place), [creme332/art98](https://github.com/creme332/art98), [chetbox/place](https://github.com/chetbox/place)
- AP isolation / client isolation（P0 リスクの根拠）: [Cisco Meraki: Wireless Client Isolation](https://documentation.meraki.com/MR/Firewall_and_Traffic_Shaping/Wireless_Client_Isolation), [ASUS: How to set up AP Isolated feature](https://www.asus.com/us/support/faq/1044821/), [UniFi: Implementing Network and Client Isolation](https://help.ui.com/hc/en-us/articles/18965560820247-Implementing-Network-and-Client-Isolation-in-UniFi)
- ターミナル QR 表示: [qr2term (crates.io)](https://crates.io/crates/qr2term), [timvisee/qr2term-rs](https://github.com/timvisee/qr2term-rs)

---

## Changelog

- 2026-07-05: 初版

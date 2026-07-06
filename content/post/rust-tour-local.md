+++
title = "Rust Tour Local"
date = 2026-07-06T22:07:04+09:00
tags = ["tech"]
+++
# Rust Tour — ローカル環境ハンズオンチュートリアル

C経験者向け。各章は「解説 → 手を動かす → チェックポイント」の構成です。
すべて `cargo run` で実際に動かしながら進めます。所要目安: 1章30〜60分 × 10章。

---

## 第0章 環境構築

```bash
# rustup（公式ツールチェーンマネージャ）のインストール
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# 確認
rustc --version
cargo --version

# Neovim用: rust-analyzerを入れる（clangdのRust版）
rustup component add rust-analyzer
```

Neovim 0.11+ なら `vim.lsp.config`/`vim.lsp.enable` で clangd と同じ流儀で設定できます:

```lua
vim.lsp.config('rust_analyzer', {
  settings = {
    ['rust-analyzer'] = {
      check = { command = 'clippy' },  -- 保存時にclippyで検査
    },
  },
})
vim.lsp.enable('rust_analyzer')
```

プロジェクト作成:

```bash
cargo new rust-tour
cd rust-tour
cargo run   # "Hello, world!" が出ればOK
```

以降、各章の演習は `src/main.rs` を書き換えて `cargo run` で確認します。
章ごとにファイルを残したい場合は `src/bin/ch01.rs` のように置いて
`cargo run --bin ch01` で実行できます（おすすめ）。

**便利コマンド:**

```bash
cargo check    # コンパイルチェックのみ（速い。頻繁に叩く）
cargo clippy   # リンター。C でいう clang-tidy
cargo fmt      # フォーマッタ。clang-format 相当
```

---

## 第1章 Hello, Ferris! — `fn main` / `println!`

### 解説

- エントリーポイントは `fn main()`。Cの `int main(void)` に相当しますが、戻り値は不要です。
- `println!` は `!` 付き＝**マクロ**。Cの `printf` と違い、フォーマット指定子は型ごとに変えず `{}` で統一されます（型はコンパイル時に検査される）。
- `{:?}` はデバッグ出力。構造体や配列を丸ごと出すときに使います。

```rust
fn main() {
    println!("Hello, Ferris!");
    println!("1 + 1 = {}", 1 + 1);
    println!("{:?}", [1, 2, 3]);      // デバッグ表示
    println!("{:08b}", 0xA5u8);        // 2進数表示: 10100101
}
```

Cとの対比: `printf("%d", x)` の `%d` を間違えるとUBですが、Rustの `{}` は型不一致がコンパイルエラーになります。

### 演習 1

`src/bin/ch01.rs` に以下を書いてください:

1. 「I am learning Rust!」を出力
2. `2 + 3` の計算結果を `{}` で埋め込んで「2 + 3 = 5」と出力
3. `0xFF` を2進数（`{:b}`）と10進数の両方で出力

### チェックポイント

```
I am learning Rust!
2 + 3 = 5
0xFF = 11111111 (255)
```

---

## 第2章 変数と可変性 — `let` / `mut` / シャドーイング

### 解説

- `let` で宣言。**デフォルト不変**。Cの `const` がデフォルトの世界です。
- 変更したいときだけ `let mut`。可変性が型シグネチャレベルで明示されます。
- **シャドーイング**: 同名で再宣言できる。型を変える変換にも使えます。

```rust
fn main() {
    let x = 5;
    // x = 6;            // error[E0384]: cannot assign twice

    let mut y = 10;
    y = 20;               // OK

    let s = "12345";      // &str
    let s = s.len();      // usize にシャドーイング
    println!("x={x}, y={y}, s={s}");   // 変数名を直接埋め込める
}
```

わざと `x = 6;` のコメントを外して `cargo check` してみてください。
Rustのコンパイルエラーは**エラーコード付きで修正案まで出ます**。
`rustc --explain E0384` で詳細解説も読めます。この習慣が学習効率を大きく上げます。

### 演習 2

1. 不変変数 `hostname` に `"router-01"` を代入
2. 可変変数 `uptime_sec` に `0` を代入し、`86400` に更新
3. `uptime_sec` を時間単位にシャドーイング（`let uptime = uptime_sec / 3600;`）
4. 「router-01 uptime: 24h」の形式で出力

---

## 第3章 データ型 — 整数 / 浮動小数点 / タプル / 配列

### 解説

Cと違いサイズが名前に含まれ、環境依存がありません:

| Rust | C相当 | 備考 |
|---|---|---|
| `i8`〜`i128` | `int8_t`〜 | 符号付き |
| `u8`〜`u128` | `uint8_t`〜 | 符号なし |
| `usize` | `size_t` | ポインタ幅 |
| `f32` / `f64` | `float` / `double` | |
| `bool` | `_Bool` | 1バイト保証 |
| `char` | ─ | **4バイト**のUnicodeスカラー値 |

- 整数オーバーフローはデバッグビルドで**パニック**（Cは静かにラップしてUB）。意図的なラップは `wrapping_add` 等を使います。
- タプル: `(i32, f64)` — 異種混合。配列: `[i32; 4]` — 固定長・同種。
- 配列の**範囲外アクセスは実行時パニック**であり、Cのようなバッファオーバーランになりません。

```rust
fn main() {
    let octets: [u8; 4] = [192, 168, 1, 1];
    let addr: u32 = ((octets[0] as u32) << 24)
                  | ((octets[1] as u32) << 16)
                  | ((octets[2] as u32) << 8)
                  |  (octets[3] as u32);
    println!("{:?} = 0x{:08X}", octets, addr);

    let pair: (&str, u16) = ("https", 443);
    println!("{} → {}", pair.0, pair.1);
}
```

### 演習 3

IPv4アドレス `10.0.0.254` を `[u8; 4]` で定義し、ビット演算で `u32` に変換して
`0x0A0000FE` の形式で出力してください（上のコード例を写経せず自力で）。
余裕があれば逆変換（`u32` → 4オクテット）も書いてみてください。
シフトと `& 0xFF` でできます。

---

## 第4章 関数 — 式指向という考え方

### 解説

- 引数の型は**必須**。戻り値は `-> 型`。
- **最後の式（セミコロンなし）が戻り値**。`return` は早期リターン専用が慣習です。
- Rustは**式指向言語**: `if` もブロックも値を返します。

```rust
fn checksum16(data: &[u8]) -> u16 {
    let mut sum: u32 = 0;
    for chunk in data.chunks(2) {
        let word = match chunk {
            [hi, lo] => ((*hi as u32) << 8) | (*lo as u32),
            [hi]     => (*hi as u32) << 8,
            _        => 0,
        };
        sum += word;
    }
    while sum > 0xFFFF {
        sum = (sum & 0xFFFF) + (sum >> 16);
    }
    !(sum as u16)   // 1の補数。最後の式が戻り値
}

fn main() {
    let data = [0x45, 0x00, 0x00, 0x3C];
    println!("checksum: 0x{:04X}", checksum16(&data));
}
```

### 演習 4

1. `fn square(n: i32) -> i32`
2. `fn is_even(n: i32) -> bool`
3. `fn clamp_port(n: i32) -> u16` — 0未満なら0、65535超なら65535、それ以外はそのまま
4. main で動作確認。`clamp_port(-1)`, `clamp_port(99999)`, `clamp_port(8080)` を試す

**セミコロンを付けた場合と付けない場合でどうエラーが変わるか、わざと試してください。**

---

## 第5章 制御フロー — `if`式 / `for` / `loop`

### 解説

- `if` は式: `let label = if n % 2 == 0 { "even" } else { "odd" };`
- 条件に括弧不要、ただし**ブロックの `{}` は省略不可**（Cのぶら下がりif問題が構文レベルで存在しない）。
- `for i in 0..4` は 0,1,2,3。`0..=4` なら 4 を含む。Cの `for(;;)` に相当する柔軟な形はなく、イテレータベースです。
- `loop` は `break 値` で値を返せる無限ループ。

```rust
fn main() {
    // サブネット内のホストアドレスを列挙
    let network = [192u8, 168, 1, 0];
    for host in 1..=5 {
        println!("{}.{}.{}.{}", network[0], network[1], network[2], host);
    }

    // TCPの指数バックオフ風
    let mut rto = 1;
    let attempts = loop {
        rto *= 2;
        if rto > 60 { break rto; }
    };
    println!("final RTO: {attempts}");
}
```

### 演習 5 — FizzBuzz

1〜30をループし、3の倍数で「Fizz」、5の倍数で「Buzz」、両方で「FizzBuzz」、それ以外は数値を出力。
書けたら**`match` 版に書き換えてみてください**:

```rust
match (i % 3, i % 5) {
    (0, 0) => ...,
    ...
}
```

タプルに対するmatchはRustらしいイディオムです。

---

## 第6章 所有権 — Rustの核心

### 解説

Cで `malloc` した領域は「誰が `free` するか」を人間が追跡します。Rustはこれを**コンパイラが追跡**します。ルールは3つ:

1. 各値には**ちょうど1つのオーナー**（変数）がいる
2. オーナーがスコープを抜けると値は自動的に破棄される（`drop` = 自動`free`）
3. 代入・関数渡しで所有権は**ムーブ**する。ムーブ後の元変数は使用不可

```rust
fn main() {
    let s1 = String::from("hello");   // ヒープ確保
    let s2 = s1;                       // ムーブ（ポインタのコピー＋s1無効化）
    // println!("{s1}");               // error[E0382]: borrow of moved value

    let s3 = s2.clone();               // 深いコピー（明示的なmemcpy相当）
    println!("{s2} {s3}");

    let x = 5;
    let y = x;                         // i32はCopyトレイト実装済み→コピー
    println!("{x} {y}");               // 両方OK
}   // ここでs2, s3が自動drop。二重解放は構造的に不可能
```

Cの視点で言うと: ムーブは「浅いコピー＋元ポインタの無効化」です。
`free` 忘れ（リーク）、二重 `free`、use-after-free がすべて**コンパイルエラー**になります。

わざと `println!("{s1}")` のコメントを外し、`cargo check` のエラーメッセージを読んでください。
どこでムーブが起きたか矢印で示してくれます。

### 演習 6

1. `fn take_and_return(s: String) -> String` — 「Took: {s}」と出力して `s` を返す
2. main で `String::from("ownership")` を渡し、戻り値を受け取って再度出力
3. **実験**: 戻り値を受け取らずに元の変数を使おうとするとどんなエラーが出るか確認し、コメントとしてエラーコード（E0382など）をメモする

---

## 第7章 参照と借用 — `&T` / `&mut T`

### 解説

毎回所有権を返すのは面倒なので、**参照（借用）**を使います。Cのポインタに似ていますが、コンパイラが生存期間と排他性を検証します。

**借用ルール**（データ競合をコンパイル時に排除する仕組み）:
- 不変参照 `&T` は**同時に何個でも**OK
- 可変参照 `&mut T` は**同時に1つだけ**。しかも不変参照との共存も不可

```rust
fn print_len(s: &String) {          // 借用（所有権は移らない）
    println!("len: {}", s.len());
}

fn add_domain(s: &mut String) {
    s.push_str(".example.com");
}

fn main() {
    let mut host = String::from("router-01");
    print_len(&host);
    add_domain(&mut host);
    println!("{host}");              // router-01.example.com

    // 借用ルール違反を試す:
    let r1 = &host;
    let r2 = &host;                  // 不変参照は複数OK
    // let r3 = &mut host;           // error[E0502]!
    println!("{r1} {r2}");
}
```

Cのポインタとの違い: nullがない、ダングリングしない（コンパイラが保証）、
そして `&mut` の排他性によりエイリアシング起因のバグが原理的に発生しません。
`restrict` がデフォルトで強制される世界と考えると近いです。

### 演習 7

1. `fn string_info(s: &str)` — 長さと大文字版を出力（`&String` より `&str` を受けるのが慣習。理由を調べてみてください）
2. `fn append_exclaim(s: &mut String)` — 末尾に「!」を追加
3. **実験**: `let r = &host;` の直後に `append_exclaim(&mut host);` を書き、その後 `r` を使うとE0502が出ることを確認。`r` を使わなければ通ることも確認（NLL: Non-Lexical Lifetimes）

---

## 第8章 構造体 — `struct` / `impl`

### 解説

Cの `struct` ＋ 関数ポインタなしでメソッドを持てる形です。データ定義（`struct`）と振る舞い（`impl`）が分離しています。

- `&self` = 不変メソッド、`&mut self` = 可変メソッド、`self` = 所有権を消費
- `self` を取らない関数は**関連関数**（`String::from` と同じ）。コンストラクタは `new` という名前が慣習
- `#[derive(Debug)]` を付けると `{:?}` で出力可能になります

```rust
#[derive(Debug)]
struct Interface {
    name: String,
    ip: [u8; 4],
    mtu: u16,
    up: bool,
}

impl Interface {
    fn new(name: &str, ip: [u8; 4]) -> Self {
        Interface { name: name.to_string(), ip, mtu: 1500, up: false }
    }
    fn bring_up(&mut self) {
        self.up = true;
        println!("{}: link up", self.name);
    }
    fn ip_string(&self) -> String {
        format!("{}.{}.{}.{}", self.ip[0], self.ip[1], self.ip[2], self.ip[3])
    }
}

fn main() {
    let mut eth0 = Interface::new("eth0", [10, 0, 0, 1]);
    eth0.bring_up();
    println!("{} {} mtu {}", eth0.name, eth0.ip_string(), eth0.mtu);
    println!("{:?}", eth0);
}
```

### 演習 8

`NetworkDevice` 構造体を作ってください:

1. フィールド: `hostname: String`, `ip: String`, `port: u16`
2. `fn new(...) -> Self`
3. `fn display(&self)` — 「router-01 (192.168.1.1:80)」形式で出力
4. `fn is_well_known_port(&self) -> bool` — port < 1024
5. `fn change_port(&mut self, new_port: u16)` — ポート変更
6. main で生成→display→change_port(8080)→display→well-known判定を出力

---

## 第9章 列挙型と `match` — `enum` / `Option<T>`

### 解説

Rustの `enum` はCの `enum` とは別物で、**各バリアントがデータを持てる**代数的データ型です。Cで言えば「タグ付き `union` をコンパイラが安全に管理してくれる」ものです。

- `match` は**全バリアント網羅が必須**。漏れはコンパイルエラー（`switch` の `default` 忘れが存在しない）
- `Option<T>` は標準ライブラリの enum: `Some(T)` か `None`。**nullが言語に存在しない**ため、「値がないかもしれない」ことが型で強制されます

```rust
enum Packet {
    Tcp { src_port: u16, dst_port: u16 },
    Udp { src_port: u16, dst_port: u16 },
    Icmp { type_: u8 },
}

fn describe(p: &Packet) -> String {
    match p {
        Packet::Tcp { src_port, dst_port } =>
            format!("TCP {src_port} → {dst_port}"),
        Packet::Udp { dst_port: 53, .. } =>
            "UDP DNS query".to_string(),        // ガード的パターン
        Packet::Udp { src_port, dst_port } =>
            format!("UDP {src_port} → {dst_port}"),
        Packet::Icmp { type_: 8 } => "ICMP echo request".to_string(),
        Packet::Icmp { type_ }    => format!("ICMP type {type_}"),
    }
}

fn well_known_service(port: u16) -> Option<&'static str> {
    match port {
        22  => Some("ssh"),
        80  => Some("http"),
        443 => Some("https"),
        _   => None,
    }
}

fn main() {
    let p = Packet::Udp { src_port: 51234, dst_port: 53 };
    println!("{}", describe(&p));

    match well_known_service(8080) {
        Some(name) => println!("service: {name}"),
        None       => println!("unknown port"),
    }
}
```

### 演習 9

1. `enum Protocol { Tcp, Udp, Icmp }`
2. `fn default_port(p: &Protocol) -> Option<u16>` — Tcp→Some(80), Udp→Some(53), Icmp→None
3. main で3つ全部を配列に入れてループし、matchで「TCP: port 80」「ICMP: no port」形式で出力
4. **実験**: matchのアーム（腕）を1つ消して、コンパイラが何と言うか確認

---

## 第10章 エラー処理 — `Result<T, E>` / `?`

### 解説

Rustに例外はありません。エラーは戻り値で表現します。Cの「戻り値 -1 で errno を見る」方式の、型安全で無視できない版です。

- `Result<T, E>` = `Ok(T)` か `Err(E)` の enum
- **`Result` を無視するとコンパイラが警告**（`#[must_use]`）。Cのエラー戻り値握りつぶしが起きにくい
- `?` 演算子: `Err` なら即座に関数から return、`Ok` なら中身を取り出す。エラー伝播の定型が1文字に
- `unwrap()` / `expect("msg")` は `Err` でパニック。試作コード以外では避ける

```rust
use std::fs;
use std::num::ParseIntError;

fn parse_port(s: &str) -> Result<u16, ParseIntError> {
    let n = s.trim().parse::<u16>()?;   // 失敗ならErrを即return
    Ok(n)
}

fn read_config(path: &str) -> Result<String, std::io::Error> {
    let content = fs::read_to_string(path)?;
    Ok(content)
}

fn main() {
    for s in ["80", "abc", "99999"] {
        match parse_port(s) {
            Ok(p)  => println!("{s} → {p}"),
            Err(e) => println!("{s} → error: {e}"),
        }
    }

    if let Err(e) = read_config("/nonexistent") {
        println!("config error: {e}");
    }
}
```

### 演習 10 — 総仕上げ

`fn validate_port(s: &str) -> Result<u16, String>` を実装:

- パース失敗 → `Err("数値ではありません".to_string())`
  （ヒント: `s.parse::<u32>().map_err(|_| "...".to_string())?`）
- 0 → `Err("ポート0は無効".to_string())`
- 65535超 → `Err("範囲外".to_string())`
- 正常 → `Ok(port as u16)`

main で `["80", "99999", "abc", "443", "0"]` をループしてテスト。

---

## 卒業課題 — ミニプロジェクト3択

どれも標準ライブラリだけで書けます。1つ選んで（または全部）:

### A. IPv4サブネット計算機（CLI）

```bash
$ cargo run -- 192.168.1.130/26
Network:   192.168.1.128
Broadcast: 192.168.1.191
Hosts:     62
Range:     192.168.1.129 - 192.168.1.190
```

`std::env::args()` で引数を取り、文字列をパースして `u32` でビット演算。
第3章・第10章の知識が全部使えます。本業の知識で検算できるのが利点。

### B. 簡易ポートスキャナ

```rust
use std::net::TcpStream;
use std::time::Duration;

// TcpStream::connect_timeout でlocalhostの1〜1024をスキャン
```

`Result` の実践演習に最適。所有権・エラー処理・ループがすべて登場します。
（自分の管理下のホストに対してのみ実行してください）

### C. パケットヘッダパーサ

`[u8]` のバイト列（自分でハードコードしたEthernet/IPv4ヘッダ）を構造体にパースする。
TCP/IPスタック自作のC実装をRustに移植する足がかりになります。
エンディアン変換は `u16::from_be_bytes([b[0], b[1]])` が使えます。

---

## 次のステップ

- **rustlings** — 公式の穴埋め演習集。`cargo install rustlings` で導入、100問弱
- **The Book（日本語版）** — https://doc.rust-jp.rs/book-ja/ 本チュートリアルの各章に対応する詳細解説
- **Rust by Example** — https://doc.rust-lang.org/rust-by-example/
- 本チュートリアルで扱わなかった重要トピック: ライフタイム注釈、トレイト、ジェネリクス、`Vec`/`HashMap`、イテレータアダプタ、`Box`/`Rc`/`RefCell`。The Bookの10章以降で
- C実装済みのTCP/IPスタックの一部（チェックサム計算、ヘッダパースなど）をRustに移植してみるのは、両言語の差を体感する最良の教材です

**詰まったら**: `rustc --explain E0xxx`、`cargo clippy`、そしてエラーメッセージを最後まで読むこと。Rustコンパイラのエラーメッセージは業界最高水準の教材です。
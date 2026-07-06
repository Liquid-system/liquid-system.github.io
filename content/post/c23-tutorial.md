+++
title = "C23 Tutorial"
date = 2026-07-06T22:04:48+09:00
tags = ["tech"]
+++
# C23 実践チュートリアル — C99/C11経験者向け

C17までのCを書ける人が、C23（ISO/IEC 9899:2024）の差分を最短で身につけるためのチュートリアル。例はネットワークプログラミング寄りに書いてある。

---

## 0. 環境構築

### コンパイラ

| コンパイラ | 対応状況 |
|---|---|
| GCC 15+ | C23がデフォルト標準 |
| GCC 13/14 | `-std=c23`（13は`-std=c2x`） |
| Clang 18+ | `-std=c23` でほぼ完全対応 |
| MSVC | 部分対応（`/std:clatest`） |

```sh
# 確認
gcc -std=c23 -dM -E - < /dev/null | grep __STDC_VERSION__
# => #define __STDC_VERSION__ 202311L
```

### clangd（Neovim等のLSP）

プロジェクトルートに `compile_flags.txt`:

```
-std=c23
-Wall
-Wextra
```

CMakeなら `set(CMAKE_C_STANDARD 23)` と `CMAKE_EXPORT_COMPILE_COMMANDS ON` を設定して `compile_commands.json` を生成する。

---

## 1. キーワードの世代交代

### bool / true / false が予約語に

`<stdbool.h>` のincludeが不要になった。

```c
// C23: includeなしで動く
bool is_connected = false;
```

`_Bool` は互換のため残るが、新規コードでは `bool` を使う。

### nullptr と nullptr_t

`NULL` の型曖昧性（`0` か `(void*)0` か処理系依存）が解消された。

```c
int *p = nullptr;

// _Genericで正しく分岐できる
#define type_name(x) _Generic((x), \
    int: "int",                    \
    int *: "int *",                \
    nullptr_t: "nullptr_t")

type_name(nullptr);  // "nullptr_t"
type_name(NULL);     // 処理系依存で int か void* に落ちる
```

可変長引数関数に渡すときの安全性が特に改善される。`NULL` が `int` に展開される処理系では `execl()` 等に渡すとバグの温床だった。

### static_assert / thread_local / alignas / alignof

アンダースコア版（`_Static_assert` 等）のマクロラッパーが不要になり、小文字が第一級キーワードになった。

```c
static_assert(sizeof(struct tcp_header) == 20,
              "TCPヘッダはオプションなしで20バイト");
```

メッセージ引数は省略可能になった（C11では必須だった）。

```c
static_assert(sizeof(void*) == 8);
```

---

## 2. constexpr（オブジェクトのみ）

C++と違い、**関数には使えない**。コンパイル時定数のオブジェクトを定義できる。

```c
constexpr size_t MTU = 1500;
constexpr size_t MAX_PAYLOAD = MTU - 20 - 20;  // IP + TCPヘッダ

// 配列サイズに使ってもVLAにならない
uint8_t buf[MAX_PAYLOAD];
```

`#define` や `enum` ハックと違い、型を持ち、スコープに従い、デバッガから見える。定数定義は今後これが基本になる。

```c
// 従来
#define BACKLOG 128          // 型なし、スコープなし
enum { BACKLOG = 128 };      // intに限定

// C23
constexpr int BACKLOG = 128;
```

---

## 3. auto による型推論

初期化子から型を推論する。**変数宣言のみ**（関数の引数・戻り値には使えない）。

```c
auto sock = socket(AF_INET, SOCK_STREAM, 0);  // int
auto len  = sizeof(struct sockaddr_in);       // size_t

// マクロで威力を発揮する
#define SWAP(a, b) do {  \
    auto tmp_ = (a);     \
    (a) = (b);           \
    (b) = tmp_;          \
} while (0)
```

乱用すると読みにくくなるので、右辺から型が自明な場合に限定するのが無難。

## 4. typeof / typeof_unqual

GCC拡張だった `typeof` が標準化された。

```c
typeof(errno) saved = errno;

const volatile int reg = 0;
typeof(reg) a;         // const volatile int
typeof_unqual(reg) b;  // int（修飾子を剥がす）
```

ジェネリックなコンテナマクロや、既存変数と型を揃えたい場面で使う。

---

## 5. リテラルと初期化

### 2進リテラルと桁区切り

```c
uint8_t flags = 0b0001'1000;   // TCPフラグ: PSH|ACK
uint32_t mask = 0xFF'FF'FF'00; // /24
constexpr long budget = 1'000'000;
```

ビットフィールドやプロトコルフラグの定義が格段に読みやすくなる。

### 空初期化子 `= {}`

```c
struct sockaddr_in addr = {};  // 全メンバをゼロ初期化
uint8_t packet[1500] = {};
```

従来の `{0}` は「最初のメンバだけ明示的にゼロ、残りは暗黙」という歪な意味論だったが、`{}` はVLAにも使える正式なゼロ初期化になった。

---

## 6. 属性（C++11スタイル）

```c
[[nodiscard]] int tcp_send(int fd, const void *buf, size_t len);
[[deprecated("use tcp_send_v2")]] int tcp_send_old(int fd);

void parse_option(uint8_t kind) {
    switch (kind) {
    case 1:
        handle_nop();
        [[fallthrough]];  // 意図的なfall-through
    case 0:
        break;
    }
}

[[noreturn]] void fatal(const char *msg);

void handler(int sig, [[maybe_unused]] void *ctx);
```

`[[nodiscard]]` は戻り値のエラーチェック漏れをコンパイラ警告にできるので、システムコールのラッパーには全部付ける価値がある。

---

## 7. _BitInt(N) — 任意精度ビット幅整数

プロトコルフィールドを正確なビット幅で表現できる。

```c
// IPv4ヘッダのフィールド
unsigned _BitInt(4)  version = 4;
unsigned _BitInt(13) frag_offset;
unsigned _BitInt(3)  flags;

// リテラル接尾辞 wb / uwb
auto x = 6uwb;   // unsigned _BitInt(3)
```

通常の `int` と違い、指定した幅を超える演算は明確にラップする。ビットフィールドと違ってアドレスが取れ、配列にもできる。ただしABIはまだ処理系差があるので、ワイヤフォーマットの直接表現に使うのは慎重に。

---

## 8. stdckdint.h — チェック付き算術

オーバーフロー検出が標準化された。パケット長の計算に最適。

```c
#include <stdckdint.h>

bool alloc_packet(size_t header, size_t payload) {
    size_t total;
    if (ckd_add(&total, header, payload)) {
        return false;  // オーバーフロー
    }
    void *p = malloc(total);
    // ...
    return p != nullptr;
}
```

`ckd_add` / `ckd_sub` / `ckd_mul` の3つ。GCC/Clangの `__builtin_add_overflow` の標準版。長さフィールドを外部入力から受け取るネットワークコードでは、これを使わない理由がない。

---

## 9. #embed — バイナリの埋め込み

xxdで生成したCソースやリンカ芸が不要になる。

```c
static const uint8_t test_packet[] = {
    #embed "testdata/syn_packet.bin"
};

static const uint8_t cert[] = {
    #embed "ca-cert.der" if_empty(0x00)
};
```

テスト用のパケットキャプチャやTLS証明書をバイナリのままソースに取り込める。

---

## 10. enumの改善

### 基底型の指定

```c
enum tcp_state : uint8_t {
    TCP_CLOSED,
    TCP_LISTEN,
    TCP_SYN_SENT,
    TCP_ESTABLISHED,
};

static_assert(sizeof(enum tcp_state) == 1);
```

ワイヤフォーマットや状態テーブルでサイズを保証できる。

### intを超える列挙子

基底型指定がなくても、値が `int` に収まらなければ自動的に拡張される。

---

## 11. unreachable()

```c
#include <stddef.h>

int state_to_ttl(enum tcp_state s) {
    switch (s) {
    case TCP_CLOSED:      return 0;
    case TCP_LISTEN:      return 64;
    case TCP_SYN_SENT:    return 64;
    case TCP_ESTABLISHED: return 64;
    }
    unreachable();  // 到達したら未定義動作＝最適化ヒント
}
```

`__builtin_unreachable()` の標準版。デバッグビルドでは `assert(false)` に切り替えるマクロを組むのが実務的。

---

## 12. ライブラリの細かい追加

```c
// strdup / strndup が標準入り（POSIX から昇格）
char *copy = strdup(hostname);

// memccpy も標準入り
memccpy(dst, src, '\0', n);

// timegm 相当の timegm() も標準化
// free済みチェック用に memset_explicit()（最適化で消されないmemset）
memset_explicit(key_material, 0, sizeof key_material);
```

`memset_explicit` は秘密鍵の消去用。従来の `memset` はdead store eliminationで消される可能性があった。

---

## 13. 削除・変更されたもの

- **K&Rスタイル関数定義の削除**: `int f(a) int a; {...}` は違法に
- **`f()` が `f(void)` と同義に**: 「引数不定」の意味が消えた
- **符号付き整数は2の補数が必須に**: 1の補数・符号絶対値は削除
- **トライグラフ削除**
- **`va_start` の第2引数が省略可能に**: `va_start(ap)` と書ける（むしろ推奨）

`f()` の意味変更は既存コードの挙動に影響しうる数少ない非互換点なので、古いコードベースをC23に上げるときは注意。

---

## 14. 練習課題

1. 手元のプロジェクトの `#define` 定数を `constexpr` に置き換え、型エラーが出る箇所を確認する
2. 長さ計算をしている箇所を `stdckdint.h` でチェック付きにする
3. `switch` の状態遷移に enum基底型指定 + `unreachable()` を導入し、`-Wswitch` で網羅性をコンパイラに検査させる
4. テストデータの読み込みを `#embed` に置き換える
5. `NULL` を `nullptr` に一括置換し、`_Generic` で型が正しく取れることを確認する

---

## 参考資料

- N3220（C23最終草案に最も近い無料ドラフト）: open-std.org で入手可
- cppreference.com — 各機能に (C23) タグ付き
- Jens Gustedt, *Modern C* 3rd ed.（C23対応、著者サイトで無料PDF）

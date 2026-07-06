+++
title = "C23 Tutorial From Basics"
date = 2026-07-06T22:03:54+09:00
tags = ["tech"]
+++
# C23で学ぶC言語 — 基礎からの完全チュートリアル

最初からC23（ISO/IEC 9899:2024）で書くことを前提にした入門〜中級チュートリアル。古い書き方（`#include <stdbool.h>`、`NULL`、`{0}`初期化など）を経由せず、現代のCを最初から身につける。

---

## 第1章 環境構築と最初のプログラム

### コンパイラの準備

GCC 15以降ならC23がデフォルト。それ以前のGCC 13/14やClang 18以降では `-std=c23` を付ける。

```sh
gcc --version
gcc -std=c23 -Wall -Wextra -o hello hello.c
./hello
```

`-Wall -Wextra` は常に付ける習慣にする。コンパイラの警告は無料のコードレビューである。

### Hello, World

```c
#include <stdio.h>

int main(void) {
    printf("Hello, C23!\n");
    return 0;
}
```

- `#include <stdio.h>` — 標準入出力の宣言を取り込む
- `int main(void)` — プログラムの入口。C23では `main()` と書いても同義だが、明示する方が読みやすい
- `printf` — 書式付き出力
- `return 0;` — OSに正常終了を伝える。C23では `main` に限り省略可能

---

## 第2章 変数と型

### 基本の型

```c
int count = 42;          // 整数（通常32ビット）
double ratio = 3.14;     // 倍精度浮動小数点
char letter = 'A';       // 1文字（実体は小さい整数）
bool done = false;       // 真偽値 — C23では予約語。includeは不要
```

C23の重要な変化: `bool` / `true` / `false` は言語組み込みになった。古いコードにある `#include <stdbool.h>` はもう不要。

### サイズが保証された整数型

移植性が必要なら `<stdint.h>` の固定幅型を使う。

```c
#include <stdint.h>

uint8_t  byte   = 255;         // 8ビット符号なし
int32_t  offset = -100000;     // 32ビット符号付き
uint64_t big    = 1'000'000'000'000;  // C23: 桁区切り ' が使える
```

C23では数値リテラルに `'` で桁区切りを入れられる。2進リテラル `0b...` も標準化された。

```c
uint8_t flags = 0b0001'1000;
```

### 定数は constexpr

```c
constexpr int MAX_USERS = 100;
constexpr double PI = 3.14159265358979;

int scores[MAX_USERS];  // 配列サイズにも使える
```

従来の `#define MAX_USERS 100` と違い、型を持ち、スコープに従い、デバッガから見える。**新規コードの定数定義はconstexprが第一候補**。

### auto による型推論

```c
auto x = 42;        // int
auto y = 3.14;      // double
auto z = sizeof(x); // size_t
```

右辺から型が自明な場合に使う。乱用は可読性を下げる。

---

## 第3章 演算子

```c
int a = 7, b = 3;

a + b;   // 10
a - b;   // 4
a * b;   // 21
a / b;   // 2   ← 整数同士の除算は切り捨て
a % b;   // 1   ← 剰余

a / (double)b;  // 2.333... ← キャストで実数除算に
```

### 比較と論理

```c
a == b   // 等しい（= は代入。混同が古典的バグ）
a != b   // 等しくない
a && b   // かつ（短絡評価: 左がfalseなら右は評価されない）
a || b   // または
!a       // 否定
```

### ビット演算

```c
uint8_t x = 0b1100;
uint8_t y = 0b1010;

x & y    // 0b1000  AND
x | y    // 0b1110  OR
x ^ y    // 0b0110  XOR
~x       // ビット反転
x << 2   // 左シフト
x >> 1   // 右シフト
```

C23では符号付き整数は2の補数表現が保証されるようになり、ビット演算の挙動がより予測可能になった。

---

## 第4章 制御構文

### if / else

```c
if (temperature > 30) {
    printf("暑い\n");
} else if (temperature > 20) {
    printf("快適\n");
} else {
    printf("寒い\n");
}
```

### switch

```c
switch (grade) {
case 'A':
    printf("優秀\n");
    break;              // breakを忘れると次のcaseに落ちる
case 'B':
    printf("良好\n");
    [[fallthrough]];    // C23属性: 意図的なfall-throughを明示
case 'C':
    printf("合格\n");
    break;
default:
    printf("不合格\n");
}
```

`[[fallthrough]]` はC23で標準化された属性。「breakを書き忘れたのか、わざとなのか」をコンパイラと人間の両方に伝える。

### ループ

```c
// for: 回数が決まっているとき
for (int i = 0; i < 10; i++) {
    printf("%d\n", i);
}

// while: 条件で回すとき
while (retry_count < 3) {
    retry_count++;
}

// do-while: 最低1回は実行したいとき
do {
    input = read_input();
} while (input != 'q');
```

`break` でループ脱出、`continue` で次の周回へ。

---

## 第5章 関数

```c
// 宣言（プロトタイプ）
int add(int a, int b);

int main(void) {
    int sum = add(3, 4);
    printf("%d\n", sum);
    return 0;
}

// 定義
int add(int a, int b) {
    return a + b;
}
```

### C23での重要な変更: f() の意味

```c
void f();      // C17まで: 「引数は不定」  C23: 「引数なし」= f(void)
```

C23では `f()` と `f(void)` が同義になった。歴史的な罠がひとつ消えた。

### 戻り値を無視させない

```c
[[nodiscard]] int open_file(const char *path);
```

`[[nodiscard]]` を付けると、戻り値を捨てた呼び出しにコンパイラが警告を出す。エラーコードを返す関数には積極的に付ける。

---

## 第6章 配列と文字列

### 配列

```c
int scores[5] = {90, 85, 70, 60, 95};
int zeros[100] = {};   // C23: 空の {} で全要素ゼロ初期化

for (size_t i = 0; i < 5; i++) {
    printf("%d\n", scores[i]);
}
```

`= {}` はC23で正式に導入された。従来の `{0}` より意図が明確。

配列は自分の長さを知らない。長さは別に管理するのがCの流儀。

```c
constexpr size_t N = 5;
int data[N] = {};
```

### 文字列

Cの文字列は「`'\0'`（ヌル文字）で終わるchar配列」である。

```c
char name[] = "Alice";   // {'A','l','i','c','e','\0'} の6バイト

#include <string.h>
size_t len = strlen(name);        // 5（\0は数えない）
char buf[16] = {};
strncpy(buf, name, sizeof(buf) - 1);  // 常に上限付きでコピー
```

C23では `strdup`（複製をmallocして返す）が標準入りした。

```c
char *copy = strdup(name);
// 使い終わったら free(copy);
```

---

## 第7章 ポインタ

Cの核心。変数の「メモリ上の住所」を扱う。

```c
int x = 42;
int *p = &x;    // & : xのアドレスを取る

printf("%d\n", *p);   // * : アドレスの先の値を読む → 42
*p = 100;             // アドレスの先に書く
printf("%d\n", x);    // 100
```

### nullptr — C23の新しい空ポインタ

```c
int *q = nullptr;   // 「どこも指していない」

if (q != nullptr) {
    *q = 1;   // nullptrの参照剥がしは未定義動作。必ずチェック
}
```

従来の `NULL` はマクロで型が曖昧だったが、`nullptr` は専用の型 `nullptr_t` を持つ。新規コードでは `nullptr` を使う。

### ポインタと配列

```c
int arr[3] = {10, 20, 30};
int *p = arr;        // 配列名は先頭要素へのポインタに変換される

p[1];      // 20
*(p + 2);  // 30 — p[i] は *(p+i) の糖衣構文
```

### 関数に配列を渡す

```c
int sum(const int *data, size_t n) {
    int total = 0;
    for (size_t i = 0; i < n; i++) {
        total += data[i];
    }
    return total;
}

// 呼び出し側
int nums[4] = {1, 2, 3, 4};
sum(nums, 4);
```

読むだけなら `const` を付ける。「この関数は書き換えない」という契約になる。

---

## 第8章 構造体と列挙型

### struct

```c
struct point {
    double x;
    double y;
};

struct point p = {.x = 1.0, .y = 2.0};  // 指示付き初期化子
struct point origin = {};                // 全メンバゼロ

p.x = 5.0;

struct point *ptr = &p;
ptr->y = 3.0;   // ポインタ経由は -> を使う
```

### typedef で名前を短く

```c
typedef struct {
    char name[32];
    int age;
} User;

User u = {.name = "Bob", .age = 30};
```

### enum — C23で基底型が指定可能に

```c
enum status : uint8_t {   // C23: サイズを保証できる
    STATUS_OK,
    STATUS_RETRY,
    STATUS_FATAL,
};

static_assert(sizeof(enum status) == 1);
```

switchと組み合わせるときは全列挙子をcaseに書き、`-Wswitch` 警告で網羅性をコンパイラに検査させる。

```c
const char *status_name(enum status s) {
    switch (s) {
    case STATUS_OK:    return "ok";
    case STATUS_RETRY: return "retry";
    case STATUS_FATAL: return "fatal";
    }
    unreachable();   // C23: ここには来ないと宣言（<stddef.h>）
}
```

---

## 第9章 動的メモリ

```c
#include <stdlib.h>

int *data = malloc(100 * sizeof(int));
if (data == nullptr) {
    return 1;   // 確保失敗は必ずチェック
}

data[0] = 42;

free(data);
data = nullptr;   // 解放後はnullptrにしておくと二重解放を防ぎやすい
```

### 鉄則

1. `malloc` したら必ず `free` する（1対1対応）
2. `free` 後のポインタは使わない（use-after-free）
3. 同じポインタを2回 `free` しない
4. 確保失敗（nullptr）を常にチェック

### オーバーフローを防いだサイズ計算 — C23

外部入力からサイズを計算するときは `<stdckdint.h>` を使う。

```c
#include <stdckdint.h>

void *alloc_items(size_t count, size_t item_size) {
    size_t total;
    if (ckd_mul(&total, count, item_size)) {
        return nullptr;   // 乗算がオーバーフローした
    }
    return malloc(total);
}
```

`ckd_add` / `ckd_sub` / `ckd_mul` の3つ。セキュリティ上重要な計算では必須級。

### AddressSanitizerで検証する習慣

```sh
gcc -std=c23 -g -fsanitize=address -o app app.c
```

メモリバグは実行時に即座に検出される。開発中は常時有効にしてよい。

---

## 第10章 ファイル入出力

```c
#include <stdio.h>

FILE *fp = fopen("data.txt", "r");
if (fp == nullptr) {
    perror("fopen");
    return 1;
}

char line[256];
while (fgets(line, sizeof(line), fp) != nullptr) {
    printf("%s", line);
}

fclose(fp);
```

モード: `"r"` 読み込み / `"w"` 書き込み（上書き）/ `"a"` 追記。バイナリなら `"rb"` `"wb"`。

書き込みは `fprintf(fp, ...)` や `fwrite`。開いたファイルは必ず `fclose` する（メモリと同じ規律）。

---

## 第11章 プリプロセッサとC23の新機能

### 基本

```c
#include <stdio.h>     // ヘッダ取り込み
#define DEBUG 1        // ただし定数はconstexprを優先

#if DEBUG
    printf("debug build\n");
#endif
```

### #embed — バイナリファイルの埋め込み（C23）

```c
static const uint8_t logo[] = {
    #embed "logo.png"
};
```

画像やテストデータをビルド時にそのまま配列へ。xxdで変換する時代は終わった。

### typeof — 式から型を得る（C23）

```c
int x = 10;
typeof(x) y = 20;   // int y = 20;

#define SWAP(a, b) do { \
    auto tmp_ = (a);    \
    (a) = (b);          \
    (b) = tmp_;         \
} while (0)
```

---

## 第12章 実践: 小さなプログラムを書く

学んだ要素を組み合わせた例。単語の出現回数を数える。

```c
#include <stdio.h>
#include <stdlib.h>
#include <string.h>

constexpr size_t MAX_WORDS = 1000;
constexpr size_t WORD_LEN  = 64;

typedef struct {
    char word[WORD_LEN];
    int count;
} Entry;

[[nodiscard]] static Entry *find_or_add(Entry *table, size_t *n,
                                        const char *word) {
    for (size_t i = 0; i < *n; i++) {
        if (strcmp(table[i].word, word) == 0) {
            return &table[i];
        }
    }
    if (*n >= MAX_WORDS) {
        return nullptr;
    }
    Entry *e = &table[*n];
    strncpy(e->word, word, WORD_LEN - 1);
    e->count = 0;
    (*n)++;
    return e;
}

int main(void) {
    static Entry table[MAX_WORDS] = {};
    size_t n = 0;
    char word[WORD_LEN];

    while (scanf("%63s", word) == 1) {
        Entry *e = find_or_add(table, &n, word);
        if (e == nullptr) {
            fprintf(stderr, "テーブル満杯\n");
            return 1;
        }
        e->count++;
    }

    for (size_t i = 0; i < n; i++) {
        printf("%s: %d\n", table[i].word, table[i].count);
    }
    return 0;
}
```

ここに含まれるC23要素: `constexpr` 定数、`= {}` 初期化、`nullptr`、`[[nodiscard]]`。

---

## 第13章 次のステップ

1. **エラー処理の設計** — 戻り値 vs errno vs out-parameter の使い分け
2. **ヘッダとソースの分割** — 複数ファイルのビルド、includeガード
3. **Makefileの基礎** — 手動コンパイルからの卒業
4. **デバッガ（gdb/lldb）** — printfデバッグからの卒業
5. **より深いC23** — `_BitInt(N)`、`memset_explicit`、属性の網羅
6. **システムプログラミング** — ソケット、ファイルディスクリプタ、シグナル

### 古いコードを読むための対応表

世の中のCコードの大半はC23以前なので、読み替えを知っておく。

| 古い書き方 | C23 |
|---|---|
| `#include <stdbool.h>` + `bool` | `bool`（include不要） |
| `NULL` | `nullptr` |
| `= {0}` | `= {}` |
| `#define MAX 100` | `constexpr int MAX = 100;` |
| `_Static_assert(...)` | `static_assert(...)` |
| `__typeof__(x)`（GCC拡張） | `typeof(x)` |
| `/* fall through */` コメント | `[[fallthrough]]` |

---

## 参考資料

- Jens Gustedt, *Modern C* 3rd ed. — C23対応、著者サイトで無料PDF
- cppreference.com — 機能ごとに (C23) タグ付きのリファレンス
- N3220 — C23最終草案に最も近い無料ドラフト（open-std.org）

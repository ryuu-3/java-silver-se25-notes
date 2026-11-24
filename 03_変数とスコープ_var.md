# 03 変数とスコープ・var

## ローカル変数のスコープ

ローカル変数は宣言した位置から、そのブロック `{}` の終わりまで有効。

### スコープが重なる同名変数は宣言できない

```java
int i = 100;
for (int i = 0; i < 3; i++) { }  // ❌ i はすでに宣言済み

void m(int x) {
    int x = 10;                  // ❌ 引数と同名のローカル変数は不可
}
```

### スコープが重ならなければ同名で OK

```java
for (int i = 0; i < 3; i++) { }
for (int i = 0; i < 3; i++) { }  // ✅ 前の i のスコープは終わっている
```

### フィールドとの同名は OK（シャドーイング）

```java
class A {
    int x = 1;
    void m() {
        int x = 2;               // ✅ フィールドを隠すだけ
        System.out.println(this.x); // → 1
    }
}
```

## var（ローカル変数型推論、Java 10〜）

初期化式の型から、コンパイラがローカル変数の型を推論する。

### 使える場所 / 使えない場所

| 場所 | 可否 |
|------|------|
| ローカル変数（初期化あり） | ✅ |
| 拡張 for / 通常 for のループ変数 | ✅ |
| try-with-resources のリソース | ✅ |
| ラムダ式の引数（Java 11〜） | ✅ |
| フィールド | ❌ |
| メソッドの引数・戻り値の型 | ❌ |
| catch の引数 | ❌ |

### コンパイルエラーになる例

```java
var a;                    // ❌ 初期化式がない
var b = null;             // ❌ 型を推論できない
var c = {1, 2, 3};        // ❌ 配列初期化子だけでは不可
var d = new int[]{1, 2};  // ✅ これは OK
var e = 1, f = 2;         // ❌ 複数の変数を一度に宣言できない
var g = () -> 1;          // ❌ ラムダ式は代入先の型（ターゲット型）が必要
var h = String::length;   // ❌ メソッド参照も同様
```

### var とダイヤモンド演算子

型引数を推論する手がかりがないため、`Object` が使われる。

```java
var list = new ArrayList<>();   // ArrayList<Object> と推論される
list.add(1);
list.add("A");                  // ✅ どちらも Object として追加できる
```

### var は予約型名（キーワードではない）

```java
var var = 10;   // ✅ 変数名としては使える
class var {}    // ❌ 型名としては使えない
```

### ラムダ式の引数での var（Java 11〜）

```java
BiFunction<Integer, Integer, Integer> f1 = (var x, var y) -> x + y; // ✅
BiFunction<Integer, Integer, Integer> f2 = (var x, y) -> x + y;     // ❌ 混在不可
Function<Integer, Integer> f3 = var x -> x;                          // ❌ 括弧が必須
```

> var を使うと引数にアノテーションを付けられる：`(@NonNull var x) -> ...`

## effectively final（実質的に final）

`final` を付けていなくても、**初期化後に一度も再代入されない**ローカル変数のこと（Java 8〜）。

ラムダ式・匿名クラス・ローカルクラスの内部から参照できるローカル変数は、`final` または effectively final に限られる。

```java
int n = 10;
Runnable r = () -> System.out.println(n);  // ✅ n は effectively final

int m = 10;
m++;                                        // 再代入しているので effectively final ではない
Runnable r2 = () -> System.out.println(m); // ❌
```

⚠️ **ひっかけ**：ラムダ式の**後ろ**で再代入しても、effectively final ではなくなりコンパイルエラーになる。

```java
int k = 1;
Runnable r3 = () -> System.out.println(k); // ❌
k = 2;
```

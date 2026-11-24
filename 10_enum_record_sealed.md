# 10 enum・record・sealed

## enum（列挙型）

```java
enum Size {
    S(1), M(2), L(3);           // 定数は先頭に書く（最後のセミコロンは他のメンバがあれば必須）

    private final int value;
    Size(int value) { this.value = value; }   // コンストラクタ
    int value() { return value; }
}
```

### enum の特徴

- 暗黙的に `java.lang.Enum` を継承する → **他のクラスを `extends` できない**（インターフェースの `implements` は可）。
- コンストラクタは **`private` または修飾子なし**のみ（修飾子なしでも暗黙的に `private`）。`public` / `protected` は ❌。
- `new` でインスタンス化できない。各定数は JVM によって1回だけ生成される。
- enum 定数ごとにメソッドの実装を変えられる（定数固有のクラス本体）。

### 初期化の順序

**enum 定数が宣言順にインスタンス化された後**に、static 初期化子・static フィールドの初期化が実行される。

```java
enum E {
    A, B;
    static { System.out.println("static"); }
    E() { System.out.println("constructor " + name()); }
}
// E.A を最初に使うと → constructor A, constructor B, static
```

> そのため、enum のコンストラクタから static フィールド（定数以外）を参照するとコンパイルエラーになる。

### 主なメソッド

| メソッド | 説明 |
|---------|------|
| `values()` | 全定数の配列（宣言順） |
| `valueOf(String)` | 名前から定数を取得。存在しないと 💥 `IllegalArgumentException` |
| `name()` | 定数名 |
| `ordinal()` | 宣言順の番号（**0 から**） |
| `compareTo()` | `ordinal()` の順で比較 |

### switch での enum

```java
switch (size) {
    case S -> ...;        // ✅ 定数名だけで書ける
    case Size.M -> ...;   // ✅ Java 21 からは修飾名も可
    default -> ...;
}
```

## record（Java 16〜）

不変のデータを保持するためのクラスを簡潔に定義できる。

```java
public record Person(String name, int age) {}
```

### 自動生成されるもの

| 生成されるもの | 内容 |
|-------------|------|
| フィールド | `private final String name;` `private final int age;` |
| 標準コンストラクタ | `Person(String name, int age)` |
| アクセサ | `name()` / `age()`（`getName()` ではない） |
| `equals()` / `hashCode()` | 全コンポーネントで比較 |
| `toString()` | `Person[name=Alice, age=20]` の形式 |

### record の制約

- 暗黙的に `final` クラス → 継承できない。
- 暗黙的に `java.lang.Record` を継承 → 他のクラスを `extends` できない（インターフェースの `implements` は可）。
- **インスタンスフィールドを追加できない**（static フィールドは可）。
- **インスタンス初期化ブロックを書けない**。
- 型パラメータを持てる：`record Pair<A, B>(A first, B second) {}`
- ネストした record は暗黙的に `static`。

### ボディに書けるもの

| 書けるもの | 書けないもの |
|----------|------------|
| static フィールド・static メソッド | インスタンスフィールド |
| インスタンスメソッド | インスタンス初期化ブロック |
| コンストラクタ（標準・コンパクト・その他） | |
| ネストした型 | |

### コンパクトコンストラクタ

引数リストを省略した形の標準コンストラクタ。**引数の検証や正規化**に使う。

```java
public record Person(String name, int age) {
    public Person {                       // 引数リストなし
        if (age < 0) throw new IllegalArgumentException();
        name = name.strip();              // ✅ 引数（パラメータ）への再代入
        // this.name = name;              // ❌ コンパイルエラー（フィールドへの代入は不可）
    }   // ← 最後にフィールドへの代入が自動で行われる
}

System.out.println(new Person("  Al ", 3));  // → Person[name=Al, age=3]
```

⚠️ **ひっかけ**：コンパクトコンストラクタでは `this.name = ...` と書くとコンパイルエラー。フィールドへの代入はコンパイラが最後に自動で行う。

- 標準（正規）コンストラクタは、record 自体と同じかそれ以上に広いアクセス修飾子が必要。

### その他のコンストラクタ

標準コンストラクタ以外のコンストラクタは、先頭で必ず `this(...)` を呼ぶ。

```java
record Person(String name, int age) {
    Person(String name) { this(name, 0); }   // ✅
}
```

### アクセサのカスタマイズ

自動生成されるアクセサと同じシグネチャのメソッドを定義すると、そちらが使われる（自動生成されない）。

```java
record Person(String name, int age) {
    public String name() { return name.toUpperCase(); }  // public が必須
}
```

### with メソッドパターン

record は不変なので、一部だけ変えたいときは新しいインスタンスを作る。

```java
record Person(String name, int age) {
    Person withAge(int newAge) { return new Person(name, newAge); }
}
```

## sealed クラス / インターフェース（Java 17〜）

継承（実装）できるサブクラスを限定する。

```java
public sealed class Shape permits Circle, Square, Polygon {}

final class Circle extends Shape {}           // これ以上継承不可
sealed class Square extends Shape permits Cube {}  // さらに制限付きで継承可
final class Cube extends Square {}
non-sealed class Polygon extends Shape {}     // 制限を解除（誰でも継承可）
```

### ルール

- 許可された**直接のサブクラス**は、必ず `final` / `sealed` / `non-sealed` のいずれかを付ける（record と enum は暗黙的に `final` 扱い）。
- サブクラスは sealed クラスと**同じモジュール**（無名モジュールなら**同じパッケージ**）に置く。
- 同じソースファイル内にサブクラスがすべてあれば `permits` は省略できる。
- `permits` に書いたクラスは、実際にその sealed クラスを直接継承していなければならない。

### sealed とパターンマッチング

sealed 型を switch で網羅すると `default` が不要になる。

```java
sealed interface Animal permits Dog, Cat {}
record Dog() implements Animal {}
record Cat() implements Animal {}

String sound(Animal a) {
    return switch (a) {
        case Dog d -> "ワン";
        case Cat c -> "ニャー";
    };   // ✅ すべて網羅しているので default 不要
}
```

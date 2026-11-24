# 13 Stream API

## Stream の構成

```java
List<String> result = list.stream()     // 1. 生成
        .filter(s -> s.length() > 2)    // 2. 中間操作（0回以上）
        .map(String::toUpperCase)
        .toList();                      // 3. 終端操作（1回）
```

## 中間操作は遅延評価

中間操作は**終端操作が呼ばれるまで実行されない**。

```java
Stream.of(1, 2, 3).peek(System.out::println);           // 何も表示されない（終端操作がない）
Stream.of(1, 2, 3).peek(System.out::println).toList();  // → 1 2 3
```

⚠️ **ひっかけ**：`peek()` はデバッグ用。終端操作があっても実行が保証されない場合がある。
Java 9 以降、`count()` はソースから要素数が分かる場合、パイプラインを実行せずに結果を返すことがある。

```java
long n = Stream.of(1, 2, 3).peek(System.out::println).count();
// → 何も表示されず、n = 3
```

## Stream は一度しか使えない

終端操作を実行した Stream は「消費済み」になり、再度操作すると 💥 `IllegalStateException`。

```java
Stream<String> s = Stream.of("a", "b");
s.forEach(System.out::println);
s.count();   // 💥 IllegalStateException
```

## 主な中間操作

| メソッド | 説明 |
|---------|------|
| `filter(Predicate)` | 条件に合う要素だけ残す |
| `map(Function)` | 要素を変換 |
| `mapToInt(ToIntFunction)` | `IntStream` に変換 |
| `flatMap(Function)` | 要素を Stream に変換して平坦化 |
| `distinct()` | 重複を除去（`equals()` で判定） |
| `sorted()` / `sorted(Comparator)` | 並べ替え |
| `limit(n)` / `skip(n)` | 先頭 n 個 / 先頭 n 個を飛ばす |
| `peek(Consumer)` | 要素を確認（デバッグ用） |

## 主な終端操作

| メソッド | 戻り値 | 説明 |
|---------|-------|------|
| `forEach(Consumer)` | `void` | 各要素を処理 |
| `count()` | `long` | 要素数（`int` ではない） |
| `toList()`（Java 16〜） | `List<T>` | **変更不可**のリスト |
| `collect(Collector)` | 任意 | `Collectors` と組み合わせる |
| `reduce(...)` | 下記参照 | 要素を1つの値にまとめる |
| `min` / `max(Comparator)` | `Optional<T>` | |
| `findFirst()` / `findAny()` | `Optional<T>` | |
| `anyMatch` / `allMatch` / `noneMatch` | `boolean` | 短絡評価される |

⚠️ **ひっかけ**：空の Stream では `allMatch` と `noneMatch` は `true`、`anyMatch` は `false`。

### reduce

| 形式 | 戻り値 |
|------|-------|
| `reduce(BinaryOperator<T>)` | `Optional<T>`（初期値なし。空なら `Optional.empty()`） |
| `reduce(T identity, BinaryOperator<T>)` | `T`（初期値あり。空なら identity） |
| `reduce(U identity, BiFunction, BinaryOperator)` | `U` |

```java
Optional<Integer> a = Stream.of(1, 2, 3).reduce(Integer::sum);  // → Optional[6]
int b = Stream.of(1, 2, 3).reduce(0, Integer::sum);             // → 6
```

## IntStream などのプリミティブストリーム

```java
IntStream.range(1, 5);        // 1, 2, 3, 4（終わりを含まない）
IntStream.rangeClosed(1, 5);  // 1, 2, 3, 4, 5（終わりを含む）

int sum = IntStream.of(1, 2, 3).sum();                     // int
OptionalDouble avg = IntStream.of(1, 2, 3).average();      // OptionalDouble
OptionalInt max = IntStream.of(1, 2, 3).max();             // OptionalInt
Stream<Integer> boxed = IntStream.of(1, 2).boxed();        // Stream<Integer> に戻す
```

### 戻り値が Optional になるメソッド / ならないメソッド

**空の Stream でも答えを1つに決められるかどうか**で決まる。

| メソッド | 空の Stream での結果 | 戻り値の型 |
|---------|-------------------|----------|
| `sum()` | `0`（何も足さなければ 0） | `int` |
| `count()` | `0` | `long` |
| `average()` | 決まらない（0 ÷ 0 になる） | `OptionalDouble` |
| `max()` / `min()` | 決まらない（比べる要素がない） | `OptionalInt` |

- `sum()` は要素が1つもなくても「合計は 0」と言えるので、値が必ずある → `Optional` で包む必要がない。
- `average()` は 0 を返すと「平均が 0 だった」と区別がつかないため、「値がないかもしれない」ことを `OptionalDouble` で表す。

```java
IntStream.empty().sum();       // → 0
IntStream.empty().average();   // → OptionalDouble.empty
IntStream.empty().max();       // → OptionalInt.empty
```

`sum()` は**初期値 0 のある `reduce`** と同じ考え方（→ 上の「reduce」の表）。

```java
int a = IntStream.of(1, 2, 3).reduce(0, Integer::sum);        // 初期値あり → int
OptionalInt b = IntStream.of(1, 2, 3).reduce(Integer::sum);   // 初期値なし → OptionalInt
```

⚠️ **ひっかけ**

- 型名は `OptionalInteger` ではなく **`OptionalInt`**（ほかに `OptionalLong`、`OptionalDouble`）。
- `average()` は `IntStream` でも **`OptionalDouble`** を返す（平均は小数になるため）。
- `Stream<Integer>` には `sum()` がない。`mapToInt(Integer::intValue)` などで `IntStream` に変換してから呼ぶ。
- `IntStream.sum()` の戻り値は `int` なので、合計が大きいと例外にならずに回り込む（オーバーフロー）。

```java
Stream.of(1, 2, 3).sum();                          // ❌ Stream<Integer> に sum() はない
Stream.of(1, 2, 3).mapToInt(Integer::intValue).sum();  // ✅ → 6
```

## Collectors

```java
List<String> list = List.of("apple", "banana", "cherry", "avocado");

list.stream().collect(Collectors.joining(", "));             // → apple, banana, cherry, avocado
list.stream().collect(Collectors.groupingBy(s -> s.charAt(0)));
// → {a=[apple, avocado], b=[banana], c=[cherry]}
list.stream().collect(Collectors.partitioningBy(s -> s.length() > 5));
// → {false=[apple], true=[banana, cherry, avocado]}
list.stream().collect(Collectors.toMap(s -> s, String::length));
```

⚠️ **ひっかけ**：`Collectors.toMap()` でキーが重複すると 💥 `IllegalStateException`（マージ関数を第3引数に渡せば回避できる）。

## Optional

```java
Optional<String> o = Optional.of("A");
Optional<String> e = Optional.empty();
Optional.of(null);          // 💥 NullPointerException
Optional.ofNullable(null);  // ✅ Optional.empty()

e.orElse("default");        // → default
e.orElseGet(() -> "gen");   // → gen
e.get();                    // 💥 NoSuchElementException
e.orElseThrow();            // 💥 NoSuchElementException（Java 10〜）
o.isPresent();              // → true
e.isEmpty();                // → true（Java 11〜）
o.ifPresent(System.out::println);
```

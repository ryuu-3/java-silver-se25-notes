# 16 日付と時刻 API（java.time）

## 主なクラス

| クラス | 内容 | 例 |
|-------|------|----|
| `LocalDate` | 日付 | `2025-09-16` |
| `LocalTime` | 時刻 | `10:15:30` |
| `LocalDateTime` | 日付＋時刻 | `2025-09-16T10:15:30` |
| `ZonedDateTime` | タイムゾーン付き日時 | `2025-09-16T10:15:30+09:00[Asia/Tokyo]` |
| `Instant` | UTC の時点（タイムスタンプ） | |
| `Period` | **日付ベース**の期間（年・月・日） | `P1Y2M3D` |
| `Duration` | **時間ベース**の期間（時・分・秒・ナノ秒） | `PT1H30M` |

## 共通の特徴

- **不変（イミュータブル）**。`plusDays()` などは新しいインスタンスを返す。
- `new` ではなく `of()` / `now()` / `parse()` で生成する。
- 月は **1 始まり**（`Month.JANUARY` = 1）。
- 存在しない日付は 💥 `DateTimeException`。

```java
LocalDate d = LocalDate.of(2025, 1, 31);
d.plusDays(1);                          // 戻り値を受け取らないと d は変わらない
System.out.println(d);                  // → 2025-01-31
System.out.println(d.plusMonths(1));    // → 2025-02-28（月末に調整される）
LocalDate.of(2025, 2, 30);              // 💥 DateTimeException
LocalDate.of(2025, 13, 1);              // 💥 DateTimeException
```

## Duration と Period の文字列表記（ISO-8601）

| 表記 | 意味 |
|------|------|
| `PT1H30M` | 1時間30分（Duration） |
| `PT45.5S` | 45.5秒（Duration） |
| `P2DT3H` | 2日と3時間（Duration。1日 = 24時間として扱い、`PT51H` になる） |
| `P1Y2M3D` | 1年2か月3日（Period） |

- **時間の部分は `T` の後ろ**に `H`（時）、`M`（分）、`S`（秒）で書く。`T` の前の `M` は「月」。
- `Duration.parse("PT1H30M")`、`Period.parse("P1Y2M3D")`

⚠️ **ひっかけ**：`Period.ofYears(1).ofMonths(2)` の結果は `P2M`。`ofXxx()` は **static メソッド**なので、前の呼び出し結果は捨てられる。

```java
System.out.println(Period.ofYears(1).ofMonths(2));  // → P2M
System.out.println(Period.of(1, 2, 0));             // → P1Y2M
```

## 期間の計算

```java
LocalDate start = LocalDate.of(2025, 1, 1);
LocalDate end   = LocalDate.of(2025, 1, 10);

ChronoUnit.DAYS.between(start, end);   // → 9（end は含まない）
Period.between(start, end);            // → P9D
start.until(end, ChronoUnit.DAYS);     // → 9
```

- `ChronoUnit.DAYS.between(start, end)` は **end を含まない**（半開区間）。
- `Duration.between()` に `LocalDate` を渡すと 💥 `UnsupportedTemporalTypeException`（時間の情報がないため）。

## DateTimeFormatter のパターン文字

| 文字 | 意味 | 例 |
|------|------|----|
| `y` / `u` | 年 | `yyyy` → 2025 |
| `M` | **月** | `MM` → 09、`MMM` → Sep |
| `d` | 日 | `dd` → 05 |
| `H` | 時（**24時間制** 0〜23） | `HH` → 14 |
| `h` | 時（**12時間制** 1〜12） | `hh` → 02 |
| `m` | **分** | `mm` → 07 |
| `s` | 秒 | `ss` → 09 |
| `a` | 午前/午後 | AM / PM |
| `E` | 曜日 | Tue |

⚠️ **ひっかけ**：`MM` は月、`mm` は分。`HH` は24時間制、`hh` は12時間制。

```java
var f = DateTimeFormatter.ofPattern("yyyy/MM/dd HH:mm");
System.out.println(LocalDateTime.of(2025, 9, 16, 14, 5).format(f));  // → 2025/09/16 14:05
LocalDate.of(2025, 9, 16).format(DateTimeFormatter.ofPattern("HH:mm"));
// 💥 UnsupportedTemporalTypeException（LocalDate には時刻がない）
```

## タイムゾーン

```java
Set<String> ids = ZoneId.getAvailableZoneIds();   // 戻り値は Set<String>
ZoneId tokyo = ZoneId.of("Asia/Tokyo");
ZonedDateTime z = ZonedDateTime.now(tokyo);
```

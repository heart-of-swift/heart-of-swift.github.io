---
layout: page
---

{% assign chapter=site.data.book.chapters[0] %}

## Value Semantics を持たない場合の問題と対応策

_Value Semantics_ とは何かわかったところで、次に、 _Value Semantics_ を持っていると何がうれしいのかを見ていきます。

### Value Semantics を持たない場合に起こりがちな問題

ここでは、 _Value Semantics_ を持たない場合の問題として次の二つを例に取り上げます。

- 意図しない変更
- 整合性の破壊

#### 意図しない変更の例

何かのサービスのアイテムを考えます。アイテムというのは、たとえばフォトアプリなら写真、 TODO アプリなら TODO の項目です。

今、アイテムが次のような `Item` クラスで表されているとします。

```swift
class Item {
    var name: String
    ...
    var settings: Settings
}
```

`settings` はアイテムの設定を表すプロパティです。 `Settings` 型は次のようなクラスとして宣言されています。

```swift
class Settings {
    var isPublic: Bool
    ...
}
```

`isPublic` は、このアイテムが公開されているものかを表すプロパティです。

この `Settings` クラスは _Value Semantics_ を持たず _Reference Semantics_ を持っています。このとき、どのような問題が起こり得るかを見てみましょう。

アイテムの複製は様々なアプリでよく行われる処理です。上記の `Item` クラスを使ってこれを実装するコードを考えてみます。今、 `Item` 型のインスタンスが変数 `item` に格納されているとして、複製の処理を次のように書いたとします。

```swift
let duplicated: Item = Item(
    name: item.name,
    ...
    settings: item.settings
)
```

複製されたアイテムは `duplicated` に格納されます。このとき、複製されたアイテムを公開したいとして、次のような処理を書くと何が起こるでしょうか。

```swift
duplicated.settings.isPublic = true
```

今、 `Item` クラスは _Reference Semantics_ を持っているので、この変更はオリジナルの `item` にも波及します。複製されたアイテムだけを公開したかったのに、オリジナルも公開されてしまいました。これは意図しない挙動で、望ましいものではありません。原因は、 `Item` インスタンスを複製するときにディープコピーしなければならなかったのに、シャローコピーをしてしまったことです。

_Value Semantics_ を持たないと、ちょっとしたコーディング時のミスや設計時の考慮漏れによってこのような問題を引き起こしがちです。

#### 整合性の破壊の例

次に、整合性の破壊の例を見てみます。

姓（ `familyName` ）と名（ `firstName` ）を持ったシンプルな `Person` クラスを考えてみます。

```swift
class Person {
    var firstName: String
    var familyName: String
}
```

姓と名を連結してフルネームを返すのはよくある処理なので、この `Person` クラスに `fullName` プロパティを追加します。 `fullName` は _Computed Property_ で、ただ `firstName` と `familyName` を連結して返すだけです。

```swift
class Person {
    var firstName: String
    var familyName: String
    var fullName: String { // 👈
        "\\(firstName) \\(familyName)"
    }
}
```

しかし、 `fullName` にアクセスする度に新しい文字列を生成するのは無駄です。そこで、最初に `fullName` にアクセスされたときに生成された文字列をキャッシュしておいて、二度目以降はその文字列を返すことにします。たとえば、次のようなコードで実装できます。

```swift
private var _fullName: String?
var fullName: String {
    if let fullName = _fullName {
        return fullName
    }
    _fullName = "\\(firstName) \\(familyName)"
    return _fullName!
}
```

また、 `firstName` や `familyName` が変更されたときにキャッシュとの整合性が崩れてしまうので、 `firstName` や `familyName` が変更されたときにはキャッシュを破棄するようにします。

```swift
var firstName: String {
    didSet { _fullName = nil }
}

var familyName: String {
    didSet { _fullName = nil }
}
```

これで、 `Person` クラスのキャッシュは適切に機能するようになりました。

さて、このとき次のような処理を考えてみます。

```swift
let person: Person = Person(
    firstName: "Taylor",
    familyName: "Swift"
)

var familyName: String = person.familyName
familyName.append("y")
```

このとき、 `familyName` は `String` インスタンスで、 `String` は _Value Semantics_ を持つ `struct` なので、 `append` による変更は `person.familyName` には影響を影響を与えません。この処理の後でも、 `person.familyName` の値は `"Swift"` のままです。

しかし、もし `String` が _Reference Semantics_ を持ったクラスだったらどのような問題を引き起こすかを考えてみましょう。

まず、 `String` が _Reference Semantics_ を持っていたら、 `familyName` と `person.familyName` は同じ `String` インスタンスを共有していることになるので、 `familyName.append("y")` の結果、 `person.familyName` も `"Swifty"` に書き換えられてしまいます。しかし、これ自体は一概に悪いことだとは言えません。

問題になるのは、 `fullName` のキャッシュが存在する場合です。もし、 `fullName` のキャッシュに `"Taylor Swift"` という文字列が保存されていると、 `familyName` は `"Swifty"` に更新されたのに `fullName` は `Taylor Swift` のままということになってしまいます。

```swift
print(person.fullName)   // "Taylor Swift"

var familyName: String = person.familyName
familyName.append("y")

print(person.familyName) // "Swifty"
print(person.fullName)   // "Taylor Swift" 😵
```

`Person` インスタンスの状態の整合性が破壊されてしまいました。状態変更時にはキャッシュをクリアするように設計したのに、何が問題だったのでしょうか。

もしこれが `person.familyName` を直接 `"Swifty"` に更新したのであれば、適切にキャッシュクリアされて問題は起こりませんでした。

```swift
print(person.fullName)   // "Taylor Swift"

person.familyName = "Swifty")

print(person.familyName) // "Swifty"
print(person.fullName)   // "Taylor Swifty" 😄
```

問題が起こったのは、 `firstName` プロパティや `familyName` プロパティを直接変更するという以外に、 `String` のメソッドを介して間接的に内部状態が変更されるという、想定しない変更のパスが存在したことです。そのようなパスを見落とすと、この例のように内部的な整合性が破壊されてしまうことがあります。

### 問題への対応策

このような問題は、特に _参照型_ 中心の言語では起こりがちなので、対応策も考えられています。よく取られる策に、次の三つがあります。

- 防御的コピー
- Read-only View
- イミュータブルクラス

しかし、 Swift は _値型_ 中心の言語で、 _値型_ を用いることで問題に対処しています。

- 値型

ここでは、イミュータブルクラスと _値型_ について取り上げて比較し、値型の優位性を見てみます。

#### イミュータブルクラスによる問題の解決と新たな問題

[前に見たように](/value-semantics/)イミュータブルクラスは _Value Semantics_ を持ちます。そのため、 _Value Semantics_ を持たないことによって引き起こされる問題に対応するには、イミュータブルクラスを導入することは直接的な解決策となります。しかし、イミュータブルクラスも万能ではなく、別の問題を引き起こします。それらを順に見てましょう。

##### イミュータブルクラスによる問題の解決

まずは、先の二つの問題がどのように解決されるのかを見てみます。

一つ目は「意図しない変更」の例です。この例については、 `Settings` クラスがイミュータブルクラスだったとすると、複製されたアイテムの `isPublic` プロパティを変更しようとするコードがコンパイルエラーになります。

```swift
let duplicated: Item = Item(
    name: item.name,
    ...
    settings: item.settings
)

duplicated.settings.isPublic = true // ⛔
```

`Settings` クラスはイミュータブルなので、 `isPublic` プロパティを変更することはできません。変更するためには、 `isPublic` が `true` な新たな `Settings` インスタンスを生成して `duplicated.settings` に代入する必要があります。

```swift
duplicated.settings = Settings(isPublic: true, ...)
```

この場合、 `duplicated.settings` に代入されるのはまったく新しい `Settings` インスタンスなので、これが `item.settings` に影響を与えることはありません。 `Settings` は _Value Semantics_ を持っているので、アイテムを複製するときに `settings` プロパティまでディープコピーする必要もなくなりました。

同じように、「整合性の破壊」の例もイミュータブルクラスによって解決されます。

```swift
var familyName: String = person.familyName
familyName.append("y") // ⛔
```

今、 `String` がイミュータブルクラスだとすると、このような `append` メソッドを実装することはできません。よって、このメソッドを介して `Person` インスタンスの内部状態が変更され、整合性が破壊されることもありません。

実際、 Java や Kotlin など、参照型中心の言語の多くでは `String` はイミュータブルクラスとして実装されています。それは、 _Value Semantics_ を持たないことに起因するこのような問題を防止するためです。 Objective-C の `NSString` クラスのインスタンスもイミュータブルですが、残念ながら `NSString` には `NSMutableString` というサブクラスが存在し、 `NSString` 型としてはイミュータブルにならないので、 `NSMutableString` を使って上記のような問題を引き起こすことができます。

##### イミュータブルクラスによって引き起こされる問題

さて、イミュータブルクラスで問題が解決できるなら、何でもイミュータブルクラスにすれば良いのでしょうか。そう簡単にはいきません。イミュータブルクラスは _Value Semantics_ を持たないことによる問題を解決してくれますが、変更が容易でないという別の問題を引き起こします。例を見てみましょう。

何らかのサービスのユーザーが `User` クラスで表されているとします。ユーザーはそのサービスで利用できるポイントを持っており、 `User` クラスの `points` プロパティで表されています。`User` クラスがミュータブルクラスとして、ユーザーに 100 ポイント付与するコードは次のように書けます。

```swift
// ミュータブルクラスの場合
let user: User = ...

user.points += 100
```

しかし、イミュータブルクラスのインスタンスは変更することができません。そのため、 `points` だけを 100 増やすことはできません。状態を変更するためには、 `points` 以外はまったく同じで `points` だけが 100 増えた新しい `User` インスタンスを生成し、変数 `user` に格納されているインスタンスをその新しいインスタンスに差し替える必要があります。

```swift
// イミュータブルクラスの場合
var user: User = ...

user = User(
    id: user.id,
    name: user.name,
    ...
    points: user.points + 100,
    ...
)
```

これは、コードとしても複雑ですし、たった一つのプロパティを更新するためだけにインスタンスを丸ごと作り直す必要があるのでパフォーマンスもよくありません。

_参照型_ 中心の言語の中には、コードの複雑さを軽減するための特殊な構文や言語仕様を提供しているものもあります。たとえば、 Kotlin の `data class` を使えば `copy` メソッドが自動生成され、目的のプロパティだけを変更した新しいインスタンスを生成することができます。

```kotlin
// Kotlin
// イミュータブルクラスの場合
var user: User = ...

user = user.copy(points = user.points + 100)
```

しかし、それでもミュータブルクラスの場合と比べると複雑ですし、ネストしたプロパティを更新しようとするとさらに大変です。

```kotlin
// Kotlin
// ミュータブルクラスの場合
group.owner.points += 100
```

```kotlin
// Kotlin
// イミュータブルクラスの場合
group = group.copy(
    owenr = group.owner.copy(
        points = group.owner.points + 100
    )
)
```

このように、 _Value Semantics_ を持たないことによる問題はイミュータブルクラスによって解決できますが、状態の変更が複雑になるという新たな問題が生まれてしまいました。すべてをイミュータブルにすると利益が不利益に見合わない場合が多く、 _参照型_ 中心の言語では大抵ミュータブルクラスとイミュータブルクラスがバランスを取って使い分けられています。そして、ミュータブルクラスを採用する場合には、 _Value Semantics_ を持たないことによる問題に対処するために、防御的コピーや Read-only View が取り入れられています。

#### 値型

_値型_ であれば簡単に _Value Semantics_ を実現することができます。「意図しない変更」や「整合性の破壊」の例も、それぞれ `Settings` と `String` が `struct` で _Value Semantics_ を持っていれば何の問題も起こりません。

さらに、 _値型_ には、ミュータブルクラスのように変更が容易であるという特徴があります。先の `User` にポイントを付与するコードも、 `User` が `struct` なら次のように書けます。

```swift
// 値型の場合
var user: User = ...

user.points += 100
```

`user.points += 100` というのはミュータブルクラスにおけるコードと全く同じです。しかも、この `User` は _Value Semantics_ も持っています。

つまり、 **_値型_ はミュータブルクラスの持つ変更の容易さと、イミュータブルクラスの持つ _Value Semantics_ のいいとこ取りをしたような存在だと考えられる** わけです。

Swift の標準ライブラリで提供される型は、ほぼすべてが _値型_ です。次に挙げるような、ぱっと思い付くような型は全部 _値型_ です。

- `Int`, `Float`, `Double`, `Bool`
- `String`, `Character`
- `Array`, `Dictionary`, `Set`
- `Optional`, `Result`

もちろん、ここで挙げたすべての型が _Value Semantics_ を持っています（ただし、型パラメータを持つ型は、型パラメータに _Value Semantics_ を持つ型が指定された場合のみ）。

これが _Value Semantics_ 関連の問題に対する Swift の戦略で、 _Value Semantics_ を持った _値型_ でコードの大部分を構成することで、 _Value Semantics_ を欠くことの問題と、ミュータブルクラスと同等の変更の容易さを実現しようとしているのです。

### まとめ

_Value Semantics_ を持たない型は、「意図しない変更」や「整合性の破壊」のような問題を引き起こしがちです。それに対処するために、 _防御的コピー_ や _Read-only View_ 、 _イミュータブルクラス_ の使用のような方法が考えられて来ましたが、一長一短があります。 _イミュータブルクラス_ は _Value Semantics_ を持つので、「意図しない変更」や「整合性の破壊」などは防げますが、変更が容易でないという問題を生みます。

_値型_ を使えば _Value Semantics_ を実現しながら変更も容易であり、 _ミュータブルクラス_ と _イミュータブルクラス_ のいいとこ取りをしたような存在だと言うことができます。 Swift の標準ライブラリの型はほぼすべてが _値型_ であり、そのような _値型_ の特性を活かしたコードを書くことが可能です。

---

{% assign next_section=chapter.sections[2] %}
{% assign prev_section=chapter.sections[0] %}

- 次のページ: {% include section-link.md section=next_section %}
- 前のページ: {% include section-link.md section=prev_section %}
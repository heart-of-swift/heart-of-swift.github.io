---
layout: section
chapter_index: 0
section_index: 1
---

_Value Semantics_ とは何かわかったところで、次に、 _Value Semantics_ を持っていると何がうれしいのかを見て行きましょう。

## Value Semantics を持たない型が起こしがちな問題

_Value Semantics_ の利点を知るために、 _Value Semantics_ を持たない型がどのような問題を引き起こすのかを見てみます。ここでは、 _Value Semantics_ を持たない型が起こしがちな問題の例として次の二つを取り上げます。

- 意図しない変更
- 整合性の破壊

### 意図しない変更の例

あるサービスのアイテムを考えます。アイテムというのは、たとえば、そのサービスがフォトアプリなら写真、メモアプリなら個々のメモです。

今、アイテムが次のような `Item` クラスで表されているとします。

```swift
class Item {
    var name: String
    ...
    var settings: Settings
}
```

`settings` はアイテムの設定を表すプロパティです。 `Settings` は次のようなクラスです。

```swift
class Settings {
    var isPublic: Bool
    ...
}
```

`isPublic` は、このアイテムが公開されているかを表すプロパティです。

この `Settings` クラスは _Value Semantics_ を持たず _Reference Semantics_ だけを持っています。このとき、どのような問題が起こり得るかを見てみましょう。

アイテムの複製は様々なアプリでよく行われる処理です。上記の `Item` クラスを使ってそれを実装するコードを考えてみます。今、 `Item` 型のインスタンスが変数 `item` に格納されているとして、複製の処理を次のように書いたとします。

```swift
let item: Item = ...

let duplicated: Item = Item(
    name: item.name,
    ...
    settings: item.settings
)
```

複製されたアイテムは定数 `duplicated` に格納されます。このとき、複製されたアイテムを公開するために次のようなコードを書くと何が起こるでしょうか。

```swift
duplicated.settings.isPublic = true
```

今、 `Settings` クラスは _Reference Semantics_ を持っているので、この変更はオリジナルの `item` の `settings` にも波及します。複製されたアイテムだけを公開したかったのに、オリジナルも公開されてしまいました。これは意図しない挙動で、望ましいものではありません。原因は、 `Item` インスタンスを複製するときに `Settings` インスタンスも複製しないといけなかった（ _ディープコピー_ しなければならなかった）のに、 _シャローコピー_ してしまったことです。

_Value Semantics_ を持たないと、ちょっとしたコーディング時のミスや設計時の考慮漏れによってこのような問題を引き起こしがちです。

### 整合性の破壊の例

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
        "\(firstName) \(familyName)"
    }
}
```

しかし、 `fullName` にアクセスする度に新しい文字列を生成するのは無駄です。そこで、最初に `fullName` にアクセスされたときに生成された文字列をキャッシュしておいて、二度目以降はその文字列を返すことにします。たとえば、次のようなコードで実装できます。

```swift
private var _fullName: String? // キャッシュ
var fullName: String {
    if let fullName = _fullName {
        return fullName
    }
    _fullName = "\(firstName) \(familyName)"
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

このとき、変数 `familyName` に格納されるのは `String` インスタンスで、 `String` は _Value Semantics_ を持つ `struct` です。変更に対して独立なので、 `append` による変更は `person.familyName` に影響を影響を与えません。この処理の後には、変数 `familyName` の値は `"Swifty"` になりますが、 `person.familyName` の値は `"Swift"` のままです。

しかし、もし仮に `String` が _Reference Semantics_ を持ったクラスだったらどうなるか考えてみましょう（ `append` メソッドは自身の状態を変更するので、このとき `String` は _ミュータブルクラス_ となり _Value Semantics_ を持ちません）。

`String` が _Reference Semantics_ を持っていたら、変数 `familyName` と `person.familyName` は同じ `String` インスタンスを共有することになります。そのため、 `familyName.append("y")` の結果、変数 `familyName` の値も `person.familyName` の値も `"Swifty"` に書き換えられます。これ自体は一概に悪いこととは言えません。

問題になるのは、 `fullName` のキャッシュが存在する場合です。もし、 `fullName` のキャッシュとして `"Taylor Swift"` という文字列が保存されているとどうなるでしょうか。 `familyName` は `"Swifty"` に更新されたのに `fullName` は `Taylor Swift` のままということになってしまいます。

```swift
print(person.fullName)   // "Taylor Swift"

var familyName: String = person.familyName
familyName.append("y")

print(person.familyName) // "Swifty"
print(person.fullName)   // "Taylor Swift" 😵
```

`Person` インスタンスの状態の整合性が破壊されてしまいました。状態変更時にはキャッシュをクリアするように設計したのに、何が問題だったのでしょうか。

`person.familyName` を直接 `"Swifty"` に更新する場合は、キャッシュが適切にクリアされて問題は起こりません。

```swift
print(person.fullName)   // "Taylor Swift"

person.familyName = "Swifty"

print(person.familyName) // "Swifty"
print(person.fullName)   // "Taylor Swifty" 😄
```

問題が起こった原因は、 `firstName` プロパティや `familyName` プロパティなどの `Person` の API を使わずに、 `String` のメソッドを介して間接的に内部状態が変更されるという想定しない変更のパスが存在したことです。 _Reference Semantics_ はそのようなパスを作りやすいので、設計時にそれを見落とすと、この例のように内部的な整合性が破壊されてしまうことがあります。

## 問題への対処法

このような問題は、特に _参照型_ 中心の言語ではよく起こるので、様々な対処法が考えられています。よく取られる対処法として次の三つがあります。

- _防御的コピー_
- _Read-only View_
- _イミュータブルクラス_

Swift は _参照型_ 中心ではなく _値型_ 中心の言語です。上記とは異なり、 _値型_ を用いることで問題に対処しています。

- 値型

ここでは、 _イミュータブルクラス_ と _値型_ について取り上げて比較し、 _値型_ の優位性を見てみます。

### イミュータブルクラス

[前に見たように](/value-semantics/) _イミュータブルクラス_ は _Value Semantics_ を持ちます。そのため、 _イミュータブルクラス_ の導入は、 _Value Semantics_ を持たない型が起こす問題を直接的に解決できます。しかし、 _イミュータブルクラス_ も万能ではなく、別の問題を引き起こします。それらを順に見てましょう。

#### イミュータブルクラスによる問題の解決

まずは、先の二つの問題が _イミュータブルクラス_ によってどのように解決されるのかを見てみます。

一つ目は「意図しない変更」の例です。 `Settings` クラスが _イミュータブルクラス_ なら、複製されたアイテムの `isPublic` プロパティを変更しようとするコードはコンパイルエラーになり、意図しない変更が防止できます。

```swift
let duplicated: Item = Item(
    name: item.name,
    ...
    settings: item.settings
)

duplicated.settings.isPublic = true // ⛔
```

`Settings` クラスは _イミュータブル_ なので、 `isPublic` プロパティを変更することはできません。変更するためには、 `isPublic` が `true` である新たな `Settings` インスタンスを生成し、 `duplicated.settings` に代入する必要があります。

```swift
duplicated.settings = Settings(isPublic: true, ...)
```

この場合、 `duplicated.settings` に代入されるのはまったく新しい `Settings` インスタンスです。 `item.settings` と共有された `Settings` インスタンスを変更したわけではないので、 `item.settings` に影響を与えることはありません。

同じように、「整合性の破壊」の例も _イミュータブルクラス_ によって解決されます。

```swift
var familyName: String = person.familyName
familyName.append("y") // ⛔
```

今、 `String` が _イミュータブルクラス_ だとすると、自身の状態を変更する `append` メソッドを持つことはできません。よって、 `append` メソッドを介して `Person` インスタンスの内部状態が変更され、整合性が破壊されることもありません。

実際、 Java や [Kotlin](https://kotlinlang.org) など、参照型中心の言語の多くでは `String` は _イミュータブルクラス_ として実装されています。それは、ここで挙げたような問題を防止するためです。 Foundation の `NSString` クラスのインスタンスも _イミュータブル_ ですが、残念ながら `NSString` には `NSMutableString` というサブクラスが存在します。 `NSString` 型としては _イミュータブル_ にならないので、 `NSMutableString` を使って上記のような問題を引き起こすことができます。

#### イミュータブルクラスによって引き起こされる問題

_イミュータブルクラス_ で問題が解決できるなら何でも _イミュータブルクラス_ にすれば良いかというと、そう簡単な話ではありません。 _イミュータブルクラス_ は変更が容易でないという別の問題を引き起こします。例を見てみましょう。

何らかのサービスのユーザーが `User` クラスで表されるとします。ユーザーはそのサービスで利用できるポイントを持っており、 `User` クラスの `points` プロパティで表されます。

```swift
// ミュータブルクラスの場合
class User {
    let id: Int
    var name: String
    ...
    var points: Int
    ...
}
```

このように `User` が _ミュータブルクラス_ なら、ユーザーに 100 ポイント付与するコードは次のように簡潔に書けます。

```swift
// ミュータブルクラスの場合
let user: User = ...

user.points += 100
```

しかし、 `User` が _イミュータブルクラス_ だとこのように簡単にはいきません。

```swift
// イミュータブルクラスの場合
final class User {
    let id: Int
    let name: String
    ...
    let points: Int
    ...
}
```

_イミュータブルクラス_ のインスタンスの状態を変更することはできないので、 100 ポイント付与したくても `User` インスタンスの `points` だけを増やすことはできません。状態を変更するためには、 `points` 以外はまったく同じで `points` だけが 100 増えた新しい `User` インスタンスを生成し、変数 `user` の値をその新しいインスタンスに差し替える必要があります。

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

このように、 _イミュータブルクラス_ を用いると _ミュータブルクラス_ と比べて状態の変更が複雑です。 _イミュータブルクラス_ は _Value Semantics_ を持たないことに起因する問題を解決してくれますが、状態の変更が複雑になるため、すべてを _イミュータブル_ にすると利益が不利益に見合わないことが多いです。そのため、 _参照型_ 中心の言語では _ミュータブルクラス_ と _イミュータブルクラス_ を利益のバランスを取って適切に使い分ける必要があります。

### 値型

前述のように、 Swift は _イミュータブルクラス_ ではなく _値型_ によって問題の解決を図ります。 _値型_ であれば簡単に _Value Semantics_ を持たせることができます。 _Value Semantics_ を持たせるという意味では、 _イミュータブルクラス_ と同じ方向性を持った対処法です。

「意図しない変更」の例は、 `Settings` が _Value Semantics_ を持った `struct` であれば何の問題も起こりません。

```swift
// 値型の場合
let duplicated: Item = item // 複製
duplicated.settings.isPublic = true // item.settings.isPublic は変更されない😄
```

「整合性の破壊」の例についても、変数 `familyName` に対する変更は `person` には何の影響も及ぼさないため、整合性が破壊されることもありません。

```swift
// 値型の場合
print(person.fullName)   // "Taylor Swift"

var familyName: String = person.familyName
familyName.append("y")

print(person.familyName) // "Swift"
print(person.fullName)   // "Taylor Swift"
```

加えて、 _値型_ には、 _ミュータブルクラス_ のように変更が容易であるという特徴があります。先の `User` にポイントを付与するコードも、 `User` が `struct` なら次のように書けます。

```swift
// 値型の場合
struct User {
    let id: Int
    var name: String
    ...
    var points: Int
    ...
}
```

```swift
// 値型の場合
var user: User = ...

user.points += 100
```

`user.points += 100` というのは _ミュータブルクラス_ におけるコードと全く同じです。しかも、この `User` は _Value Semantics_ も持っています。

つまり、 **_値型_ は _ミュータブルクラス_ の持つ変更の容易さと、 _イミュータブルクラス_ の持つ _Value Semantics_ のいいとこ取りをしたような存在だと考えられる** わけです。

Swift の標準ライブラリで提供される型は、ほぼすべてが _値型_ です。次に挙げるぱっと思い付くような型は全部 _値型_ です。

- `Int`, `Float`, `Double`, `Bool`
- `String`, `Character`
- `Array`, `Dictionary`, `Set`
- `Optional`, `Result`

もちろん、ここで挙げたすべての型が _Value Semantics_ を持っています（ただし、型パラメータを持つ型は、型パラメータに _Value Semantics_ を持つ型が指定された場合のみ _Value Semantics_ を持ちます）。

## まとめ

_Value Semantics_ を持たない型は、「意図しない変更」や「整合性の破壊」などの問題を引き起こしがちです。それに対処するために、 _防御的コピー_ や _Read-only View_ 、 _イミュータブルクラス_ の使用など方法が考えられてきました。しかし、それらには一長一短があり、たとえば _イミュータブルクラス_ には変更が容易でないという問題があります。

_値型_ を使えば簡単に _Value Semantics_ を実現でき、変更も容易です。 _値型_ は、 _ミュータブルクラス_ と _イミュータブルクラス_ のいいとこ取りをしたような存在だと言えます。 Swift の標準ライブラリの型はほぼすべてが _Value Semantics_ を持つ _値型_ であり、問題の回避と変更の容易さを両立できます。

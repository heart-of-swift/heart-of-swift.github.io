---
layout: page
title: Value Semantics
---

## Value Semantics とは

Swift における **_Value Semantics_** については WWDC 2015 のセッション ["Building Better Apps with Value Types in Swift"](https://developer.apple.com/videos/play/wwdc2015/414/) で詳しく説明されていますが、残念ながらその中では _Value Semantics_ の定義については述べられていません。 Swift における _Value Semantics_ の定義は、 swift リポジトリの中のドキュメント ["Value Semantics in Swift"](https://github.com/apple/swift/blob/master/docs/proposals/ValueSemantics.rst) に記載されています。

これは 201３ 年に書かれた古いドキュメントで、 [docs/proposals](https://github.com/apple/swift/blob/master/docs/proposals) という正式に認められていないドキュメントが収められたディレクトリの中にありますが、 Swift Core Team の Dave Abrahams さんがその著者であり、内容も WWDC で話されていることと一貫性があるので、信頼のおけるドキュメントだと考えて良いと思われます。

このドキュメントの中に、 Value Semantics の定義は次のように書かれています。

> For a type with value semantics, variable initialization, assignment, and argument-passing each create an independently modifiable copy of the source value that is interchangeable with the source.
>
> （参考訳）ある型が Value Semantics を持っているとき、その型の変数を初期化したり、値を代入したり、引数に渡したりしたときに、元の値のコピーが作られて、そのコピーと元の値は独立に変更することができる。つまり、どちらかを変更してももう片方には影響を与えない。
>
> "Value Semantics in Swift", https://github.com/apple/swift/blob/master/docs/proposals/ValueSemantics.rst

コードを使って例を挙げてみます。

### Value Semantics を持つ例

`value` という Stored Property を一つだけ持つシンプルな `struct` 、 `Foo` を考えてみます。

```swift
struct Foo {
    var value: Int = 0
}
```

この `Foo` を使って次のような処理を考えます。

```swift
var a = Foo()
var b = a
a.value = 2
```

このときに、 `b.value` も変更されるかというのが問題です。結論から言うと、この例では `Foo` は **_値型_** なので、 `a.value` を変更しても `b.value` されません。

_値型_ のインスタンスは変数に直接格納されています。インスタンスを表すバイト列が、メモリ上のその変数を表す領域に直接書かれているわけです。変数に代入したときには、そのバイト列がそのままコピーされて書き込まれます。これが、 _値型_ において代入時にインスタンスがコピーされる理由です。

上記の例では、 `var b = a` をした時点で `a` の領域に格納されたバイト列が `b` の領域にコピーされるわけです。代入直後の時点では、 `a` と `b` には同じ内容のバイト列が書かれていますが、内容が同じ別々の `Foo` インスタンスを表しています。 `a.value` を変更しても `a` と `b` は異なるインスタンスを格納しているので、 `b.value` が変更されることはありません。

今、 `a` と `b` は変更に対して独立である、つまり、 `a` と `b` のどちらかに変更を加えてももう一方には影響を及ぼさないので、 `Foo` は _Value Semantics_ を持っていると言えます。

### Value Semantics を持たない例

次に、 Value Semantics を持たない例を見てみます。先程のコードの `struct` だった部分を `class` に変更します。それ以外はまったく同じです。

```swift
class Foo {
    var value: Int = 0
}

var a = Foo()
var b = a
a.value = 2
```

この場合は、 `Foo` はクラスで **_参照型_** なので、 `a.value` を変更することで `b.value` も変更されてしまいます。

_参照型_ のインスタンスは変数に直接格納されません。インスタンスの実体を表すバイト列はメモリ上の別の領域（ヒープ領域のどこか）に格納されていて、その領域を表すメモリのアドレスが変数に格納されます。たとえば、そのアドレスが `0x123DEF` だとすると、変数に実際に格納されているのは `0x123DEF` というアドレスです。 `var b = a` では、その `0x123DEF` というアドレスが `a` から `b` へコピーされます。このとき、 `a` と `b` は同じアドレス `0x123DEF` を介して同一の `Foo` インスタンスを参照しているので、 `a.value` を変更すると `b.value` も変更されてしまうわけです。

この例では変更に対する独立性を持たないので、 `Foo` は _Value Semantics_ を持ちません。 

このように、片一方を変更するともう一方にも影響を与えるケースは、 _Value Semantics_ と対比して、 **_Reference Semantics_** を持っていると言われます。

### Semantics vs Type

先の例や名前からも推測できる通り、 _Value Semantics_ は _値型（ Value Type ）_ と、 _Reference Semantics_ は _参照型（ Reference Type ）_ と深い関係があります。しかし、これらは同じものではないので注意が必要です。

- _Value Semantics_ ≠ _値型（ Value Type ）_
- _Reference Semantics_ ≠ _参照型（ Reference Type ）_

たとえば、 **_値型_ だけど _Value Semantics_ を持たない型** や、 **_参照型_ だけど _Reference Semantics_** を持つ型も存在します。 _Value Semantics_ / _Reference Semantics_ と、 _値型_ / _参照型_ をきちんと区別して考えることが重要です。

### 値型だけど Value Semantics を持たない例

Semantics と Type を区別して考えるために、値型だけど _Value Semantics_ を持たない例を見てみます。

`struct Foo` に加えて、 `Bar` というクラスを導入します。

```swift
class Bar {
    var value: Int = 0
}
```

この `Bar` 型のプロパティを `Foo` に追加します。

```swift
struct Foo {
    var value: Int = 0
    var bar: Bar = Bar() // 👈
}
```

これを使って先程と似たようなことをしてみます。ただし、 `a.value` に加えて、最後に `a.bar.value` も変更します。

```swift
var a = Foo()
var b = a
a.value = 2
a.bar.value = 3 // 👈
```

このとき、 `Foo` は _値型_ なので `a` と `b` には独立した別々の `Foo` インスタンスが格納されます。しかし、 `Bar` は _参照型_ なので、 `a` と `b` の `bar` プロパティには同じ `Bar` インスタンスのアドレスが格納され、そのアドレスを介して同じインスタンスを参照していることになります。

その状態で `a.value` を変更しても `a` と `b` には異なる `Foo` インスタンスが格納されているので `b.value` には影響を与えません。しかし、 `a.bar` と `b.bar` は同じ `Bar` インスタンスを参照しているので、 `a.bar.value` に変更を加えると `b.bar.value` も変更されます。

そのため、 `Foo` は _値型_ であるにも関わらず変更に対する独立性を持たない、つまり _Value Semantics_ を持たないことになります。なお、 `a.value` と `b.value` は独立なので、この `Foo` は _Reference Type_ も持ちません。

_Value Semantics_ も _Reference Semantics_ も持たない型は扱いづらく、そのような型を作ってしまわないように注意する必要があります。次のようなクラスのインスタンスを _値型_のプロパティに持たせると、 _Value Semantics_ も _Reference Semantics_ も持たない型を作ってしまうことになりがちなので注意が必要です。

- `UILabel`, `UISwitch`
- `AVAudioPlayer`
- `CMMotionManager`

### 参照型プロパティを持つ値型だけど Value Semantics を持つ例

一見、先の例は _参照型_ のプロパティを持ったことが _Value Semantics_ を失った原因のように思えます。次は、 _参照型_ のプロパティを持つけれども _Value Semantics_ を持つ例を見てみます。

先のコードの `Bar` クラスをイミュータブルクラスに変更します。

```swift
struct Foo {
    var value: Int = 0
    var bar: Bar = Bar()
}

final class Bar {
    let value: Int = 0
}
```

`Bar` クラスをイミュータブルにするために、 `Bar` の `value` プロパティを `let` にして、 `final class` に変更します。 `final class` にするのは、 `Bar` のミュータブルなサブクラスが作られてしまうと `Bar` 型のイミュータビリティが破壊されてしまうからです。

この `Foo` と `Bar` を使って先程と同じことをしてみます。

```swift
var a = Foo()
var b = a
a.value = 2
a.bar.value = 3 // ⛔
```

今、 `Bar` はイミュータブルクラスなので、当然 `a.bar.value` を変更しようとするとコンパイルエラーになります。たしかに、 `Bar` は _参照型_ で `a.bar` と `b.bar` は同じインスタンスを参照していますが、 `Bar` はイミュータブルなので、そのインスタンスを通して状態を変更することはできません。

そのため、 `Foo` インスタンスは変更に対する独立性を持っているということになり、 _参照型_ のプロパティを持つにも関わらず、 `Foo` は _Value Semantics_ を持つということになります。

このようなケースはよく見られ、たとえば次のようなクラスのプロパティを持っても _Value Semantics_ を破壊する原因にはなりません。

- `NSNumber`, `NSNull`
- `UIImage`
- `KeyPath`

### イミュータビリティと Semantics

イミュータブルクラスのインスタンスをプロパティを持つだけでなく、イミュータブルクラス自体の Semantics はどのように考えれば良いでしょうか。

次のようなイミュータブルクラス `Foo` を考えます。

```swift
final class Foo {
    let value: Int = 0
}
```

この `Foo` に対して、同様の処理を行います。

```swift
var a = Foo()
var b = a
a.value = 2 // ⛔
```

イミュータブルクラスのインスタンスは変更することができないので、変更の独立性の前提となる、変更そのものを実行することができません。 "Value Semantics in Swift" には、このようなケースも _Value Semantics_ を持っていると考えて良いと書かれています。

また、 "Value Semantics in Swift" にはさらに興味深いことが書かれていて、イミュータビリティを持つ場合は _Value Semantics_ と _Reference Semantics_ が区別できないとあります。そのため、上記のようなイミュータブルな `Foo` クラスは _Value Semantics_ と _Reference Semantics_ を両方持つと言えます。また、イミュータブルでさえあれば、 _値型_ であっても _Reference Semantics_ を持っていると言えるでしょう。

```swift
struct Foo {
    let value: Int = 0
}
```

### ミュータブルな参照型をプロパティに持つけど Value Semantics を持つ例

先程の例では、ミュータブルな _参照型_ をプロパティに持つ場合は _Value Semantics_ を持ちませんでした。

```swift
struct Foo {
    var value: Int = 0
    var bar: Bar = Bar()
}

class Bar {
    var value: Int = 0
}
```

しかし、そのようにパターンで判断するのは危険です。たとえば、標準ライブラリの `Array` は内部にミュータブルな _参照型_ を保持していますが、 _Copy-on-Write_ という仕組みを使って _Value Semantics_ を実現しています。

_Value Semantics_ を持つかどうかはパターンで判断するのではなく、定義に基づいて判断することが大切です。

### まとめ

_値型_ だからといって _Value Semantics_ を持つとは限りませんし、 _参照型_ でも _Value Semantics_ を持つこともあります。 Type と Semantics を区別して理解することが重要です。

_Value Semantics_ を持つかどうかをパターンに当てはめて考えると、様々な例外を考慮しなければなりません。変更に対する独立性を持つかどうかという、 _Value Semantics_  の定義に基づいて判断しましょう。
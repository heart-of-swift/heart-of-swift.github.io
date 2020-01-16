---
layout: section
chapter_index: 1
section_index: 2
---

本章では、制約としてプロトコルを用いることで、 _値型_ 中心のコードにおいてもパフォーマンスを損ねずにコードを抽象化できることを見てきました。しかし、 Swift の誕生から時間が経過し、様々なユースケースが生じる中で、制約としてのプロトコルによるコード抽象化に欠けているものが明らかになってきました。

そのような問題と解決策について Core Team の Joe Groff さんがまとめたドキュメントが ["Improving the UI of generics"](https://forums.swift.org/t/improving-the-ui-of-generics/22814) です。このドキュメントの中で **_リバースジェネリクス_** という新しい概念が説明され、その簡易形である **_Opaque Result Type_** が Swift 5.1 で部分的に導入されました。

本節では、制約としてのプロトコルと _リバースジェネリクス_ および _Opaque Result Type_ の関係を説明し、プロトコルを使ったコード抽象化の全体像を示します。

### 制約としてのプロトコルに欠けていた抽象化

[前節]({{ prev_section.path }})まで見てきたように、 Swift にとっては制約としてのプロトコルが適しています。実際、 Swift の標準ライブラリでも制約としてのプロトコルが広く使われており、ほとんどすべてのプロトコルが制約として使用されています。

たとえば、 `Sequence` プロトコルでは `IteratorProtocol` が `associatedtype` の制約として使用されています。

```swift
protocol Sequence {
    associatedtype Iterator: IteratorProtocol // 制約として使われている
    func makeIterator() -> Iterator
}
```

興味深いことに、 [Kotlin](https://kotlinlang.org) では同様の目的でインタフェース（ Swift のプロトコルのような役割を果たすもの）が `iterator` メソッド（ Swift の `makeIterator` メソッドに相当）の戻り値の型として使用されています。

```kotlin
// Kotlin
interface Iterable<out T> {
    operator fun iterator(): Iterator<T> // 型として使われている
}
```

Swift では `IteratorProtocol` が制約として使われ、 Kotlin では `Iterator` インタフェースが型として使われているわけです。これは、両言語の特徴をよく表しています。 Kotlin は参照型中心の言語なので、 `Iterator` インタフェースを型として使ってもオーバーヘッドは大きくありません。しかし、値型中心の Swift ではそうはいきません。

そもそも、 `IteratorProtocol` は `Element` という `associatedtype` を持っているので、現状の Swift では型として使うことができません。しかし、 _Generalized Existential_ がサポートされれば `IteratorProtocol` も型として使うことが可能になります。

```swift
// IteratorProtocol を型として使う場合
protocol Sequence {
    associatedtype Element
    func makeIterator() ->
        IteratorProtocol<.Element == Element>  // 型として使われている
}
```

`Sequence` プロトコルが上記のように宣言されれば、 Kotlin の `Iterable` インタフェースとほぼ同じ内容になります。

しかし、たとえ _Generalized Existential_ がサポートされても、 `makeIterator` メソッドの戻り値を `IteratorProtocol` 型にするのは望ましくありません。もし `IteratorProtocol` を型として使うと、イテレータの `next` メソッドを使って要素を取り出す度に _Existential Container_ のオーバーヘッドが発生します。イテレーションのように繰り返し実行される基本的な処理において _Existential Container_ のオーバーヘッドを許容することはできません。そのため、 Swift では `IteratorProtocol` を制約として用いることで、 _Existential Container_ のオーバーヘッドを防止しています。

たとえば、 `Array` の場合、 `makeIterator` メソッドの戻り値の型は `IndexingIterator<[Element]>` という具体的な型になります。

```swift
extension Array: Sequence {
    func makeIterator() -> IndexingIterator<[Element]> { ... }
}
```

同様に `String` の場合には `makeIterator` メソッドの戻り値の型は `String.Iterator` になります。

```swift
extension String: Sequence {
    func makeIterator() -> String.Iterator { ... }
}
```

`IteratorProtocol` は `Iterator` という `associatedtype` の制約としてのみ現れ、 `Array` や `String` などの具体的な型を実装する際には `IndexingIterator<[Element]>` や `String.Iterator` という具体的な型に置き換えられるわけです。そうすると、これらのイテレータを利用する際にも、抽象的な `IteratorProtocol` 型としてではなく、それぞれの具体的なイテレータ型として利用することができ、 _Existential Container_ のオーバーヘッドが発生しません。 `next` メソッドの呼び出しもオーバーヘッドのない _静的ディスパッチ_ になります。

しかし、未解決の問題が一つあります。

前述のように、 `Array` の `makeIterator` メソッドは `IndexingIterator<[Element]>` を返し、 `String` の `makeIterator` メソッドは `String.Iterator` を返します。しかし、これが露出していることは必ずしも望ましくありません。

Swift の標準ライブラリには次のように、 `IteratorProtocol` に適合する大量のイテレータ型が存在します。

- `AnyIterator`
- `CollectionOfOne.Iterator`
- `Dictionary.Iterator`
- `Dictionary.Keys.Iterator`
- `Dictionary.Values.Iterator`
- `DropWhileSequence.Iterator`
- `EmptyCollection.Iterator`
- `EnumeratedSequence.Iterator`
- `FlattenSequence.Iterator`
- `IndexingIterator`
- `IteratorSequence`
- `JoinedSequence.Iterator`
- `LazyDropWhileSequence.Iterator`
- `LazyFilterSequence.Iterator`
- `LazyMapSequence.Iterator`
- `LazyPrefixWhileSequence.Iterator`
- `PartialRangeFrom.Iterator`
- `PrefixSequence.Iterator`
- `ReversedCollection.Iterator`
- `Set.Iterator`
- `StrideThroughIterator`
- `StrideToIterator`
- `String.Iterator`
- `String.UnicodeScalarView.Iterator`
- `String.UTF16View.Iterator`
- `UnfoldSequence`
- `UnsafeBufferPointer.Iterator`
- `UnsafeRawBufferPointer.Iterator`
- `Zip2Sequence.Iterator`

`IteratorProtocol` に適合する型を実装する際には、これらのイテレータ型を使い分けて、もしくは独自のイテレータ型を実装して返さなければなりません。それ自体は問題ではなく、 `IteratorProtocol` が「型として」・「制約として」どちらの方法で使われていたとしても必要なことです。

問題となるのはイテレータの利用時です。 `makeIterator` メソッドの利用者はこれらのイテレータ型の違いを意識する必要はありません。

```swift
var iterator: IndexingIterator<[Int]> // この型を意識する必要はない
    = array.makeIterator()
```

この変数 `iterator` の型が `IndexingIterator<[Int]>` だろうと、 `Int` を取り出す他の何らかのイテレータであろうと、その違いを利用者が意識することはありません。前述のほとんどのイテレータ型は `next` メソッドしか持たず、 `Sequence` から要素を取り出すという意味でどれも同じ機能を提供しています。

にも関わらず、 `Array` の `makeIterator` メソッドは戻り値の型 `IndexingIterator<[Int]>` を公開しています。もし将来的に `Array` に特化したより高速なイテレータが実装されたとしても、 `Array` の `makeIterator` メソッドの戻り値の型を変更するのは困難です。 `Array` の `makeIterator` メソッドは Swift 標準ライブラリの `public` な API であり、その型を変更するということは、標準ライブラリの API の型を変更するということだからです。

もし、 `Array` の `makeIterator` メソッドの戻り値の型が次のように抽象化されているとどうなるでしょうか。

```swift
extension Array: Sequence {
    func makeIterator() -> Elementを取り出す何らかのイテレータ { ... }
}
```

これであれば、実体として `IndexingIterator<[Element]>` が返されていたところを、 `Array` に特化したより高速なイテレータに差し替えられても API としての表面上の型に変更はありません。 **利用者にとって本来必要なのはこのレベルの抽象度です。** 具体的なイテレータの型を知る必要はありません。

しかし、かといって戻り値の型に _Generalized Existential_ を使う（ `IteratorProtocol` を型として使う）わけにはいきません。

```swift
extension Array: Sequence {
    func makeIterator() -> IteratorProtocol<.Element == Element> { ... }
}
```

今の Swift ではこれはできませんが（ _Generalized Existential_ がサポートされていないことに加えて、 _Self-conformance_ もサポートされていないので、 `IteratorProtocol<.Element == Element>` が `associatedtype Iterator: IteratorProtocol` を満たせない）、仮にできたとしても _Existential Container_ のオーバーヘッドの問題が残ります。 `makeIterator` メソッドの戻り値の型は抽象的に書きたいですが、抽象化のために _Existential Container_ のオーバーヘッドを受け入れることはできません。

つまり、今望んでいるのは、 **抽象的にコードを書きながら、具象型と同じパフォーマンスがほしい** ということです。

### 抽象的なコードと具象型のパフォーマンス（引数の場合）

似たような話が前にもありました。ジェネリック関数です。プロトコルを制約として使ってコードを抽象化することで、具象型と同等のパフォーマンスを得ることができました。

```swift
// 抽象型引数と具象型のパフォーマンス
func useAnimal<A: Animal>(_ animal: A) {
    print(animal.foo())
}
```

上記のコードでは引数 `animal` の型が型パラメータ `A` によって抽象化されていますが、コンパイル時に型パラメータが _特殊化_ されることによって、具象型で記述したのと同等のパフォーマンスが得られました。たとえば、上記の `useAnimal` 関数の型パラメータ `A` が `Cat` に _特殊化_ されると、下記の `useAnimal` 関数と同等のパフォーマンスを実現できます。

```swift
// 具象型引数
func useAnimal(_ animal: Cat) {
    print(animal.foo())
}
```

これと同じようなアプローチで、 `makeIterator` メソッドにおいても抽象的なコードと具象型のパフォーマンスを両立できないでしょうか。

### 抽象的なコードと具象型のパフォーマンス（戻り値の場合）

`makeIterator` メソッドと先の `useAnimal` 関数の違いは、抽象化するのが戻り値の型なのか引数の型なのかという点です。ただ、 `makeIterator` メソッドは `useAnimal` 関数と違って複雑です。関数ではなくメソッドですし、プロトコルによって宣言されています。また、 `IteratorProtocol` の `associatedtype` である `Element` も関係しています。まずはよりシンプルな例を考えてみましょう。

`useAnimal` 関数は、引数として `Animal` を受け取ります。それとの対比として、戻り値として `Animal` を返す `makeAnimal` 関数を考えてみましょう。まずは、最もシンプルに具象型 `Cat` を返す `makeAnimal` 関数を考えます。

```swift
// 具象型戻り値
func makeAnimal() -> Cat {
    Cat()
}
```

先の `useAnimal` 関数のように、これをジェネリック関数にして戻り値の型を抽象化できないでしょうか。しかし、次のようなジェネリック関数にしようとするとコンパイルエラーになってしまいます。

```swift
// 抽象型戻り値と具象型のパフォーマンス？🤔
func makeAnimal<A: Animal>() -> A {
    Cat() // ⛔コンパイルエラー
}
```

なぜなら、ジェネリクスの型パラメータは必ずその API の利用者が決定するからです。 `useAnimal` 関数であれば、関数の利用者が引数に `Cat` 型の値を渡すことによって、型パラメータ `A` が `Cat` に決定されます。

```swift
useAnimal(Cat()) // 利用者が A を Cat に決定
```

そして、抽象的な型 `A` を利用するのは `useAnimal` 関数の実装者です。

```swift
func useAnimal<A: Animal>(_ animal: A) {
    print(animal.foo()) // 実装者が A を使用
}
```

しかし、 `makeAnimal` 関数ではこの関係が逆転します。 `makeAnimal` 関数が `Cat` 型を値を返すことを決定するのは関数の実装者です。

```swift
func makeAnimal() -> A { // A はどのように宣言する？🤔
    Cat() // 実装者が A を Cat に決定
}
```

そして、 `makeAnimal` 関数の利用者は戻り値の型を抽象的な型 `A` として扱います。

```swift
let animal = makeAnimal() // 利用者が A を使用
```

上記の関係をまとめると次のようになります。

<table>
<tr><th><code>useAnimal</code></th><td><strong>利用者</strong>が具象型を決定する。<strong>実装者</strong>が抽象型を使用する。</td></tr>
<tr><th><code>makeAnimal</code></th><td><strong>実装者</strong>が具象型を決定する。<strong>利用者</strong>が抽象型を使用する。</td></tr>
</table>

`useAnimal` と `makeAnimal` で **利用者** と **実装者** が逆転しています。 `makeAnimal` の戻り値の型を抽象化することは通常のジェネリクスではできません。

これを可能にするものとして、 **_リバースジェネリクス_** という概念が Manolo van Ee さんによって[提唱されました](https://forums.swift.org/t/reverse-generics-and-opaque-result-types/21608)。また、 Core Team の Joe Groff さんが関連する議論を整理し、 ["Improving the UI of generics"](https://forums.swift.org/t/improving-the-ui-of-generics/22814) というドキュメントにまとめました。このドキュメントは、過去にジェネリクス回りのロードマップとして示された ["Generics Manifesto"](https://github.com/apple/swift/blob/master/docs/GenericsManifesto.md) を補完する位置づけのものです。 Swift 5.1 時点では _リバースジェネリクス_ は採択されていません。しかし、機能的に _リバースジェネリクス_ のサブセットと言える _Opaque Result Type_ （後述）は Swift 5.1 で部分的にサポートされました。そのような経緯から、 _リバースジェネリクス_ は将来的に何らかの形で採択される可能性が高いと筆者は考えています。

ここでは、 _リバースジェネリクス_ について議論されている中で最も有力な次のシンタックスを採用します。通常のジェネリクスと異なり、 _リバースジェネリクス_ では型パラメータを `->` の後ろに記述します。

```swift
func makeAnimal() -> <A: Animal> A {
    Cat()
}
```

この `makeAnimal` 関数は次のように使えます。

```swift
let animal = makeAnimal()
print(animal.foo()) // ✅
```

`animal` の型は抽象的な `A` として扱われます。しかし、 `A` は制約上 `Animal` に適合しないといけないので、 `animal` が `foo` メソッドを持っていることは保証されます。そのため、 `animal.foo()` が実行できます。

また、 `A` は抽象的な型ですが、通常のジェネリクスと同じようにコンパイラによって _特殊化_ できます。そのため、 **実行時には**  `A` は `Cat` であるかのように扱われ、抽象化によるオーバーヘッドが発生しません。

ただし、 **コンパイル時には** `A` は型の上で明確に `Cat` と区別されます。 `animal` に格納されるインスタンスの実体は `Cat` ですが、 `animal` を `Cat` 型変数に代入することはできません。たとえ `A` が `Cat` であることをコンパイラが知っていても（ _特殊化_ できるということは知っているということです）、型エラーとしてコンパイルエラーにします。

```swift
let cat: Cat = animal // ⛔ animal の実体は Cat だけどコンパイルエラー
```

これは、 `A` の実体を隠蔽する上で重要です。もし、上記のコードを許すと何が起こるでしょうか。 `makeAnimal` 関数が `Cat` ではなく `Dog` を返すように変更されたとしましょう。すると、上記のコードは `Dog` インスタンスを `Cat` 型変数に代入しようとしていることになり、コンパイルエラーになってしまいます。

`makeAnimal` 関数の利用者目線では、これはとんでもないことです。この `makeAnimal` 関数が何らかのライブラリの API だったとしましょう。ライブラリをアップデートしたときに前述の変更が加えられていると、 `makeAnimal` 関数の型は変更されていないのに、それまでコンパイルが通っていたコードが急にコンパイルが通らなくなってしまうということです。たとえ `A` が `Cat` であることがわかっていても、最初から `A` と `Cat` を区別して扱っておくことでそのような事態を防ぐことができます。

`makeIterator` メソッドについても、 `makeAnimal` 関数と同じことが言えます。 Swift 5.1 時点では、 `Array` の `makeIterator` メソッドは次のように `IndexingIterator` を返すことを露出してしまっています。

```swift
// 具象型戻り値
extension Array: Sequence {
    func makeIterator() -> IndexingIterator<[Element]> { ... }
}
```

リバースジェネリクスを使えば次のようにイテレータの型を隠蔽できます。しかも、 _特殊化_ されれば抽象化による実行時のオーバーヘッドはありません。

```swift
// 抽象型戻り値と具象型のパフォーマンス😄
extension Array: Sequence {
    func makeIterator() -> <I: IteratorProtocol> I
        where I.Element == Element { ... }
}
```

**ジェネリクスが実行時のオーバーヘッドなく引数の型を抽象化できるように、 _リバースジェネリクス_ を使えば実行時のオーバーヘッドなく戻り値の型を抽象化できるのです。**

また、このとき `makeIterator` メソッドが返すイテレータの実体は `IndexingIterator` のままですが、それを `IndexingIterator` として受けることはできなくなります。

```swift
// makeIterator メソッドにリバースジェネリクスが使われたとき
var iterator: IndexingIterator<[Int]>
    = [2, 3, 5].makeIterator() // ⛔ コンパイルエラー
```

もし上記のコードが許されていると、将来的に `Array` の `makeIterator` メソッドが、別のより効率的なイテレータを返すように変更されたときに、急に上記のコードがコンパイルエラーになってしまいます。初めから `I` と `IndexingIterator` を区別しておくことで下記のようなコードを書くことを強制し、そのような問題の発生を防止することができます。

```swift
var iterator = [2, 3, 5].makeIterator() // ✅
```

### Opaque Result Type

_リバースジェネリクス_ を使えば抽象的な戻り値と具象型のパフォーマンスを両立できますが、必ずしもシンタックスがわかりやすいとは言えません。 `makeAnimal` 関数が何らかの `Animal` を返す場合、それを意味するコードは次の通りです。

```swift
// リバースジェネリクス
func makeAnimal() -> <A: Animal> A {
    Cat()
}
```

「何らかの `Animal` 」、つまり「ある `Animal` 」を返すわけです。英語で言えば「 some `Animal` 」です。次のように書ければよりわかりやすいはずです。

```swift
// Opaque Result Type
func makeAnimal() -> some Animal {
    Cat()
}
```

これが **_Opaque Result Type_** です。この "Result" は処理の「結果」、つまり、戻り値を意味しています。隠蔽された不透明（ Opaque ）な戻り値の型なので _Opaque Result Type_ です。

_Opaque Result Type_ は _リバースジェネリクス_ を簡潔に書くためのシンタックスシュガーだと考えることができます。シンタックスシュガーなので、上記の二つのコード（ _リバースジェネリクス_ 版と _Opaque Result Type_ 版の `makeAnimal` 関数）は全く同じことを意味します。

_Opaque Result Type_ は [SE-0244](https://github.com/apple/swift-evolution/blob/master/proposals/0244-opaque-result-types.md) で部分的に採択され、 Swift 5.1 でサポートされました。「部分的に」というのは、 Swift 5.1 時点では `some Sequence<.Element == Int>` のように `associatedtype` を指定したり、 `Set<some Animal>` や `[some Animal]` 、 `(some Animal)?` のように型パラメータを埋めるときに `some` を使うことができないからです。これらの機能については SE-0244 や "Improving the UI of generics" の中で言及されており、 "first step" として _Opaque Result Type_ の部分的な機能を導入すると述べられているため、将来的に導入される可能性が高いと考えられます。

### Opaque Argument Type

_Opaque Result Type_ を使えば _リバースジェネリクス_ を簡潔に記述できました。同じことは通常のジェネリクスについても言えるはずです。ジェネリックな引数を `some` を使って簡潔に書けるようにしようというのが **_Opaque Argument Type_** です。

例として、通常のジェネリクスで書かれた次のような関数 `useAnimal` を考えます。

```swift
// ジェネリクス
func useAnimal<A: Animal>(_ animal: A) {
    print(animal.foo())
}
```

これを _Opaque Argument Type_ で書いたコードが下記です。

```swift
// Opaque Argument Type
func useAnimal(_ animal: some Animal) {
    print(animal.foo())
}
```

_Opaque Result Type_ と _リバースジェネリクス_ の関係と同じように、 _Opaque Argument Type_ はジェネリクスのシンタックスシュガーです。上記の二つの `useAnimal` 関数はどちらの書き方をしてもまったく同じ意味になります。

なお、 _Opaque Argument Type_ は Swift 5.1 ではサポートされていません。 _Opaque Result Type_ の完全なサポート同様、今後導入される可能性が高そうです。

### ジェネリクスでしかできないこと

_Opaque Result Type_ と _Opaque Argument Type_ を合わせて **_Opaque Type_** と呼びます。 _Opaque Type_ がジェネリクス（ _リバースジェネリクス_ を含みます）のシンタックスシュガーなら、 _Opaque Type_ さえあればジェネリクスは不要なのでしょうか。そうではありません。ジェネリクスでしかできないことの例を見てみましょう。

たとえば、 `Animal` のつがいを引数に受け取る関数 `useAnimalPair` を考えてみます。

```swift
// ジェネリクス
func useAnimalPair<A: Animal>(_ pair: (A, A)) {
    ...
}
```

つがいなので、引数には同種の `Animal` を渡さなければなりません（ `Cat` と `Dog` ではいけません）。そのため、 `pair` の型は一つの型パラメータ `A` で書かれたタプル `(A, A)` となっています。

これを _Opaque Type_ で書こうとするとどうなるでしょうか。

```swift
// Opaque Argument Type
func useAnimalPair( _ pair: (some Animal, some Animal)) { // これで良い？🤔
    ...
}
```

しかし、上記のコードは下記のコードと同じ意味になってしまいます。

```swift
// ジェネリクス
func useAnimalPair<A1: Animal, A2: Animal>(_ pair: (A1, A2)) { // これではダメ😵
    ...
}
```

`some Animal` を 2 回書いた場合、それらは異なる型を意味することになります。もし、最初の `useAnimalPair` 関数のように、同種の `Animal` を二つ引数にとりたい場合にはジェネリクスを使うしかありません。

このように、 **_Opaque Type_ ではなくジェネリクスでしかできないこともあります。 _Opaque Type_ はジェネリクスでできることの一部を簡潔に書くための手段でしかありません。** このことからも、筆者はいずれ _リバースジェネリクス_ は採択されるだろうと考えています。 _Opaque Result Type_ が完全にサポートされても、 _リバースジェネリクス_ でしかできないことが存在するからです。

### Opaque Type と `some`

 _Opaque Type_ には `some` というキーワードを使いますが、筆者はこのキーワードの選び方が秀逸だと考えています。

下記のコードは、 先程 _Opaque Type_ で書こうとした `useAnimalPair` 関数のものです。

```swift
func useAnimalPair( _ pair: (some Animal, some Animal)) {
    ...
}
```

このとき、二つの `some Animal` は別の型を意味しますが、字面の上ではまったく同じです。しかし、「ある（ some ） `Animal` 」と「ある（ some ） `Animal` 」が異なる `Animal` を示すのは言語的には自然です。 `some` の代わりに当初考えられていた `opaque` が選ばれていると、二つの `opaque Animal` が異なる型を表すというのはよりわかりづらかったでしょう。

また、次のような例からも `some` というキーワード選定の秀逸さが見て取れます。

```swift
func useAnimals(_ animals: [some Animal]) {
    ...
}
```

この関数の引数 `animals` は、 _Homogeneous_ な（同種の値しか格納できない） `Array` です。「ある（ some ） `Animal` 」の `Array` がある一種類の `Animal` のインスタンスしか格納できないことは、言語的に自然です。

### SwiftUI と Opaque Type

Swift 5.1 と同時に [SwiftUI](https://developer.apple.com/documentation/swiftui) がリリースされました。 SwiftUI で初めて _Opaque Result Type_ に触れたという人も多いのではないかと思います。 _Opaque Result Type_ のユースケースの例として、 SwiftUI の中で _Opaque Result Type_ がどのように使われているのか、それがないと何が起こるかを見てみましょう。

SwiftUI を使うと宣言的に UI を記述できますが、その文脈では [Function Builder](https://github.com/apple/swift-evolution/blob/9992cf3c11c2d5e0ea20bee98657d93902d5b174/proposals/XXXX-function-builders.md) に焦点が当たりがちです。しかし、メソッドチェーンを主体とした API もその一端を担っています。

たとえば、次のように `padding` メソッドと `background` メソッドを連ねることによって、下記の虹色の正方形を作ることができます。

```swift
Spacer().frame(width: 20, height: 20)
    .padding(20).background(Color.violet)
    .padding(20).background(Color.indigo)
    .padding(20).background(Color.blue)
    .padding(20).background(Color.green)
    .padding(20).background(Color.yellow)
    .padding(20).background(Color.orange)
    .padding(20).background(Color.red)
```

<p>
<div style="background-color: #FF0000; padding: 20px; width: 260px; height: 260px;">
    <div style="background-color: #FF7F00; padding: 20px; width: 220px; height: 220px;">
        <div style="background-color: #FFFF00; padding: 20px; width: 180px; height: 180px;">
            <div style="background-color: #00FF00; padding: 20px; width: 140px; height: 140px;">
                <div style="background-color: #007FFF; padding: 20px; width: 100px; height: 100px;">
                    <div style="background-color: #3F00FF; padding: 20px; width: 60px; height: 60px;">
                        <div style="background-color: #7F00FF; padding: 20px; width: 20px; height: 20px;">
                        </div>
                    </div>
                </div>
            </div>
        </div>
    </div>
</div>
</p>

では、上記の式の型はどうなるでしょうか。一見 `Spacer` が `padding` をプロパティとして持っていれば、 `Spacer().padding(...)` の結果も同じ `Spacer` 型として扱えそうです。しかし、 `padding` を二つ連ねるとどうでしょうか。しかも、設定できるのは `padding` だけではありません。あらゆる設定値を複雑に組み合わせ入れ子状に設定できなければなりません。上のコードでも `frame`, `padding`, `background` の三つのメソッドを組み合わせて入れ子構造を作っています。

これを実現するには、たとえば `Padding` 型や `Background` 型を作って、それらの入れ子構造として表現することができます。そうすると、上記のメソッドチェーンは次のイニシャライザの入れ子と同じことにできます。

```swift
// ※ あくまで例であり、実際の SwiftUI とは異なります
Background(Padding(
    Background(Padding(
        Background(Padding(
            Background(Padding(
                Background(Padding(
                    Background(Padding(
                        Background(Padding(
                            Frame(
                                Spacer()
                            , width: 20, height: 20)
                        , 20), Color.violet)
                    , 20), Color.indigo)
                , 20), Color.blue)
            , 20), Color.green)
        , 20), Color.yellow)
    , 20), Color.orange)
, 20), Color.red)
```

この式の型であれば考えやすそうです。もし、 `Background` とそのイニシャライザが次にように実装されていれば、上記の式の型は `Background` です。

```swift
// ※ あくまで例であり、実際の SwiftUI とは異なります
struct Background: View {
    init(_ view: View, _ color: Color) { ... }
    ...
}
```

もし UIKit の `UIView` のように、 SwiftUI の `View` がビューの基底クラスになっているのであればこれで問題ありません。しかし、 SwiftUI の `View` はプロトコルです。そして、個々のビューは `struct` です。そもそも、[前節]({{ prev_section.path }})で説明した通り、 `View` プロトコルは `associatedtype Body` を持っているため、これを抽象的な `View` 型として扱うことはできません。仮に扱うことができたとしても、 _Existential Container_ のオーバーヘッドが発生してしまいます。 `AnyView` を使っても同様です。

では、 `Background` はどのように実装すれば良いでしょうか。そして、最初に挙げたメソッドチェーン（もしくは先のイニシャライザの入れ子呼び出し）の型はどうなるでしょうか。

抽象的な型を使わずに書くことを考えると、 `Background` の実装は次のようになります。

```swift
// ※ あくまで例であり、実際の SwiftUI とは異なります
struct Background<Content: View>: View {
    init(_ view: Content, _ color: Color) { ... }
    ...
}
```

`Background` に型パラメータを持たせて、対象となるビューの型を指定できるようにします。たとえば、 `Background(Spacer(), Color.red)` という式の型は `Background<Spacer>` 型になります。

`Padding` や `Frame` も同様です。

```swift
// ※ あくまで例であり、実際の SwiftUI とは異なります
struct Padding<Content: View>: View {
    init(_ view: Content, _ padding: CGFloat) { ... }
    ...
}

struct Frame<Content: View>: View {
    init(_ view: Content, witdh: CGFloat, height: CGFloat) { ... }
    ...
}
```

そうすると、最初のメソッドチェーン（もしくは先のイニシャライザの入れ子呼び出し）の型は次のようになります。

<!-- ```swift
Background<Padding<Background<Padding<Background<Padding<Background<Padding<Background<Padding<Background<Padding<Background<Padding<Frame<Spacer>>>>>>>>>
``` -->

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight" style="white-space: pre-wrap; word-break: break-all;"><code><span class="kt">Background</span><span class="o">&lt;</span><span class="kt">Padding</span><span class="o">&lt;</span><span class="kt">Background</span><span class="o">&lt;</span><span class="kt">Padding</span><span class="o">&lt;</span><span class="kt">Background</span><span class="o">&lt;</span><span class="kt">Padding</span><span class="o">&lt;</span><span class="kt">Background</span><span class="o">&lt;</span><span class="kt">Padding</span><span class="o">&lt;</span><span class="kt">Background</span><span class="o">&lt;</span><span class="kt">Padding</span><span class="o">&lt;</span><span class="kt">Background</span><span class="o">&lt;</span><span class="kt">Padding</span><span class="o">&lt;</span><span class="kt">Background</span><span class="o">&lt;</span><span class="kt">Padding</span><span class="o">&lt;</span><span class="kt">Frame</span><span class="o">&lt;</span><span class="kt">Spacer</span><span class="o">&gt;&gt;&gt;&gt;&gt;&gt;&gt;&gt;&gt;</span>
</code></pre></div></div>

とても長いですね。実際の SwiftUI ではもう少し複雑で、次のようになります。

<!-- ```swift
ModifiedContent<ModifiedContent<ModifiedContent<ModifiedContent<ModifiedContent<ModifiedContent<ModifiedContent<ModifiedContent<ModifiedContent<ModifiedContent<ModifiedContent<ModifiedContent<ModifiedContent<ModifiedContent<ModifiedContent<Spacer, _FrameLayout>, _PaddingLayout>, _BackgroundModifier<Color>>, _PaddingLayout>, _BackgroundModifier<Color>>, _PaddingLayout>, _BackgroundModifier<Color>>, _PaddingLayout>, _BackgroundModifier<Color>>, _PaddingLayout>, _BackgroundModifier<Color>>, _PaddingLayout>, _BackgroundModifier<Color>>, _PaddingLayout>, _BackgroundModifier<Color>>
``` -->

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight" style="white-space: pre-wrap; word-break: break-all;"><code><span class="kt">ModifiedContent</span><span class="o">&lt;</span><span class="kt">ModifiedContent</span><span class="o">&lt;</span><span class="kt">ModifiedContent</span><span class="o">&lt;</span><span class="kt">ModifiedContent</span><span class="o">&lt;</span><span class="kt">ModifiedContent</span><span class="o">&lt;</span><span class="kt">ModifiedContent</span><span class="o">&lt;</span><span class="kt">ModifiedContent</span><span class="o">&lt;</span><span class="kt">ModifiedContent</span><span class="o">&lt;</span><span class="kt">ModifiedContent</span><span class="o">&lt;</span><span class="kt">ModifiedContent</span><span class="o">&lt;</span><span class="kt">ModifiedContent</span><span class="o">&lt;</span><span class="kt">ModifiedContent</span><span class="o">&lt;</span><span class="kt">ModifiedContent</span><span class="o">&lt;</span><span class="kt">ModifiedContent</span><span class="o">&lt;</span><span class="kt">ModifiedContent</span><span class="o">&lt;</span><span class="kt">Spacer</span><span class="p">,</span> <span class="n">_FrameLayout</span><span class="o">&gt;</span><span class="p">,</span> <span class="n">_PaddingLayout</span><span class="o">&gt;</span><span class="p">,</span> <span class="n">_BackgroundModifier</span><span class="o">&lt;</span><span class="kt">Color</span><span class="o">&gt;&gt;</span><span class="p">,</span> <span class="n">_PaddingLayout</span><span class="o">&gt;</span><span class="p">,</span> <span class="n">_BackgroundModifier</span><span class="o">&lt;</span><span class="kt">Color</span><span class="o">&gt;&gt;</span><span class="p">,</span> <span class="n">_PaddingLayout</span><span class="o">&gt;</span><span class="p">,</span> <span class="n">_BackgroundModifier</span><span class="o">&lt;</span><span class="kt">Color</span><span class="o">&gt;&gt;</span><span class="p">,</span> <span class="n">_PaddingLayout</span><span class="o">&gt;</span><span class="p">,</span> <span class="n">_BackgroundModifier</span><span class="o">&lt;</span><span class="kt">Color</span><span class="o">&gt;&gt;</span><span class="p">,</span> <span class="n">_PaddingLayout</span><span class="o">&gt;</span><span class="p">,</span> <span class="n">_BackgroundModifier</span><span class="o">&lt;</span><span class="kt">Color</span><span class="o">&gt;&gt;</span><span class="p">,</span> <span class="n">_PaddingLayout</span><span class="o">&gt;</span><span class="p">,</span> <span class="n">_BackgroundModifier</span><span class="o">&lt;</span><span class="kt">Color</span><span class="o">&gt;&gt;</span><span class="p">,</span> <span class="n">_PaddingLayout</span><span class="o">&gt;</span><span class="p">,</span> <span class="n">_BackgroundModifier</span><span class="o">&lt;</span><span class="kt">Color</span><span class="o">&gt;&gt;</span>
</code></pre></div></div>

もしこのような虹色の正方形を表示するビュー `RainbowSquare` を実装しようとすると、 `body` プロパティの型としてそれが現れます。

<!-- ```swift
struct RainbowSquare: View {
    var body: ModifiedContent<ModifiedContent<ModifiedContent<ModifiedContent<ModifiedContent<ModifiedContent<ModifiedContent<ModifiedContent<ModifiedContent<ModifiedContent<ModifiedContent<ModifiedContent<ModifiedContent<ModifiedContent<ModifiedContent<Spacer, _FrameLayout>, _PaddingLayout>, _BackgroundModifier<Color>>, _PaddingLayout>, _BackgroundModifier<Color>>, _PaddingLayout>, _BackgroundModifier<Color>>, _PaddingLayout>, _BackgroundModifier<Color>>, _PaddingLayout>, _BackgroundModifier<Color>>, _PaddingLayout>, _BackgroundModifier<Color>>, _PaddingLayout>, _BackgroundModifier<Color>> {
        Spacer().frame(width: 20, height: 20)
            .padding(20).background(Color.violet)
            .padding(20).background(Color.indigo)
            .padding(20).background(Color.blue)
            .padding(20).background(Color.green)
            .padding(20).background(Color.yellow)
            .padding(20).background(Color.orange)
            .padding(20).background(Color.red)
    }
}
``` -->

<div class="language-swift highlighter-rouge"><div class="highlight"><pre class="highlight" style="white-space: pre-wrap; word-break: break-all;"><code><span class="kd">struct</span> <span class="kt">RainbowSquare</span><span class="p">:</span> <span class="kt">View</span> <span class="p">{</span>
    <span class="k">var</span> <span class="nv">body</span><span class="p">:</span> <span class="kt">ModifiedContent</span><span class="o">&lt;</span><span class="kt">ModifiedContent</span><span class="o">&lt;</span><span class="kt">ModifiedContent</span><span class="o">&lt;</span><span class="kt">ModifiedContent</span><span class="o">&lt;</span><span class="kt">ModifiedContent</span><span class="o">&lt;</span><span class="kt">ModifiedContent</span><span class="o">&lt;</span><span class="kt">ModifiedContent</span><span class="o">&lt;</span><span class="kt">ModifiedContent</span><span class="o">&lt;</span><span class="kt">ModifiedContent</span><span class="o">&lt;</span><span class="kt">ModifiedContent</span><span class="o">&lt;</span><span class="kt">ModifiedContent</span><span class="o">&lt;</span><span class="kt">ModifiedContent</span><span class="o">&lt;</span><span class="kt">ModifiedContent</span><span class="o">&lt;</span><span class="kt">ModifiedContent</span><span class="o">&lt;</span><span class="kt">ModifiedContent</span><span class="o">&lt;</span><span class="kt">Spacer</span><span class="p">,</span> <span class="n">_FrameLayout</span><span class="o">&gt;</span><span class="p">,</span> <span class="n">_PaddingLayout</span><span class="o">&gt;</span><span class="p">,</span> <span class="n">_BackgroundModifier</span><span class="o">&lt;</span><span class="kt">Color</span><span class="o">&gt;&gt;</span><span class="p">,</span> <span class="n">_PaddingLayout</span><span class="o">&gt;</span><span class="p">,</span> <span class="n">_BackgroundModifier</span><span class="o">&lt;</span><span class="kt">Color</span><span class="o">&gt;&gt;</span><span class="p">,</span> <span class="n">_PaddingLayout</span><span class="o">&gt;</span><span class="p">,</span> <span class="n">_BackgroundModifier</span><span class="o">&lt;</span><span class="kt">Color</span><span class="o">&gt;&gt;</span><span class="p">,</span> <span class="n">_PaddingLayout</span><span class="o">&gt;</span><span class="p">,</span> <span class="n">_BackgroundModifier</span><span class="o">&lt;</span><span class="kt">Color</span><span class="o">&gt;&gt;</span><span class="p">,</span> <span class="n">_PaddingLayout</span><span class="o">&gt;</span><span class="p">,</span> <span class="n">_BackgroundModifier</span><span class="o">&lt;</span><span class="kt">Color</span><span class="o">&gt;&gt;</span><span class="p">,</span> <span class="n">_PaddingLayout</span><span class="o">&gt;</span><span class="p">,</span> <span class="n">_BackgroundModifier</span><span class="o">&lt;</span><span class="kt">Color</span><span class="o">&gt;&gt;</span><span class="p">,</span> <span class="n">_PaddingLayout</span><span class="o">&gt;</span><span class="p">,</span> <span class="n">_BackgroundModifier</span><span class="o">&lt;</span><span class="kt">Color</span><span class="o">&gt;&gt;</span> <span class="p">{</span>
        <span class="kt">Spacer</span><span class="p">()</span><span class="o">.</span><span class="nf">frame</span><span class="p">(</span><span class="nv">width</span><span class="p">:</span> <span class="mi">20</span><span class="p">,</span> <span class="nv">height</span><span class="p">:</span> <span class="mi">20</span><span class="p">)</span>
            <span class="o">.</span><span class="nf">padding</span><span class="p">(</span><span class="mi">20</span><span class="p">)</span><span class="o">.</span><span class="nf">background</span><span class="p">(</span><span class="kt">Color</span><span class="o">.</span><span class="n">violet</span><span class="p">)</span>
            <span class="o">.</span><span class="nf">padding</span><span class="p">(</span><span class="mi">20</span><span class="p">)</span><span class="o">.</span><span class="nf">background</span><span class="p">(</span><span class="kt">Color</span><span class="o">.</span><span class="n">indigo</span><span class="p">)</span>
            <span class="o">.</span><span class="nf">padding</span><span class="p">(</span><span class="mi">20</span><span class="p">)</span><span class="o">.</span><span class="nf">background</span><span class="p">(</span><span class="kt">Color</span><span class="o">.</span><span class="n">blue</span><span class="p">)</span>
            <span class="o">.</span><span class="nf">padding</span><span class="p">(</span><span class="mi">20</span><span class="p">)</span><span class="o">.</span><span class="nf">background</span><span class="p">(</span><span class="kt">Color</span><span class="o">.</span><span class="n">green</span><span class="p">)</span>
            <span class="o">.</span><span class="nf">padding</span><span class="p">(</span><span class="mi">20</span><span class="p">)</span><span class="o">.</span><span class="nf">background</span><span class="p">(</span><span class="kt">Color</span><span class="o">.</span><span class="n">yellow</span><span class="p">)</span>
            <span class="o">.</span><span class="nf">padding</span><span class="p">(</span><span class="mi">20</span><span class="p">)</span><span class="o">.</span><span class="nf">background</span><span class="p">(</span><span class="kt">Color</span><span class="o">.</span><span class="n">orange</span><span class="p">)</span>
            <span class="o">.</span><span class="nf">padding</span><span class="p">(</span><span class="mi">20</span><span class="p">)</span><span class="o">.</span><span class="nf">background</span><span class="p">(</span><span class="kt">Color</span><span class="o">.</span><span class="n">red</span><span class="p">)</span>
    <span class="p">}</span>
<span class="p">}</span>
</code></pre></div></div>

とても書けたものではありません。しかも、虹を 7 色から 8 色に変更しようと思った場合（虹を何色で表すかは地域によって異なります）、 `body` プロパティの中身だけでなく型まで変更しなければなりません。少しコードを変更しただけで `body` の型が変わってしまうのでは、中身の実装に集中することができません。

_Opaque Result Type_ を使えば、この長ったらしいビューの型を `some View` として簡潔に記述できます。 `body` の中身を変更しても、 `some View` の部分を変更する必要はありません。

```swift
struct RainbowSquare: View {
    var body: some View {
        Spacer().frame(width: 20, height: 20)
            .padding(20).background(Color.violet)
            .padding(20).background(Color.indigo)
            .padding(20).background(Color.blue)
            .padding(20).background(Color.green)
            .padding(20).background(Color.yellow)
            .padding(20).background(Color.orange)
            .padding(20).background(Color.red)
    }
}
```

このように、 _Opaque Result Type_ を使って `body` プロパティの実際の型を隠蔽したことで何か不都合が生じるでしょうか。 API の利用者にとって、この `body` プロパティの実際の型が何であるかは重要ではありません。 `body` の型が `View` プロトコルに適合していることがわかれば十分です。つまり、「ある（ some ） `View` 」で良いわけです。一般的に、 `body` プロパティの型を `some View` に隠蔽して困ることはありません。 `some View` 型として表現することは、抽象度として適切な選択だと言えるでしょう。

そのような理由で、 _Opaque Result Type_ は SwiftUI で欠かせない役割を果たしています。

### Opaque Type と Existential Type

ここで _Opaque Type_ と _Existential Type_ の関係を整理しておきたいと思います。どちらも抽象化のために使用されます。

例として、 `useAnimal` 関数の引数の型の抽象化を考えます。 _Opaque Type_ の場合、次のようになります。

```swift
// Opaque Type
func useAnimal(_ animal: some Animal) {
    print(animal.foo())
}
```

一方で、 _Existential Type_ の場合は下記のようになります。このとき、 _Existential Type_ の方が引数 `animal` の型を簡潔に記述できます。

```swift
// Existential Type
func useAnimal(_ animal: Animal) { // Opaque Type より簡潔
    print(animal.foo())
}
```

しかし、上記の二つのコードの内、望ましいのは _Opaque Type_ による記述です。 _Opaque Type_ はジェネリクスのシンタックスシュガーであり、実行時のオーバーヘッドがありません。にも関わらず、今のシンタックスでは _Existential Type_ の方が簡潔に書けてしまっています。

ジェネリクスと _Existential Type_ ではシンタックスの見た目が大きく異なったためあまり気になりませんでしたが、 _Opaque Type_ を導入するとシンタックスの見た目が近くなったこともあり、この逆転が気になります。

そこで、 _Opaque Type_ に `some` というキーワードが必要なように、 _Existential Type_ にも `any` というキーワードを付与することが "Improving the UI of generics" の中で[提案されています](https://forums.swift.org/t/improving-the-ui-of-generics/22814#heading--clarifying-existentials)。 `any` が導入されると、たとえば、上記の _Existential Type_ を使った `useAnimal` 関数は次のようになります。

```swift
// Existential Type
func useAnimal(_ animal: any Animal) { // any が必要
    print(animal.foo())
}
```

このシンタックスが採用されれば、 _Opaque Type_ と _Existential Type_ の記述が、 `some Animal` vs `Animal` だったのが `some Animal` vs `any Animal` となり、シンタックスの上で対等になります。

さらに、 `any` の導入にはもう一つ大きな意味があります。これまで見てきた通り、 **プロトコルには、「型として」・「制約として」の二通りの異なる使い方があります。 `any` が導入されれば、型としてのプロトコル（ _Existential Type_ ）には `any` が付与されることとなり、プロトコルがどちらの使い方をされているのかコード上で一目瞭然となります。**

```swift
func useAnimal(_ animal: any Animal) { // any があるので「型として」
    print(animal.foo())
}

func useAnimal<A: Animal>(_ animal: A) { // any がないので「制約として」
    print(animal.foo())
}

```

また、筆者はこの `some` と `any` というキーワードは素晴らしい選択だと考えています。 `some` というキーワードの秀逸さについては先に述べましたが、 `any` と対比することでその素晴らしさがより際立ちます。

先程と同じように、 `some Animal` の `Array` を受け取る `useAnimals` 関数を考えてみます。

```swift
// Opaque Type
func useAnimals(_ animals: [some Animal]) {
    ...
}
```

このとき、引数 `animals` は「ある（ some ） `Animal` 」の `Array` です。「ある `Animal` 」の `Array` が特定の一種類の `Animal` しか格納できない（ _Homogeneous_ である）ことは言語的に自然です。

```swift
// Opaque Type
useAnimals([Cat(), Dog()]) // ⛔
useAnimals([Cat(), Cat()]) // ✅
useAnimals([Dog(), Dog()]) // ✅
```

同じように、 `any Animal` の `Array` を受け取る `useAnimals` 関数を考えます。

```swift
// Existential Type
func useAnimals(_ animals: [any Animal]) {
    ...
}
```

このとき、引数 `animals` は「任意の（ any ） `Animal` 」の `Array` です。「任意の `Animal` 」の `Array` が `Cat` や `Dog` を混在させることのできる（ _Heterogeneous_ である）ことは言語的に自然です。

```swift
// Existential Type
useAnimals([Cat(), Dog()]) // ✅
```

つまり、 _Opaque Type_ と _Existential Type_ という型システム上の概念が、 `some` と `any` というキーワードによって言語的にも対応しているわけです。シンタックスがセマンティクスを自然に表しているという意味で、すばらしいキーワード選定だと言えるのではないでしょうか。

### まとめ

本節では _リバースジェネリクス_ 、 _Opaque Type_ 、 _Existential Type_ など多くの項目を扱いましたが、それらの関係を表にまとめると次のようになります。

| | 引数 | 戻り値 |
|:--|:--:|:--:|
| _ジェネリクス_ <br/><small><em>Type-level abstraction</em></small><br/><small>制約としてのプロトコル</small> | `<A: Animal>(A) -> Void`  | <small><em>リバースジェネリクス</em></small><br/> `() -> <A: Animal> A` <br/><small>未サポート</small> |
| _Opaque Type_ <br/><small><em>Type-level abstraction</em></small><br/><small>ジェネリクスのシュガー</small> | <small><em>Opaque Argument Type</em></small><br/> `(some Animal) -> Void` <br/><small>未サポート</small> | <small><em>Opaque Result Type</em></small><br/> `() -> some Animal` <br/><small>一部サポート</small> |
| _Existential Type_ <br/><small><em>Value-level abstraction</em></small><br/><small>型としてのプロトコル</small>  | `(any Animal) -> Void` <br/><small>一部サポート</small> | `() -> any Animal` <br/><small>一部サポート</small> |

上二段の _ジェネリクス_ と _Opaque Type_ は、コンパイル時に静的に抽象化を扱います。コード上で抽象化された型がコンパイル時に _特殊化_ によって展開され、実行時のパフォーマンスに影響のない抽象化が可能となります。 Core Team の Joe Groff さんは "Improving the UI of generics" の中で、これを **_Type-level abstraction_** と呼んでいます。

一方、最下段の _Existential Type_ は、実行時に動的に抽象化を扱います。このときの主役は値です。値はプロトコルに適合していることだけが求められ、その型は重視されません。値の挙動は _Existential Container_ を用いて動的に解決されます。 Joe Groff さんはこれを **_Value-level abstraction_** と呼んでいます。

Swift は _値型_ 中心の言語であるにも関わらず、 _値型_ 前提で抽象化を考えたときに、実行時のオーバーヘッドなく戻り値の型を抽象化する方法がありませんでした（戻り値 × _Type-level abstraction_）。 _リバースジェネリクス_ とそのシンタックスシュガーである _Opaque Result Type_ によって、パフォーマンスを損ねることなく戻り値の型を抽象化できるようになりました。

ただし、いつでも _Type-level abstraction_ で良いというわけではなく、状況に応じて _Value-level abstraction_ も必要となります。そういう意味で、 _Existential Type_ も Swift には欠かせない存在です。その _Existential Type_ に `any` というキーワードを付与することが提案されており、それによって _Existential Type_ と _Opaque Type_ をシンタックス上、対等にすることができます、また、 `any` が付与されているかどうかで、プロトコルが型として用いられているのか、制約として用いられているのかがシンタックス上わかりやすくなります。

"Improving the UI of generics" の中でこれらの関係性が示されて、 Swift に必要な道具が明確になりました。しかし、上記の表の通り、 Swift 5.1 時点ではまだまだサポートされていない道具が多くあります。 _Opaque Result Type_ は、現時点では `[some Animal]` や `some Sequence<.Element == Int>` などができません。 _Existential Type_ についても、 _Generalized Existential_ （ `any Sequence<.Element == Int>` や `any View` など）が欠けたままです。 _Opaque Argument Type_ や _Reverse Generics_ については部分的にすらサポートされていません。それらの必要性は、本章を通して、 Swift が _値型_ 中心の言語であることから必然的に導かれました。すべてが揃うことによって、 Swift は _値型_ 中心の言語として、より完成に近づけることでしょう。

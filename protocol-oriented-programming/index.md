---
layout: section
title: "Protocol-oriented Programming とは"
chapter_index: 1
section_index: 0
---

{% assign chapter=site.data.book.chapters[page.chapter_index] %}
{% assign next_section_index=page.section_index | plus: 1 %}
{% assign next_section=chapter.sections[next_section_index] %}

_Value Semantics_ と違って、 **_Protocol-oriented Programming_** という用語は Swift プログラマの間で広く知られています。しかし、筆者の知る限り _Protocol-oriented Programming_ の定義は明確に述べられていません。 WWDC 2015 のセッション ["Protocol-Oriented Programming in Swift"](https://developer.apple.com/videos/play/wwdc2015/408/) の中では、

- _Self-requirement_
- _Protocol Extension_
- プロトコルの後付け

など数多くの例が取り上げられていますが、何が _Protocol-oriented Programming_ なのかは明確に述べられていません。

それでは、_Protocol-oriented Programming_ とは何なのでしょうか。筆者の解釈は次の通りです。 _Protocol-oriented Programming_ という用語は _Object-oriented Programming （オブジェクト指向プログラミング）_ との対比によって生み出されたものだと、筆者は考えています。 Swift におけるプロトコルは抽象化の道具です。 _オブジェクト指向プログラミング_ における抽象化の側面に目を向けてみると、 _オブジェクト指向プログラミング_ では

- クラスの _継承_
- _ポリモーフィズム_

を用いてコードを抽象化します。しかし、前章 ["{{ site.data.book.chapters[0].name }}"]({{ site.data.book.chapters[0].path }}) で見てきたように、 Swift は _値型_ 中心の言語です。 _参照型_ と異なり、 _値型_ は原理的に _継承_ することができません。そのため、 Swift ではクラスの _継承_ ではなくプロトコルが抽象化の主役になります。 _Protocol-oriented Programming_ とは、そのようなプロトコルを用いてコードを抽象化する手法全般を指しているというのが筆者の考えです。

本章では、プロトコルを用いてコードを抽象化する方法と、その結果として必要とされる _リバースジェネリクス_ や _Opaque Result Type_ 等について説明します。

## プロトコルによる不適切な抽象化

プロトコルを用いた抽象化とはどのようなものでしょうか。 _継承_ の代わりにプロトコルを使おうと考えると、次のような方法を思い浮かべるかもしれません。

`Animal` というプロトコルがあるとします。

```swift
protocol Animal {
    func foo() -> Int
}
```

これに適合した二つの `struct` 、 `Cat` と `Dog` を考えます。

```swift
struct Cat: Animal {
    func foo() -> Int { 2 }
}

struct Dog: Animal {
    func foo() -> Int { 1 }
}
```

`struct` は _値型_ ですが、 _継承_ はできなくてもプロトコルに _適合_ することはできます。

今、 `Cat` や `Dog` は `Animal` に適合しているので、 `Cat` や `Dog` のインスタンスを `Animal` 型変数に代入して使うことができます。

```swift
let animal: Animal = Bool.random() ? Cat() : Dog() // ✅
print(animal.foo())
```

_オブジェクト指向プログラミング_ でよく見られるポリモーフィズムです。

しかし、 Swift においてこのようなプロトコルの使い方は必ずしも適切ではありません。なぜ適切でないのか、どのようにプロトコルを使えば良いのかというのが本節のテーマです。

## Existential Type と Existential Container

先のコードが不適切な理由を考えるために、コードを少し改変します。 `Cat` には 1 バイトの、 `Dog` には 4 バイトの Stored Property を持たせてみましょう。

```swift
struct Cat: Animal {
    var value: UInt8 = 2 // 1 バイト
    func foo() -> Int { ... }
}

struct Dog: Animal {
    var value: Int32 = 1 // 4 バイト
    func foo() -> Int { ... }
}
```

`Cat` と `Dog` は _値型_ なので、変数・定数にはそれらのインスタンスが直接格納されます。そのため、 `Cat` 型変数は 1 バイトの、 `Dog` 型変数は 4 バイトの領域を必要とします。

```swift
let cat: Cat = Cat() // 1 バイト
let dog: Dog = Dog() // 4 バイト
```

では、 `Animal` 型変数を作って `Cat` か `Dog` のインスタンスを代入するとき、 `Animal` 型変数は何バイトの領域を必要とするでしょうか。

```swift
let animal: Animal = Bool.random() ? cat : dog // 何バイト？🤔
```

一般的な 64 ビット環境では、 `Animal` 型変数はなんと 40 バイトもの領域を必要とします。

```swift
print(MemoryLayout.size(ofValue: animal)) // 40
```

1 バイトか 4 バイトの値を格納するだけなら 4 バイトあれば十分なように思えます。なぜ 40 バイトもの領域が必要なのでしょうか。

`Animal` 型変数に格納できるのは `Cat` と `Dog` のインスタンスだけではありません。 `Aniaml` 型として考えると、 `Animal` に適合した任意の型のインスタンスを格納できなければなりません。しかし、 `Animal` に適合した型は、理論上いくらでも大きくできます。たとえば、 1 バイトの Stored Property を 1,000 個持たせれば 1,000 バイトになります。単純に大きな領域を用意するだけでは任意の型のインスタンスを格納することはできません。

そのため、プロトコル型変数にインスタンスを格納する際には、 **_Existential Container_** という特殊な入れ物が用いられます。 _Existential Container_ は任意のサイズのインスタンスを格納できる入れ物です。インスタンスは _Existential Container_ に入れられた上で変数に格納されます。上記の例では _Existential Container_ のサイズが 40 バイトなので、 `animal` のサイズも 40 バイトとなっています。

なお、 _Existential Container_ という名称は、プロトコルを型として用いた場合、その型が **_Existential Type_** と[呼ばれる](https://docs.swift.org/swift-book/LanguageGuide/Protocols.html#ID275)ことに由来します。

_Existential Container_ についての詳しい説明は本書の範囲を超えてしまうため、本書ではこれ以上説明しません。 _Existential Container_ については [Swift のリポジトリ](https://github.com/apple/swift)にあるドキュメント ["Type Layout"](https://github.com/apple/swift/blob/master/docs/ABI/TypeLayout.rst#existential-container-layout) の ["Existential Container Layout"](https://github.com/apple/swift/blob/master/docs/ABI/TypeLayout.rst#existential-container-layout) を御覧下さい。

本書を読み進める上で _Existential Container_ について知っておくべきことは、プロトコル型変数に値を代入すると _Existential Container_ に包まれるということです。そしてそれは、

- メモリを余分に消費する
- _Existential Container_ に包んだり取り出したりする実行時のオーバーヘッドが発生する

ということを意味します。

## 参照型とポリモーフィズム

_Existential Container_ のオーバーヘッドは _値型_ とプロトコルに起因するものです。 _参照型_ であるクラスの _継承_ ではそのようなオーバーヘッドは発生しません。

たとえば、先の `Animal`, `Cat`, `Dog` をすべてクラスにしてみましょう。

```swift
class Animal {}
class Cat: Animal {}
class Dog: Animal {}
```

["Value Semantics とは"](/value-semantics/) でも見たように、 _参照型_ のインスタンスは変数に直接格納されるわけではありません。インスタンスはメモリ上の別の場所に存在し、その場所を指し示すアドレスが変数に格納されます。一般的には、 64 ビット環境においてアドレスは 8 バイト（ 64 ビット）で表されます。そのため、 _参照型_ の変数はどのような型でも 8 バイトの領域しか必要としません。

```swift
let cat: Cat = Cat() // 8 バイト
let dog: Dog = Dog() // 8 バイト
let animal: Animal   // 8 バイト
    = Bool.random() ? cat : dog
```

上記のコードでは、 `cat` や `dog` 、 `animal` に格納されているのはすべてインスタンスのアドレスです。 `cat` や `dog` を `animal` に代入する際に _Existential Container_ に包む必要はありません。

_参照型_ の場合、 _Existential Container_ のようなオーバーヘッドが存在しないので、 _継承_ と _ポリモーフィズム_ による抽象化は理に適った方法だと言えます。

## 値型に適したポリモーフィズム

それでは、 _値型_ ではどのようにコードを抽象化すれば良いでしょうか。

抽象化はプログラミングにおける一大テーマです。 _Value Semantics_ のために _値型_ 中心にした結果、抽象化が妨げられるというのは許容できません。

実は、一口に _ポリモーフィズム_ と言っても様々な種類の _ポリモーフィズム_ があります。ここでは 2 種類の _ポリモーフィズム_ を比較し、 _値型_ に適した _ポリモーフィズム_ を考えます。

### サブタイプポリモーフィズム

 _オブジェクト指向_ の文脈で _ポリモーフィズム_ と言った場合、多くは **_サブタイプポリモーフィズム_** を指しています（ _サブタイピング_ と呼ばれる方が一般的です）。これまでに本章で見てきた _ポリモーフィズム_ はすべて _サブタイプポリモーフィズム_ です。

プロトコルによって _サブタイプポリモーフィズム_ を実現する場合、プロトコルは **型として** 用いられます。次のコードでは、 `Animal` プロトコルが `Animal` 型として用いられています。

```swift
// サブタイプポリモーフィズム
func useAnimal(_ animal: Animal) {
    print(animal.foo())
}
```

### パラメトリックポリモーフィズム

同様の挙動を示す `useAnimal` 関数は、 **_パラメトリックポリモーフィズム_** を用いても実現できます（ _パラメータ多相_ と呼ばれることも多いです）。

_パラメトリックポリモーフィズム_ を使う場合、プロトコルは **制約として** 用いられます。次のコードでは、 `Animal` プロトコルが _型パラメータ_ `A` の制約として用いられています。

```swift
// パラメトリックポリモーフィズム
func useAnimal<A: Animal>(_ animal: A) {
    print(animal.foo())
}
```

### 実行時に起こること（サブタイプポリモーフィズム）

これらの二つの `useAnimal` 関数はどちらも同じように使うことができ、実行結果も同じです。

```swift
// どちらでも同じように使い、結果も同じ
useAnimal(Cat())
useAnimal(Dog())
```

しかし、実行時に内部的に行われていることはまったく異なります。

_サブタイプポリモーフィズム_ の場合、 `useAnimal` に引数を渡すときにインスタンスを _Existential Container_ に包む必要があります。

```swift
// サブタイプポリモーフィズム
useAnimal(Cat()) // Existential Container に包む
useAnimal(Dog()) // Existential Container に包む
```

また、 `useAnimal` の中で `animal.foo()` を呼び出す箇所では、 `animal` が `Cat` か `Dog` か、または別の型のインスタンスかは実行時までわかりません。 `Cat` の `foo` を呼び出すか `Dog` の `foo` を呼び出すかは実行時に決定されます（ **_動的ディスパッチ_** ）。

```swift
// サブタイプポリモーフィズム
func useAnimal(_ animal: Animal) {
    print(animal.foo()) // 動的ディスパッチ
}
```

_動的ディスパッチ_ には当然実行時のオーバーヘッドがありますが、 _Existential Type_ を経由してメソッドを呼び出すオーバーヘッドはそれだけではありません。クラスの継承の場合は _vtable_ を介してメソッドを見つけるだけなので大したオーバーヘッドではないですが、 _Existential Type_ を経由してメソッドを呼び出す場合は、一度 _Existential Container_ からインスタンスを取り出すオーバーヘッドも発生します。

### 実行時に起こること（パラメトリックポリモーフィズム）

_パラメトリックポリモーフィズム_ の場合はどうなるでしょうか。

_パラメトリックポリモーフィズム_ の場合、 `useAnimal` は _型パラメータ_ `A` を持つジェネリック関数です。ジェネリック関数は、様々な型に対して個別に関数を実装する代わりに、まとめて一つの関数として実装する手段と言えます。

たとえば、オーバーロードを用いて `Cat` 用の `useAnimal` 関数と `Dog` 用の `useAnimal` 関数を個別に実装することもできます。

```swift
// Cat 用の useAnimal
func useAnimal(_ animal: Cat) {
    print(animal.foo())
}

// Dog 用の useAnimal
func useAnimal(_ animal: Dog) {
    print(animal.foo())
}
```

しかし、これでは類似のコードが重複してしまいます。また、 `Animal` プロトコルに適合する型は `Cat` と `Dog` だけではありません。それらのすべての型に対応しようとするとキリがありません。

そこで、それぞれの型に対して関数をオーバーロードする代わりに、 _型パラメータ_ という概念を導入して一つの関数として実装できるようにしたのがジェネリック関数です。

```swift
// ジェネリック関数として実装された useAnimal
func useAnimal<A: Animal>(_ animal: A) { // A が Cat や Dog などを表す
    print(animal.foo())
}
```

概念的には、ジェネリック関数は _型パラメータ_ に `Cat` や `Dog` などの具体的な型を当てはめ、それらの関数がオーバーロードされているのと（ほぼ）同じです。上記のような `useAnimal` 関数を一つ実装すれば、 `Cat` 用の `useAnimal` や `Dog` 用の `useAnimal` を個別実装してオーバーロードするのと同じように振る舞います。

さらに、概念上だけでなく実行時の挙動もそれと似たものになります。コンパイラが **_特殊化_** （最適化の一種）を行った場合、ジェネリック関数としての `useAnimal` のバイナリに加えて、型パラメータに `Cat` や `Dog` などの具体的な型を当てはめた `useAnimal` のバイナリも生成されます。そして、 `useAnimal` に `Cat` インスタンスを渡す箇所では `Cat` 用の `useAnimal` が、 `Dog` インスタンスを渡す箇所では `Dog` 用の `useAnimal` が呼び出されるようにコンパイルされます。

そのため、 _サブタイプポリモーフィズム_ の場合と異なり、 `useAnimal` に `Cat` インスタンスや `Dog` インスタンスを渡す際に _Existential Container_ に包む必要はありません。 `Cat` インスタンスは `Cat` 用 `useAnimal` に、 `Dog` インスタンスは `Dog` 用 `useAnimal` にそのまま直接渡されます。

```swift
// パラメトリックポリモーフィズム
useAnimal(Cat()) // Cat のまま直接渡される
useAnimal(Dog()) // Dog のまま直接渡される
```

`useAnimal` 関数の中で `animal.foo()` を呼び出す箇所についても、 `Cat` 用の `useAnimal` の中では `animal` が `Cat` であることがわかっているので、コンパイル時に `Cat` の `foo` を呼び出せば良いと決定できます（ **_静的ディスパッチ_** ）。そのため、実行時にメソッドを選択するオーバーヘッドが発生しません。

```swift
// パラメトリックポリモーフィズム
func useAnimal<A: Animal>(_ animal: A) {
    print(animal.foo()) // 静的ディスパッチ
}
```

さらに、 _静的ディスパッチ_ の場合は、コンパイラが最適化によってメソッド呼び出しを **_インライン展開_** （ _インライン化_ ）することがあります。 _インライン展開_ とは、関数やメソッドの呼び出し箇所に、その関数やメソッドの中身を展開して埋め込むことです。そうすることで、実行時に関数やメソッドを呼び出すオーバーヘッドがゼロになり、パフォーマンスを向上させることができます（ただし、コンパイル後のバイナリのサイズが増大するなどの不利益も存在します。サイズが増大するのは、一つの関数が復複数箇所に展開されることになるためです）。

このように、 _パラメトリックポリモーフィズム_ を用いると、実行時のオーバーヘッドを発生させずにコードを抽象化することができます。上記のコードでは、 `Cat` 用の `useAnimal` と `Dog` 用の `useAnimal` が一つのジェネリック関数として抽象化されていますが、実行時のオーバーヘッドはありません。 **_値型_ 中心の Swift においては、** _値型_ と組み合わせたときにオーバーヘッドの大きい _サブタイピングポリモーフィズム_ よりも、 _値型_ であってもオーバーヘッドの発生しない **_パラメトリックポリモーフィズム_ の方が適しています。** 実際、 Swift の標準ライブラリで用いられている _ポリモーフィズム_ のほとんどが _パラメトリックポリモーフィズム_ です。

ただし、いつでも _パラメトリックポリモーフィズム_ を用いれば良いというわけではありません。 `useAnimal` 関数は _サブタイプポリモーフィズム_ と _パラメトリックポリモーフィズム_ のどちらを用いても実装することができました。そのような場合には、 _パラメトリックポリモーフィズム_ を用いると良いでしょう。しかし、二つの _ポリモーフィズム_ のどちらか片方でしかできないこともあります。 _サブタイプポリモーフィズム_ でしかできないことには _サブタイプポリモーフィズム_ を用いる必要があります（二つの _ポリモーフィズム_ の使い分けについては[次節]({{ next_section.path }})で詳しく説明します）。

## 制約としてのプロトコル

_サビタイプポリモーフィズム_ よりも _パラメトリックポリモーフィズム_ を優先するということを、プロトコルを中心にして言い替えると次のようになります。

- **プロトコルを「型として」ではなく「制約として」使用することを優先する**

["Protocol-Oriented Programming in Swift"](https://developer.apple.com/videos/play/wwdc2015/408/) の中では特別そのことは強調されていません。しかし、本節で述べた内容や標準ライブラリにおけるプロトコルの利用例を見る限り、これこそが Swift のプロトコルの使い方ついて最も重要なことだと筆者は考えています。

## まとめ

_Protocol-oriented Programming_ という用語は広く知られていますが、筆者の知る限り明確に定義されていません。筆者の解釈では、 _Object-oriented Programming_ （ _オブジェクト指向プログラミング_ ）との対比となる用語であり、プロトコルを用いてコードを抽象化する手法全般を指していると考えています。

_オブジェクト指向プログラミング_ においてはコードの抽象化のために _サブタイプポリモーフィズム_ が多用されます。しかし、プロトコルと _値型_ を使って _サブタイプポリモーフィズム_ を実現しようとすると、 _Existential Container_ に包んだり取り出したりするオーバーヘッドが生じます。 Swift は _値型_ を中心とした言語であり、 _オブジェクト指向言語_ と同じような方法で _サブタイプポリモーフィズム_ を用いるとパフォーマンスに悪影響を与えます。

_パラメトリックポリモーフィズム_ を用いれば、プロトコルと _値型_ に対しても、実行時のオーバーヘッドが生じない形でコードを抽象化できます。そのため、 Swift においては _サブタイプポリモーフィズム_ よりも _パラメトリックポリモーフィズム_ を優先的に使用するのが望ましいです。ただし、 _サビタイプポリモーフィズム_ と _パラメトリックポリモーフィズム_ はまったく同じことができるわけではありません。どちらかでしか実現できないことに対しては、適切に二つの _ポリモーフィズム_ を使い分ける必要があります。

プロトコルに着目すると、上記の話は、プロトコルを「型として」ではなく「制約として」使用することを優先すると言い替えることができます。それが、 _Protocol-oriented Programming_ において最も重要なことだと筆者は考えています。

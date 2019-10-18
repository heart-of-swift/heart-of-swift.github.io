---
layout: section
chapter_index: 1
section_index: 1
---

{% assign chapter=site.data.book.chapters[page.chapter_index] %}
{% assign prev_section_index=page.section_index | minus: 1 %}
{% assign prev_section=chapter.sections[prev_section_index] %}

[前節]({{ prev_section.path }})で見たように、型としてのプロトコルと制約としてのプロトコルには、どちらでもできること・どちらかでしかできないことがあります。

この関係を図で表すと次のようになります。

<img src="img/protocol-roles.png" alt="「型として」・「制約として」のプロトコルにできること" style="width: 512px; max-width: 100%;" />

青い円が型としてのプロトコルでできることを、赤い円が制約としてのプロトコルでできることを表しています。二つの円の重なる部分が、「型として」・「制約として」のどちらのプロトコルでもできることです。[前節]({{ prev_section.path }})の `useAnimal` 関数はどちらでも実装できるので、この紫の領域に属します。

```swift
// 型としてのプロトコル
func useAnimal(_ animal: Animal) {
    print(animal.foo())
}

// 制約としてのプロトコル
func useAnimal<A: Animal>(_ animal: A) {
    print(animal.foo())
}
```

型としてのプロトコルは、 _値型_ にとって実行時のオーバーヘッドが大きいプロトコルの使い方でした。そのため、 _値型_ 中心の Swift では制約としてのプロトコルを優先するのが望ましいというのが[前節]({{ prev_section.path }})の結論でした。つまり、紫の領域については制約としてのプロトコルを用いれば良いということになります。これを先程と同じように図で表すと次のようになります。

<img src="img/protocol-roles-2.png" alt="制約としてプロトコルを優先する" style="width: 512px; max-width: 100%;" />

しかし、図から青い領域がなくなったわけではありません。どんな場合でもプロトコルを制約として用いれば良いというわけではないのです。青い領域に対して、制約としてのプロトコルで無理やり対処しようとすると、わかりづらく非効率なコードを書くことになるでしょう。状況に応じて、型としてのプロトコルを適切に利用する必要があります。

本節では、型としてのプロトコルでしかできないこと・制約としてのプロトコルでしかできないことを示し、それらの使い分けについて説明します。

### 型としてのプロトコルでしかできないこと

まず、型としてのプロトコルでしかできないことの例を見てみます。

`useAnimal` 関数を少し変更して、複数の `Animal` を受け取る関数 `useAnimals` を考えてみます。元々は単一の `Animal` を引数で受け取っていましたが、複数の `Animal` を受け取るために引数の型を `[Animal]` に変更しています。

```swift
func useAnimals(_ animals: [Animal]) {
    ...
}
```

上記の引数 `animals` は任意の `Animal` を要素として格納できる `Array` です。次のように、 `Cat` インスタンスと `Dog` インスタンスを混在させることも可能です。

```swift
useAnimals([Cat(), Dog()]) // ✅
```

このように、異なる型のインスタンスが混在したコレクションを _Heterogeneous Collection_ と呼びます。

制約としてのプロトコルでは、これと同じことはできません。 `useAnimal` 関数を制約としてのプロトコルで実装しようとすると次のようになります。

```swift
func useAnimals<A: Animal>(_ animals: [A]) {
    ...
}
```

しかし、この関数に `Cat` と `Dog` が混在した `Array` を渡そうとするとコンパイルエラーになります。

```swift
useAnimals([Cat(), Dog()]) // ⛔
```

`useAnimals` 関数の型パラメータ `A` は、 `Cat` や `Dog` などの具体的な一つの型を表します。 `A` が `Cat` のときは `animals` の型は `[Cat]` に、 `A` が `Dog` のときは `animals` の型は `[Dog]` になります。そのため、次のように `[Cat]` や `[Dog]` を渡すことはできます。

```swift
useAnimals([Cat(), Cat()]) // ✅
useAnimals([Dog(), Dog()]) // ✅
```

しかし、 `[A]` が `[Animal]` を表すことはできません。 `A` が表すことができるのは、 `Cat` や `Dog` などの **具体的な** 型だけです。このように、 **_Heterogeneous Collection_ が必要な場合にはプロトコルを型として使う** ことになります。

なお、 Swift の型システム的には、 `[A]` が `[Animal]` を表せないのは、プロトコル型がそのプロトコル自体に適合しないからです。 `Animal` 型は `Animal` プロトコル自体に適合しません。プロトコル型がそのプロトコル自体に適合することを **_Self-conformance_** と言いますが、 Swift では一般的なプロトコルは _Self-conformance_ を持ちません（例外的に [SE-0235](https://github.com/apple/swift-evolution/blob/master/proposals/0235-add-result.md) で `Error` プロトコルに _Self-conformance_ が[追加されました](https://github.com/apple/swift-evolution/blob/master/proposals/0235-add-result.md#adding-swifterror-self-conformance)）。

#### Heterogeneous Collection の具体例


そのようなケースの代表的な例は、 GUI のビューツリーです。典型的な GUI のビューは、各ビューが複数の子ビューを持つ階層的なツリー構造を持ちます。一つの階層には、次のように、チェックボックスやラベル、ボタンなど、異なる種類のビューを格納できます。

<div class="html-demo">
    <div><input type="checkbox" /> 利用規約に同意します。</div>
    <div><button>送信</button></div>
</div>

この階層を `Array` で表すと次のようになります。

```swift
let views = [
    Checkbox(),
    Label("利用規約に同意します。"),
    Button("送信"),
]
```

このとき、 `views` の型はどうなるでしょうか。 `Checkbox`, `Button`, `Label` がそれぞれ `View` プロトコルに適合しているなら、 `views` の型は `[View]` として表せます。

では、


これらを子ビューとして

しかし、 `View` プロトコルを制約として用いて

#### Generalized Existential

#### 型消去

### 制約としてのプロトコルでしかできないこと

```swift
protocol Equatable {
    static func == (
        lhs: Self, // 👈
        rhs: Self  // 👈
    ) -> Bool
}
```

```swift
extension Int: Equatable {
    static func == (
        lhs: Int, // 👈
        rhs: Int  // 👈
    ) -> Bool { ... }
}
```

```swift
extension String: Equatable {
    static func == (
        lhs: String, // 👈
        rhs: String  // 👈
    ) -> Bool { ... }
}
```

もし `Equatable` プロトコルが型として使えたら

```swift
let a: Equatable = 42
let b: Equatable = "42"
a == b // 🤔
```

```swift
let a: Equatable = 42
let b: Equatable = "42"
a == b // ⛔
```

```swift
let a: Equatable = 42
let b: Equatable = 42
a == b // ⛔
```

`Equatable` プロトコルを型として使う意味はない

```swift
extension Sequence where Element: Equatable { // 👈
    func contains(_ element: Element) -> Bool {
        ...
    }
}
```

`Self` 引数を持つプロトコルは制約として使う前提

### まとめ

- プロトコルは型としても制約としても使える
- 値型には「制約」の方が適している
- 「型」と「制約」でできることは異なる
- 「制約」優先で、適切に「型」と使い分ける

---
layout: page
---

{% assign chapter=site.data.book.chapters[0] %}
{% assign section_index=2 %}
{% assign section=chapter.sections[section_index] %}
{% assign prev_section_index=section_index | minus: 1 %}
{% assign prev_section=chapter.sections[prev_section_index] %}

## {{ section.name }}

[前ページ]({{ prev_section.path }})で見たように、 Swift の標準ライブラリで提供される型はほぼすべてが _値型_ で _Value Semantics_ を持っています。このような言語は稀です。 _値型_ が _ミュータブルクラス_ と _イミュータブルクラス_ のいいとこ取りをできる優れたものなら、なぜ他の言語はそれを採用していないのでしょうか。 Swift にそれができた理由は何でしょうか。筆者は、次の二つがその理由だと考えています。

- Swift の標準ライブラリではコレクションも _値型_
- Swift は _値型_ の使い勝手を向上させる言語仕様が充実

これらに基づいて Swift が _値型_ 中心の言語である理由を示し、また、 _値型_ に適したコードの書き方を説明します。

### Swift の標準ライブラリではコレクションも値型

多くの言語では `Array` や `Dictionary` などのコレクションは _参照型_ です。

たとえば、 `Python` の `List` （ Swift の `Array` に相当）は _参照型_ なので、次のようなことが起こります。

```py
# Python
a = [2, 3, 4]
b = a
a[2] = 5
print(b)  # [2, 3, 5]
```

_Reference Semantics_ を持っているので `a[2]` を変更すると `b[2]` も変更されました。

同じことを Swift の `Array` でやってみると次のようになります。

```swift
// Swift
var a = [2, 3, 4]
var b = a
a[2] = 5
print(b) // [2, 3, 4]
```

_Value Semantics_ を持っているので `a[2]` を変更しても `b[2]` には影響を与えません。このように、 Swift のコレクションは他の多くの言語のコレクションと異なります。

しかし、コレクションが _値型_ なのは一見すると合理的でないように思えます。 _値型_ のインスタンスは変数に代入されたり引数に渡されたりするとコピーされます。 100 万個の要素を持つ `Array` インスタンスが関数の引数に渡されただけで 100 万個丸ごとコピーされるようでは、パフォーマンスが悪すぎて使い物になりません。

```swift
let a = Array(1...1_000_000)
sum(of: a) // コピーされる？🤔
```

Swift のコレクションは、 **_Copy-on-Write_** という仕組みを使ってパフォーマンスの問題を克服しています。 _Copy-on-Write_ は、簡単に言うと必要なときだけコピーが行われる仕組みです。上記の例では、 `sum(of:)` に渡された `Array` インスタンスが `sum(of:)` の中で変更されることはないので 100 万個の要素のコピーは行われません。

本題から逸れてしまうので _Copy-on-Write_ の詳しい説明はここでは行いません。詳細は次の記事に譲ります。重要なのは、 **_Copy-on-Write_ によって Swift のコレクションはパフォーマンスと _Value Semantics_ を両立している** ということです。

- ["Swift での Copy on Write の実装方法の解説"](https://qiita.com/omochimetaru/items/f32d81eaa4e9750293cd), @omochimetaru, Qiita

コレクションを使わずにコードが書けるということは滅多にありません。コレクションが _参照型_ であるということは（イミュータブルなコレクションしか使わないというのでない限り） _Reference Semantics_ の引き起こす問題と付き合っていかないといけないということです。

たとえば、 C# は Swift と似た `struct` を持ちますが、コレクションは _参照型_ です。そのため、 _値型_ と _Value Semantics_ の世界で完結してコードを書こうとしても困難です。 _防御的コピー_ や _Read-only View_ を用いて _Reference Semantics_ と付き合っていくことになります。

Swift は _Copy-on-Write_ を用いることで _値型_ のコレクションを実現しました。それによって、 Swift では _値型_ と _Value Semantics_ 中心のコードを書くことが可能となっています。

### Swift は値型の使い勝手を向上させる言語仕様が充実

[前ページ]({{ prev_section.path }})で _ミュータブルクラス_ と _値型_ のコードを比較し、 _値型_ は _ミュータブルクラス_ と同等の変更の容易さを実現できると説明しました。しかし、[前ページ]({{ prev_section.path }})のような単純な例ではなく、より複雑なケースでは一筋縄にいかないこともあります。

Swift には、そのような場合でも対処するための、 _値型_ の使い勝手を向上させる言語仕様が充実しています。たとえ _値型_ 中心の標準ライブラリを提供していても、そのような言語仕様を欠いていると _値型_ 中心のコードを書くことは困難でしょう。ここでは、次の各項について、 Swift が _値型_ の使い勝手をどのように改善しているかを見ていきます。

- 安全な `inout` 引数
- `mutating func`
- Computed Property を介した変更
- 高階関数と `inout` 引数の組み合わせ

#### 安全な `inout` 引数

[前ページ]({{ prev_section.path }})では `User` にポイントを付与する例を取り上げましたが、逆にポイントを消費する例を考えてみましょう。

```swift
user.points -= 100
```

しかし、これではポイントが 0 より小さくなってしまうかもしれません。毎回チェックをするのは大変なので、ポイントが足りるかのチェックを含めて `consumePoints` という関数にしてみます。

`User` が _ミュータブルクラス_ の場合には次のように書けます。

```swift
// ミュータブルクラスの場合
func consumePoints(_ points: Int, of user: User) throws {
    guard user.points >= points else {
        throw PointsConsumptionError()
    }
    user.points -= points
}

let user: User = ...
try consumePoints(100, of: user)
```

`guard` 文でポイントが足りるかをチェックし、足りない場合はエラーを `throw` します。何の変哲もない関数です。

_値型_ で同じことをしようとするとどうなるでしょうか。 `User` が `struct` の場合、同様のコードはコンパイルエラーになります。

```swift
// 値型の場合
func consumePoints(_ points: Int, of user: User) throws {
    guard user.points >= points else {
        throw PointsConsumptionError()
    }
    user.points -= points // ⛔
}
```

引数で渡された `user` は定数扱いなので変更することができません。また、次のように一度変数に代入して変更しても、 `User` は _Value Semantics_ を持っているため、関数の呼び出し元に変更を反映することはできません。

```swift
// 値型の場合
func consumePoints(_ points: Int, of user: User) throws {
    var user: User = user // 変更できるように変数に代入してみたが
    guard user.points >= points else {
        throw PointsConsumptionError()
    }
    user.points -= points // 呼び出し元に変更が反映されない😵
}

var user: User = ...
try consumePoints(100, of: user) // ポイントは減らない😵
```

これを実現するための言語仕様が `inout` 引数です。 `inout` 引数を使えば、 _値型_ のための `consumePoints` を次のように書けます。

```swift
// 値型の場合
func consumePoints(_ points: Int, of user: inout User) throws {
    guard points <= user.points else {
        throw PointsShortError()
    }
    user.points -= points // ✅
}

var user: User = ...
try consumePoints(100, of: &user) // ポイントが減る😄
```

`inout` 引数は、引数に渡された値が関数の中で変更された場合、その変更を呼び出し元に反映します。そのため、呼び出し元の `user` に `consumePoints` の中での変更を反映できるわけです。

_ミュータブルクラス_ の場合のコードと比べても、 `consumePoints` では `inout` が、呼び出し元のコードでは `&` が増えただけで、コードを書く労力はほとんど変わりません。 `inout` や `&` の記述は手間を増やすものではなく、 _値型_ にとって不自然な挙動を引き起こすための明示的な意思表示として欠かせないものです。

`inout` 引数のような機能は、一歩間違えると _Value Semantics_ を破壊する危険なものです。 Swift は、 `inout` 引数を通して危険な操作ができないように注意深く設計されています。 `inout` は Swift では第一級ではなく、 `inout` な変数やプロパティを作ったり、 `inout` な戻り値の型を作ったりすることはできません。

```swift
// Swift ではこんなことはできない⛔
func foo(_ x: inout Int) -> inout Int {
    return &x
}

var a = 2
var b = &foo(&a)
a = 3
print(b) // 3
```

しかし、 PHP ではこれができてしまいます。

```php
// PHP
<?php
function &foo(&$x) {
  return $x;
}

$a = 2;
$b =&foo($a);
$a = 3;
echo $b; // 3
?>
```

このように、 Swift では利便性と安全性のバランスを考えて `inout` 引数が設計されています。

#### `mutating func`

`consumePoints` は関数よりもメソッドである方が自然です。

_ミュータブルクラス_ の場合、 `consumePoints` メソッドは次のように書けます。

```swift
// ミュータブルクラスの場合
extension User {
    func consumePoints(_ points: Int) throws {
        guard self.points >= points else {
            throw PointsConsumptionError()
        }
        self.points -= points
    }
}

let user: User = ...
try user.consumePoints(100)
```

_値型_ の `User` に `consumePoints` メソッドを実装する場合、 `mutating` 修飾子が必要になります。

```swift
// 値型の場合
extension User {
    mutating func consumePoints(_ points: Int) throws {
        guard points <= self.points else {
            throw PointsShortError()
        }
        self.points -= points
    }
}

var user: User = ...
try user.consumePoints(100)
```

この `mutating` 修飾子は何のためにあるのでしょうか。

そもそも、メソッドは暗黙的な引数 `self` を省略した関数とみなすことができます。

```swift
// 関数
func bar(_ self: Foo, _ x: Int) { ... }

// メソッド
extension Foo {
    // 暗黙の引数 `self` を持つ
    func bar(_ x: Int) { ... }
}
```

上記の関数とメソッドについて、次のように呼び出すと同じ結果が得られます。

```swift
// 関数
bar(foo, 42)

// メソッド
foo.bar(42)
```

メソッドにおける暗黙の引数 `self` という考え方は一般的なもので、 Python 等の一部の言語では、メソッドの宣言時に `self` を明記することが求められます。

```py
# Python
class Foo:
    def bar(self, x):  # 暗黙の引数 `self` を明記
        ...

foo.bar(42)  # 利用時には引数として渡す必要はない
```

関数とメソッドの関係をそのように捉えて二つの `consumePoints` を見比べてみます。

```swift
// 値型の場合

// 関数
func consumePoints(_ points: Int, of user: inout User) throws {
    ...
}
try consumePoints(100, of: &user)

// メソッド
extension User {
    mutating func consumePoints(_ points: Int) throws {
        ...
    }
}

try user.consumePoints(100)
```

そうすると、 `consumePoints` 関数の `inout` 引数が、 `consumePoints` メソッドの暗黙の引数 `self` に相当していることがわかります。 _値型_ の値に変更を加えて呼び出し元に変更を反映させるには `inout` 引数が必要でした。つまり、 `mutating func` は `self` に変更を加える必要がある場合に、暗黙の引数 `self` に `inout` を付与していると解釈することができます。

`inout` 引数に定数や関数の戻り値が渡せないのと同様に、定数や関数の戻り値に対して `mutating func` を呼び出すことはできません。たとえば、下記のコードでは `owner` プロパティに対して `consumePoints` を呼び出して状態を変更しようとしていますが、 `owner` プロパティは `let` で宣言されており変更できないので、関数とメソッドのどちらの場合もコンパイルエラーになります。

```swift
struct Group {
    let owner: User
    ...
}

var group: Group = ...
try consumePoints(100, of: &group.owner) // ⛔
try group.owner.consumePoints(100)       // ⛔
```

これは当たり前のようですが、 Swift と似た `struct` を持つ C# では異なる挙動をします。 C# は `inout` 引数相当の言語仕様を持ちますが、 `mutating func` に相当するものを持ちません。そのため、 `nonmutating` なメソッドと `mutating` なメソッドをコンパイラが区別して扱うことができません。これは値型定数（ C# では `readonly` ）のメソッドを呼び出したときの挙動について悩ましい問題を生みます。

もし、値型定数に対してもそのままメソッドを呼び出せてしまうと、定数であるにも関わらず状態が変更されてしまうかもしれません。しかし、 `mutating` なメソッドだけを区別して禁止することができないため、そのような変更を防ぐためには一律すべてのメソッド呼び出しを禁止する必要があります。さすがにそれは現実的ではありません。そこで、 C# では値型定数のメソッドを呼び出す場合、インスタンスの暗黙的なコピーを生成し、そのコピーに対してメソッドを実行するという戦略をとっています。そうすれば、 `mutating` なメソッドが定数を変更してしまうこともありませんし（変更結果はすぐに破棄される暗黙的なコピーにのみ反映され、オリジナルのインスタンスには影響を与えません）、定数に対するメソッド呼び出しを禁止する必要もありません。

しかし、それは定数に対する `mutating` なメソッド呼び出しもコンパイルエラーにならないことを意味します。定数に対して `mutating` なメソッドを呼び出した場合、実行されても何も起こりません。先のコードと同じようなコードを C# で書くと、 `group.owner.ConsumePoints(100)` はコンパイルエラーにならず、実行してもポイントを消費することもありません。

```cs
// C#
public struct Group {
    public readonly User owner;
    ...
}

ConsumePoints(100, ref group.owner); // ⛔
group.owner.ConsumePoints(100); // コンパイルが通り、実行しても何も起こらない😵
```

これはミスリーディングでわかりづらい挙動ではないでしょうか。 Swift は `inout` 引数同様に `mutating func` を通常のメソッドと区別して扱うことで、このような問題を防止しています。

#### Computed Property を介した変更

Swift では、 Stored Property と同じように Computed Property を扱うことができます。 Computed Property を介して状態を変更することも可能です。

先程の `owner` プロパティが Computed Property だったとしましょう。

```swift
struct Group {
    var owner: User {
        get { members[0] }
        set { members[0] = newValue }
    }
    ...
}
```

この場合でも、 `owner` プロパティを介して状態を変更することができます。

```swift
group.owner.points = 0 // ✅
```

これができるのは自然なことに思えるかもしれませんが、 _値型_ についてはそれほど自然なことではありません。 Computed Property はプロパティのふりをしたメソッドのようなものです。 `owner` が Computed Property ではなくメソッドだったらと考えると、 `group.owner.points = 0` で `group.owner` を変更できることが不自然に思えてきます。。

```swift
// owner がメソッドだったら
struct Group {
    func owner() -> User {
        members[0]
    }
}
```

`User` が _値型_ の場合、 `owner` メソッドの戻り値に対して変更を加えようとするとコンパイルエラーになります。

```swift
// 値型の場合
group.owner().points = 0 // ⛔
```

`User` は _値型_ なので、 `group.owner()` の戻り値は `return` される際にコピーして生成されます。生成された `User` インスタンスは、 `group.owner().points = 0` が実行される間、一時的にメモリ上に存在するだけです。これは、 `let a = (3 + 5) + 7` のような計算を実行したときに、 `3 + 5` の計算結果である `8` が一時的にメモリ上に存在し、 `8 + 7` が計算された後に破棄されるのと同じです。一時的に存在するだけの `User` インスタンスに変更を加えても（ `points` を `0` にしても）、その結果は破棄されてしまうので意味がありません。そのような無意味なケースを Swift コンパイラはコンパイルエラーとします。

では、 `owner` が Computed Property の場合には、どうして `points` の変更結果を `group` に反映することができるのでしょうか。実行時に何が行われているのでしょう。 `owner` が Computed Property の場合、次のような手順で処理が実行されます。

1. `group.owner` の `get` を用いて `User` インスタンスのコピーが返される。
2. そのコピーの `points` が `0` に変更される。
3. 変更を加えられたコピーが `group.owner` の `set` の `newValue` として渡され、変更が `group` に反映される。

`group.owner().points = 0` という一つの式が実行される間に `owner` プロパティの `get` と `set` がそれぞれ異なるタイミングで呼び出されることによって、 Computed Property を介した状態の変更が実現されているわけです。

`inout` 引数や `mutating func` 同様に、これも _ミュータブルクラス_ 相当の使い勝手の良さを _値型_ で実現するために欠かせない言語仕様です。もし `User` が _ミュータブルクラス_ であれば、 Computed Property を介した状態の変更は当たり前のようにできます。 `User` が _参照型_ なら、 Computed Property によって返されるインスタンスは `group` が内部で保持しているインスタンスと共有されており、同一です。 `owner` プロパティに返されたインスタンスに変更を加えるということは、 `group` が内部に保持しているインスタンスに変更を加えることです。 `owner` プロパティの `set` すら必要ありません。 Stored Property であろうと Computed Property であろうと、それらを介して変更を加えられることは _ミュータブルクラス_ にとっては当然のことです。 _値型_ においてもこれと同じ使い勝手を実現するためには、 Computed Property を介しても変更を反映できる仕組みが必要なわけです。

Computed Property に加えて、 `subscript` を介した変更も同じように動作します。

```swift
group.members[i].points = 0 // ✅
```

これも `User` が _値型_ のときには、 `subscript` の `get` 、 `points` の変更、 `subscript` の `set` の順に実行されます。

Computed Property や `subscript` を介した変更は、 `inout` 引数や `mutating func` と組み合わせることで更に強力に機能します。

```swift
// 値型の場合

// inout 引数との組み合わせ
try consumePoints(100, of: &group.owner) // ✅

// mutating func との組み合わせ
try group.owner.consumePoints(100) // ✅
```

この特徴を活かした API に、 `Bool` の `toggle` メソッドがあります（[API リファレンス](https://developer.apple.com/documentation/swift/bool/2994863-toggle)）。 `toggle` は `Bool` の `mutating func` で、自身が `true` なら `false` に、 `false` なら `true` に自分自身を変更します。下記のコードのメソッドチェーンのどこかに Computed Property が挟まっていたとしても、このコードは `isPublic` の状態を変更することができます。

```swift
foo.bar.baz.items[i].isPublic.toggle() // ✅
```

Swift を使っていると当然のようにこれらの恩恵を受けることができますが、他の言語と比較してみると、それがいかに _値型_ の使い勝手を向上させているかがわかります。たとえば、 C# ではどのパターンも Swift と同じような挙動にはなりません。

```cs
// C#
group.Owner.Points = 0; // コンパイルエラー⛔
ConsumePoints(100, ref group.Owner); // コンパイルエラー⛔
group.Owner.ConsumePoints(100); // コンパイルが通り、実行しても何も起こらない😵
```

#### 高階関数と `inout` 引数の組み合わせ

これまでは一人の `User` に対する変更を例に様々なパターンを見てきましたが、ここでは複数の `User` 、つまり `[User]` に対する変更を考えてみましょう。

`User` が _ミュータブルクラス_ の場合、特別なことは何もありません。 `for-in` ループで `User` を一人ずつ取り出して変更を加えるだけです。 `[User]` に格納された全 `User` に 100 ポイントを付与するコードは次のようになります。

```swift
// ミュータブルクラスの場合
let users: [User] = ...
for user in users {
    user.points += 100
}
```

`User` が _値型_ のとき、これと同じことをやろうとすると一筋縄ではいきません。まず、 `for-in` ループのループ変数はデフォルトで定数なので、そのままではコンパイルエラーになります。

```swift
// 値型の場合
var users: [User] = ...
for user in users {
    user.points += 100 // ⛔
}
```

だからといってこれを変数にすれば良いわけではありません。 `User` は _Value Semantics_ を持っているので、ループ変数 `user` を変更したからといって、それが `users` に反映されるわけではありません。

```swift
// 値型の場合
var users: [User] = ...
for var user in users {
    user.points += 100 // コンパイルが通り、実行しても何も起こらない😵
}
```

このようなケースでの一般的な対処法は、インデックスを介して状態を変更することでしょう。

```swift
// 値型の場合
var users: [User] = ...
for i in users.indices {
    users[i].points += 100 // ✅
}
```

しかし、別の方法もあります。それが、次のような `modifyEach` メソッドを導入することです。

```swift
extension Array {
    mutating func modifyEach(_ body: (inout Element) -> Void) {
        for i in indices {
            body(&self[i])
        }
    }
}
```

この `modifyEach` メソッドを使うと、次のようにして `[User]` 中の全 `User` の状態を変更することができます。

```swift
// 値型の場合
var users: [User] = ...
users.modifyEach { user in
    user.points += 100
}
```

`modifyEach` メソッドは引数 `body` にクロージャを受け取る高階メソッドですが、おもしろいのは `body` が `inout` 引数を持っていることです。 `modifyEach` に渡すクロージャは引数として `user` を受け取るわけですが、これが `inout` 引数になっており、クロージャの中で `user` に加えた変更は `users` に反映されるわけです。

残念ながら `modifyEach` は [swift-evolution で議論されたこと](https://forums.swift.org/t/in-place-map-for-mutablecollection/11111)はありますが、標準ライブラリに加えられるには至っていません。しかし、 Swift 4 で追加された `reduce(into:_:)` という高階メソッド（[API リファレンス](https://developer.apple.com/documentation/swift/sequence/3128812-reduce)）は `modifyEach` と同じように `inout` 引数との組み合わせを活用しています。

```swift
func reduce<Result>(
    into initialResult: Result,
    _: (inout Result, Element) throws -> ()
) rethrows -> Result
```

従来の `reduce` メソッドは関数型言語由来で _値型_ との相性がよくありませんでした。

```swift
let idToUser: [Int: User] = users.reduce([:]) { result, user in
    var result = result
    result[user.id] = user // O(N): ここで Dictionary 全体をコピー😵
    return result
}
```

`reduce(into:_:)` の追加により _値型_ との親和性が高まりました。

```swift
let idToUser: [Int: User] = users.reduce(into: [:]) { result, user in
    result[user.id] = user // O(1) 😄
}
```

なお、 `modifyEach` 相当のことは、 [Ownership Manifesto の中で示されている](https://github.com/apple/swift/blob/master/docs/OwnershipManifesto.md#mutating-iteration) `for-in` ループと `inout` の組み合わせで将来的にできるようになる可能性があります。

```swift
// 値型の場合（将来的にできるようになるかもしれないこと）
var users: [User] = ...
for inout user in users {
    user.points += 100 // ✅
}
```

### 値型とミュータブルクラスの使い分け

_値型_ を使うことで _ミュータブルクラス_ と _イミュータブルクラス_ のいいとこ取りが可能となり、 Swift には _値型_ の使い勝手を向上する言語仕様が充実していることを見てきました。しかし、 _値型_ も万能ではありません。すべてを _値型_ にすることが必ずしも望ましいわけではなく、 _値型_ に固執しすぎるとかえってコードを複雑化してしまうでしょう。適切に _値型_ と _参照型_ と使い分けることが重要です。

_参照型_ といっても、 Swift において自分自身で _イミュータブルクラス_ を実装すべきケースはほとんどありません。 _イミュータブルクラス_ がほしくなった場合には `struct` や `enum` などの _値型_ で代替できないかを考えてみるのが良いでしょう。注意が必要なのは、ある機能を持った型を _イミュータブルクラス_ の代わりに `struct` で実装しようとする場合、必ずしもその `struct` は _イミュータブル_ でなくて良いということです。 _イミュータブルクラス_ がほしい場合、実はその理由のほとんどは _Value Semantics_ がほしいということです。多くの言語で `String` は _イミュータブルクラス_ ですが、 Swift の `String` は _ミュータブル_ な `struct` です。 Swift の `String` はそれでも問題なく機能しています。それは、 `String` に _イミュータブル_ であってほしかった理由が _Value Semantics_ がほしいということだったからです。

問題は _値型_ と _ミュータブルクラス_ をどのように使い分けるかです。より正確に言うと、これは _Value Semantics_ と _Reference Semantics_ をどのように使い分けるかという問題です。 ["Value Semantics とは"](./) で見たように、 _Value Semantics_ も _Reference Semantics_ も持たない型は使い勝手が良くありません。また、 _Value Semantics_ と _Reference Semantics_ を兼ね備えた _イミュータブル_ な型を作る場合は、基本的には _値型_ にすれば問題ありません。そうすると、 _値型_ か _ミュータブルクラス_ か考えなければならないのは、 _ミュータブル_ な型に _Value Semantics_ を持たせるのか _Reference Semantics_ を持たせるのかという場合です。

そのような場合にどちらが望ましいかはケース・バイ・ケースです。一概に線引きをするのは困難です。判断に迷った場合に筆者がおすすめする方法は、まず _値型_ × _Value Semantics_ で実装し、それでうまくいかない場合に _ミュータブルクラス_ × _Reference Semantics_ を検討するというものです。 _Reference Semantics_ の利用を最小限に留めることで、 _Reference Semantics_ が引き起こす問題を最小化し、問題のコントロールが容易になります。

そのときに、 _Value Semantics_ の世界と、 _Reference Semantics_ の世界を分離しておくことが重要です。 _Value Semantics_ の世界では _Reference Semantics_ の問題を気にする必要がありません。 _Value Semantics_ の世界をどれだけ広げられるかというのが、設計における一つのポイントになるでしょう。これは、 _参照型_ 中心の言語で _イミュータブル_ な世界を分離して、できるだけ広げる努力をするのと本質的に同じことです。しかし、 _値型_ には _イミュータブルクラス_ よりも状態の変更が容易であるという特性があるため、変更の煩雑さを気にせず、より積極的に _Value Semantics_ を採用しやすいでしょう。

### まとめ

_値型_ は _ミュータブルクラス_ と _イミュータブルクラス_ のいいとこ取りをした存在であり、 Swift はその _値型_ を中心とした言語です。 _値型_ 中心の言語はめずらしいのですが、 Swift でそれが可能だったのは、 _Copy-on-Write_ を使って効率よく _値型_ のコレクションを実装できたことと、 _値型_ を使ったコードを書く場合に問題になりがちな点をカバーし、 _値型_ の使い勝手を向上させる言語仕様のおかげだと筆者は考えています。

ここでは、 _値型_ の使い勝手を向上させる言語仕様として、次の四つを挙げました。

- 安全な `inout` 引数
- `mutating func`
- Computed Property を介した変更
- 高階関数と `inout` 引数の組み合わせ

Swift は進化の途上であり、 `for-in` ループと `inout` の組み合わせなど、今後さらに _値型_ の使い勝手が改善されていくものと思われます。

ただし、 _値型_ も万能ではありません。 _値型_ × _Value Semantics_ と _ミュータブルクラス_ × _Reference Semantics_ を適切に使い分ける必要があります。どちらを採用すべきかわからないときは、まず _値型_ × _Value Semantics_ で始め、問題が生じたら _参照型_ × _Reference Semantics_ を検討することをおすすめします。

---

- 前のページ: {% include section-link.md section=prev_section %}
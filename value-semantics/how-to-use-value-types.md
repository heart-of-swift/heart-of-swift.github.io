---
layout: section
title: "Swift が値型中心の言語になれた理由とその使い方"
chapter_index: 0
section_index: 2
---

[前ページ]({{ prev_section.path }})で見たように、 Swift の標準ライブラリで提供される型はほぼすべてが _値型_ で _Value Semantics_ を持っています。このような言語は稀です。 _値型_ が _ミュータブルクラス_ と _イミュータブルクラス_ のいいとこ取りをできる優れものなら、なぜ他の言語はそれを採用しないのでしょうか。 Swift にそれができた理由は何でしょうか。筆者は、次の二つがその理由だと考えています。

- Swift の標準ライブラリはコレクションも _値型_
- Swift は _値型_ の使い勝手を向上させる言語仕様が豊富

これらに基づいて Swift が _値型_ 中心の言語になれた理由を示し、また、 _値型_ に適したコードの書き方を説明します。

## Swift の標準ライブラリはコレクションも値型

多くの言語では、 `Array` や `Dictionary` などのコレクションは _参照型_ です。

たとえば、 `Python` の `List` （ Swift の `Array` に相当）は _参照型_ なので、次のようなことが起こります。

```py
# Python
a = [2, 3, 4]
b = a
a[2] = 5
print(b)  # [2, 3, 5]
```

_Reference Semantics_ を持っているので `a[2]` を変更すると `b[2]` も変更されました。

同じことを Swift の `Array` ですると次のようになります。

```swift
// Swift
var a = [2, 3, 4]
var b = a
a[2] = 5
print(b) // [2, 3, 4]
```

_Value Semantics_ を持っているので `a[2]` を変更しても `b[2]` には影響を与えません。このように、 Swift のコレクションは他の多くの言語のコレクションと異なります。

しかし、コレクションが _値型_ なのは一見すると合理的でないように思えます。 _値型_ のインスタンスは変数に代入されたり引数に渡されたりする度にコピーされます。 100 万個の要素を持つ `Array` インスタンスが関数の引数に渡されただけで 100 万個丸ごとコピーされるようでは、パフォーマンスが悪すぎて使い物になりません。

```swift
let a = Array(1...1_000_000)
sum(of: a) // コピーされる？🤔
```

Swift のコレクションは、 **_Copy-on-Write_** という仕組みを使って _Value Semantics_ とパフォーマンスを両立しています。 _Copy-on-Write_ は、簡単に言うと必要なときだけコピーが行われる仕組みです。上記の例では、 `sum(of:)` に渡された `Array` インスタンスが `sum(of:)` の中で変更されることはないので、 `Array` が共有されても変更の独立性に影響を与えません。そのためコピーは必要なく、省略されます。 _Copy-on-Write_ の挙動と仕組みの詳細については ["付録" > "Copy-on-Write"](/appendix/copy-on-write) を御覧下さい。

コレクションを使わずにコードが書けることは滅多にありません。コレクションが _参照型_ であるということは（ _イミュータブル_ なコレクションしか使わないというのでない限り）、「意図しない変更」や「整合性の破壊」のような問題と付き合っていかなければならないということです。

たとえば、 C# は Swift と似た `struct` を持ちますが、コレクションは _参照型_ です。そのため、 _値型_ 中心のコードを書こうとしてもすぐに _参照型_ が混入してしまいます。 Swift は _Copy-on-Write_ を用いることで _値型_ のコレクションを実現しました。コレクションも _値型_ なので、 Swift では _値型_ と _Value Semantics_ 中心のコードを書くことが可能なわけです。

## Swift は値型の使い勝手を向上させる言語仕様が豊富

 _値型_ 中心の標準ライブラリを提供するだけで _値型_ 中心の言語が実現されるわけではありません。いくら標準ライブラリの型のほとんどが _値型_ でも、肝心の _値型_ の使い勝手が悪いと単に不便なだけです。

[前ページ]({{ prev_section.path }})で _ミュータブルクラス_ と _値型_ のコードを比較し、 _値型_ は _ミュータブルクラス_ のように変更が容易であると説明しました。しかし、[前ページ]({{ prev_section.path }})のような単純なケースではなく、より複雑なケースでは一筋縄にいかないこともあります。 Swift にはそのようなケースに対処するための言語仕様が豊富に備わっており、 _値型_ の使い勝手向上につながっています。

ここでは、次の各項について Swift が _値型_ の使い勝手をどのように改善しているかを見ていきます。

- 安全な `inout` 引数
- `mutating func`
- Computed Property を介した変更
- 高階関数と `inout` 引数の組み合わせ

### 安全な `inout` 引数

[前ページ]({{ prev_section.path }})では `User` にポイントを付与する例を取り上げました。ここでは、逆にポイントを消費する例を考えてみましょう。単純に考えれば次のようになります。

```swift
user.points -= 100
```

しかし、これではポイントが 0 より小さくなってしまうかもしれません。ポイントを消費するコードを書く度に、ポイントが足りているかチェックするのは大変です。チェックを含めて `consumePoints` という関数にしましょう。

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

`guard` 文でポイントが足りるかをチェックし、足りない場合はエラーを `throw` します。ポイントが足りるなら `user.points -= points` でポイントを減らします。

_値型_ の `User` に対して同じことをしようとするとどうなるでしょうか。 `User` が `struct` の場合、同様のコードはコンパイルエラーになります。引数で渡された `user` は定数扱いなので変更することができません。

```swift
// 値型の場合
func consumePoints(_ points: Int, of user: User) throws {
    guard user.points >= points else {
        throw PointsConsumptionError()
    }
    user.points -= points // ⛔
}
```

では、次のように一度変数に代入してから変更すれば良いのでしょうか。しかし、このコードでは `user.points -= points` で行われた変更を関数の呼び出し元に反映することはできません。

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

`var user: User = user` で `User` インスタンスがコピーされているからです。

この問題を解決してくれるのが `inout` 引数です。 `inout` 引数を使えば、 _値型_ のための `consumePoints` を次のように書けます。

```swift
// 値型の場合
func consumePoints(_ points: Int, of user: inout User) throws {
    guard points <= user.points else {
        throw PointsShortError()
    }
    user.points -= points // コンパイルが通り、変更も反映される✅
}

var user: User = ...
try consumePoints(100, of: &user) // ポイントが減る😄
```

`inout` 引数は、引数に渡された値が関数の中で変更された場合、その変更を呼び出し元に反映します。そのため、 `consumePoints` の中で `user` のポイントを減らすと、それを呼び出し元の `User` インスタンスのポイントを減らすことができます。

`User` が _ミュータブルクラス_ の場合のコードと比べても、関数の宣言部で `inout` が、呼び出し元のコードで `&` が増えただけで、コードを書く労力はほとんど変わりません。 `inout` や `&` を記述しなければならないのも、手間が増えたというより、 _値型_ にとって不自然な挙動を引き起こすための明示的な意思表示と考えればむしろ有用なものと言えます。

「 _値型_ にとって不自然な挙動」と書きましたが、 `inout` 引数のような機能は、一歩間違えると _Value Semantics_ を破壊する危険なものです。 Swift は、 `inout` 引数を通して危険な操作ができないように注意深く設計されています。 Swift の `inout` は第一級ではなく、 `inout` な変数やプロパティを作ったり、 `inout` な戻り値を作ったりすることはできません。

```swift
// Swift ではこんなことはできない⛔
func foo(_ x: inout Int) -> inout Int {
    return &x
}

var a = 2
var b = &foo(&a)
a = 3    // b も変更される
print(b) // 3
```

PHP ではこれができるため、注意深く使わないと _Value Semantics_ を破壊してしまうことがあります。

```php
<?php
function &foo(&$x) {
  return $x;
}

$a = 2;
$b = &foo($a);
$a = 3;  // b も変更される
echo $b; // 3
?>
```

このように、 `inout` は便利ですが、一方で _Value Semantics_ を破壊する可能性を秘めた諸刃の剣でもあります。  Swift の `inout` 引数は _Value Semantics_ を破壊しないように注意深く設計されているため、安心して利用することができます。

### `mutating func`

`consumePoints` を関数ではなくメソッドとして実装することを考えてみましょう。

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

_値型_ では、メソッドがインスタンスに変更を加える場合に `mutating` 修飾子が必要になります。 `consumePoints` メソッドも `points` プロパティに変更を加えるため、 `mutating func` でなければなりません。

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

この `mutating` 修飾子は何のためにあるのでしょうか。さっき作った関数の `consumePoints` と、今作ったメソッドの `consumePoints` を見比べてみましょう。

結論を出す前に、急がば回れでもう少し根本的なところから考えてみます。一度 `inout` 引数や `mutating` 修飾子のことは置いておいて、普通の関数とメソッドのことを考えてみましょう。関数とメソッドにはどのような関係があるのでしょうか。

メソッドは、暗黙的な引数 `self` を省略した関数と考えることができます。次のコードを見てみて下さい。

```swift
// 関数
func bar(_ self: Foo, _ x: Int) {
    print(self.value + x)
}

// メソッド
extension Foo {
    // 暗黙の引数 `self` を持つ
    func bar(_ x: Int) {
        print(self.value + x)
    }
}
```

↓は、上記の `bar` 関数と `bar` メソッドを使うコードです。次のように呼び出すと同じ結果が得られます。

```swift
let foo: Foo = ...

// 関数
bar(foo, 42)

// メソッド
foo.bar(42)
```

このように見比べると、メソッドでは `foo.` と書くことで、 `foo` が暗黙的な引数 `self` として `bar` メソッドに渡されていると解釈することができます。この考え方は突飛なものではなく、 Python 等の一部の言語ではメソッドの宣言時に引数 `self` を明記します。

```py
# Python
class Foo:
    def bar(self, x):  # 暗黙の引数 `self` を明記
        print(self.value + x)
    ...

foo.bar(42)  # 利用時には引数として渡す必要はない
```

さて、これで関数とメソッドの関係がわかりました。それでは、本題に戻って二つの `consumePoints` を見比べてみましょう。

```swift
// 値型の場合

// 関数
func consumePoints(_ points: Int, of user: inout User) throws {
    guard points <= user.points else {
        throw PointsShortError()
    }
    user.points -= points
}
try consumePoints(100, of: &user)

// メソッド
extension User {
    mutating func consumePoints(_ points: Int) throws {
        guard points <= self.points else {
            throw PointsShortError()
        }
        self.points -= points
    }
}

try user.consumePoints(100)
```

そうすると、関数版の引数 `user` が、メソッド版の暗黙の引数 `self` に相当していることがわかります。しかし、 `user` はただの引数ではありません。 `inout` 引数です。つまり、 `mutating` 修飾子は、暗黙の引数 `self` に `inout` を付けることに対応していると考えられます。

`inout` 引数を使うと、関数の中で加えられた変更を呼び出し元に反映することができました。 `mutating` なメソッドも同様です。メソッドの中で `self` に加えられた変更を、呼び出し元のインスタンスに反映することができます。 `mutating` を付与することは、暗黙の引数 `self` に `inout` を付けるのと同じ意味を持つわけです。

これで `mutating` 修飾子の役割はわかりました。以下では、さらにもう少し `mutating` を掘り下げみます。

`inout` 引数に定数を渡せないのと同じように、定数に対して `mutating func` を呼び出すことはできません（より正確には、定数だけでなく _左辺値_ でない値を渡すことはできません）。たとえば下記のコードでは、 `let` で宣言されている `owner` プロパティに対して `consumePoints` を呼び出しているので、関数・メソッドのどちらの場合でもコンパイルエラーになります。

```swift
struct Group {
    let owner: User
    ...
}

var group: Group = ...
try consumePoints(100, of: &group.owner) // ⛔
try group.owner.consumePoints(100)       // ⛔
```

この挙動は当たり前に思えるかもしれませんが、 C# では異なる挙動をします。同様のコードを C# で書くと、関数の場合はコンパイルエラーになりますが、メソッドの場合はコンパイルエラーになりません。コードを実行することができます。しかし、実行してもポイントは減りません。

```cs
// C#
public struct Group {
    public readonly User owner;
    ...
}

ConsumePoints(100, ref group.owner); // ⛔
group.owner.ConsumePoints(100); // コンパイルが通り、実行してもポイントは減らない😵
```

どうしてこんなことが起こるのでしょうか。

C# は `inout` 引数相当の言語仕様を持ちますが、 `mutating func` に相当するものを持ちません。そのため、 `nonmutating` なメソッドと `mutating` なメソッドをコンパイラが区別して扱うことができません。これは _値型_ 定数（ C# では `readonly` ）のメソッドを呼び出した場合の挙動について、次のような悩ましい問題を生みます。

`nonmutating` なメソッドと `mutating` なメソッドを区別できないということは、あるメソッドがあったときに、そのメソッドが `self` を変更し得るかどうかコンパイラにはわからないということです。では、 _値型_ 定数に対するメソッド呼び出しを、コンパイラはどのように扱えば良いでしょうか。もしそのメソッドが `mutating` だと、定数が変更されてしまうかもしれません。そんなことは許されません。 _値型_ 定数に対するメソッド呼び出しをすべてコンパイルエラーにしてしまえば、定数が変更される事態は避けられます。しかし、それでは `self` を変更しない `nonmutating` なメソッドも、すべて禁止されることになってしまいます。さすがにそれは使い勝手の面で現実的ではありません。

そこで、 C# では _値型_ 定数のメソッドを呼び出す場合、次のような戦略をとっています。

1. インスタンスの暗黙的なコピーを生成する。
2. そのコピーに対してメソッドを呼び出す。

Swift のコードで表してみます。 `Foo` が _値型_ 、 `bar` が `mutating` なメソッドだとして、次のようなコードを考えます。

```swift
let foo: Foo = ...
foo.bar()
```

このとき、 C# では↓のように変換して実行されます。

```swift
let foo: Foo = ...
var _foo: Foo = foo // 1. 暗黙的なコピーを作成
_foo.bar()          // 2. コピーのメソッドを呼び出す
```

暗黙的なコピー `_foo` に対して加えられた変更は即座に破棄されるため、 `foo` が変更されてしまうことはありません。これによって、

- _値型_ 定数に対してもメソッドを呼び出せること
- _値型_ 定数が変更されないこと

を両立しているのです。

しかし、 C# のこの仕様は、 _値型_ 定数に対して `mutating` なメソッドを呼び出すと、インスタンスの状態変更が無視されるという結果を生みます。

_値型_ 定数に対して `mutating` なメソッドを呼び出しているのはコードのミスである可能性があり、コンパイラによって検出されるのが望ましいです。 C# の仕様では、そのようなミスがコンパイルエラーどころか実行時エラーにもならずに黙殺されます。これは、原因究明の難しいバグにつながりかねません。 Swift では、 `mutating func` が通常のメソッドと区別され扱われることで、このようなミスがコンパイルエラーとして検出されます。

`mutating func` は Swift を学んだ人であれば誰でも知っているような基礎的なものです。しかしこのように掘り下げてみると、 _値型_ とうまく付き合うために大きな役割を果たしてくれていることがわかります。

### Computed Property を介した変更

Swift では、 Computed Property を Stored Property と同じように扱うことができます。 Computed Property を介して状態を変更することも可能です。

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

これができるのは自然なことに思えるかもしれません。しかし、 _値型_ にとってこれはそれほど自然なことではありません。

Computed Property はプロパティのふりをしたメソッドのようなものです。 `owner` が Computed Property ではなくメソッドだったらと考えると、上記のコードで状態を変更できることが不自然に思えてきます。

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

今 `User` は _値型_ なので、 `group.owner()` の戻り値は `return` される際にコピーされたものです。コピーされた `User` インスタンスは、 `group.owner().points = 0` が実行される間、一時的にメモリ上に存在するだけです。一時的な存在の `User` インスタンスに変更を加えても（ `points` を `0` にしても）、その結果は破棄されてしまうので意味がありません。そのような無意味なケースを Swift コンパイラはコンパイルエラーとして検出します。

では、 `owner` が Computed Property の場合には、どうして `points` の変更結果を `group` に反映することができるのでしょうか。それは、たとえば `group.owner.points = 0` の実行時には次のような手順で処理が行われるからです。

1. `group.owner` の `get` を用いて `User` インスタンスのコピーが返される。
2. そのコピーの `points` が `0` に変更される。
3. 変更を加えられたコピーが `group.owner` の `set` の `newValue` として渡され、変更が `group` に反映される。

`group.owner.points = 0` という一つの式が実行される間に `owner` プロパティの `get` と `set` がそれぞれ異なるタイミングで呼び出されることによって、 Computed Property を介した状態の変更が実現されているわけです。

`inout` 引数や `mutating func` 同様に、これも _値型_ の使い勝手を向上させるために欠かせない言語仕様です。 `User` が _ミュータブルクラス_ であれば、 `group.owner.points = 0` ができるのは当たり前です。 `User` が _参照型_ であれば、 `owner` プロパティが返すインスタンスは `group` と共有されることになります。そのインスタンスに変更を加えれば、その変更は当然 `group` とも共有されます。 _値型_ においても同じように `group.owner.points = 0` と書けるようにするには、 Computed Property を介しても変更を反映できる仕組みが必要なわけです。

Computed Property に加えて、 `subscript` を介した変更も同じように動作します。

```swift
group.members[i].points = 0 // ✅
```

これも `User` が _値型_ のときには、

1. `subscript` の `get`
2. `points` の変更
3. `subscript` の `set`

の順に実行されます。

さらに、 Computed Property や `subscript` を介した変更は、 `inout` 引数や `mutating func` と組み合わせることもできます。

```swift
// 値型の場合

// inout 引数との組み合わせ
try consumePoints(100, of: &group.owner) // ✅

// mutating func との組み合わせ
try group.owner.consumePoints(100) // ✅
```

Swift を使っていると当然のようにこれらの恩恵を受けていますが、他の言語と比較してみると、それがいかに _値型_ の使い勝手を向上させているかがわかります。たとえば、 C# ではどのパターンも Swift と同じような挙動にはなりません。

```cs
// C#
group.Owner.Points = 0; // コンパイルエラー⛔
ConsumePoints(100, ref group.Owner); // コンパイルエラー⛔
group.Owner.ConsumePoints(100); // コンパイルが通り、実行しても何も起こらない😵
```

### 高階関数と `inout` 引数の組み合わせ

これまでの例では単一の `User` に対する変更を取り上げてきました。ここでは複数の `User` 、つまり `[User]` に対する変更を考えてみましょう。

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

だからといって `user` を変数にすれば良いわけではありません。 `var user` にしてループの中で `user` を変更したからといって、それが `users` に反映されるわけではありません。 `users` から一人のデータをコピーして `user` に格納しているだけだからです。コピーに変更を加えても、オリジナルには反映されません。

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
    users[i].points += 100 // コンパイルが通り、変更も反映される✅
}
```

しかし、別の方法もあります。それは、次のような `modifyEach` メソッドを導入することです。

```swift
extension MutableCollection {
    mutating func modifyEach(_ body: (inout Element) -> Void) {
        for i in indices {
            body(&self[i])
        }
    }
}
```

この `modifyEach` メソッドを用いると、次のようにして `[User]` に格納された全 `User` インスタンスの状態を変更することができます。

```swift
// 値型の場合
var users: [User] = ...
users.modifyEach { user in
    user.points += 100 // コンパイルが通り、変更も反映される✅
}
```

`modifyEach` メソッドは引数 `body` にクロージャを受け取る高階メソッドです。 `map` や `filter` などの仲間と考えることができます。おもしろいのは `body` が `inout` 引数を持っていることです。上記の例では、 `modifyEach` に渡すクロージャは引数 `user` を受け取りますが、これが `inout` 引数になっています。そのため、クロージャの中で `user` に加えられた変更が `users` に反映されるわけです。

残念ながら `modifyEach` については [swift-evolution で議論されたこと](https://forums.swift.org/t/in-place-map-for-mutablecollection/11111)はあるものの、標準ライブラリに加えられるには至っていません。しかし、 Swift 4 で追加された `reduce(into:_:)` というメソッド（[API リファレンス](https://developer.apple.com/documentation/swift/sequence/3128812-reduce)）は `modifyEach` と同じように `inout` 引数と高階メソッドを組み合わせたものです。

```swift
extension Sequence {
    func reduce<Result>(
        into initialResult: Result,
        _: (inout Result, Element) throws -> ()
    ) rethrows -> Result {
        ...
    }
}
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

このように、高階関数（メソッド）と `inout` 引数を組み合わせは、実際に Swift 標準ライブラリの中でも活用されています。

なお、 `modifyEach` 相当のことは、将来的には [Ownership Manifesto の中で示されている](https://github.com/apple/swift/blob/master/docs/OwnershipManifesto.md#mutating-iteration) `for-in` ループと `inout` の組み合わせでできるようになる可能性があります。

```swift
// 値型の場合（将来的にできるようになるかもしれないこと）
var users: [User] = ...
for inout user in users {
    user.points += 100 // コンパイルが通り、変更も反映される✅
}
```

## 値型とミュータブルクラスの使い分け

これまで、 _値型_ を使うことで _ミュータブルクラス_ と _イミュータブルクラス_ のいいとこ取りができること、 Swift には _値型_ の使い勝手を向上する言語仕様が充実していることを見てきました。しかし、 _値型_ も万能ではありません。すべてを _値型_ にすることが必ずしも望ましいわけではありません。 _値型_ に固執しすぎるとかえってコードを複雑化してしまうこともあります。 _値型_ と _参照型_ を適切に使い分けることが重要です。

_参照型_ といっても、 Swift において _イミュータブルクラス_ を作るべきケースはほとんどありません。 _イミュータブルクラス_ がほしくなったら `struct` や `enum` などの _値型_ で代替できないか考えてみるのが良いでしょう。注意が必要なのは、何かの型を _イミュータブルクラス_ の代わりに `struct` で実装しようとする場合、必ずしもその `struct` は _イミュータブル_ でなくても良いということです。

_イミュータブルクラス_ が求められる最大の理由は _Value Semantics_ です。たとえば、多くの言語で `String` は _イミュータブルクラス_ ですが、それは _Value Semantics_ のためです。 Swift の `String` は _ミュータブル_ な `struct` ですが、それが問題になったという話は聞いたことがありません。 Swift の `String` は _ミュータブル_ でも _Value Semantics_ を持っているからです。 `String` に求めらているのは _イミュータビリティ_ ではなく _Value Semantics_ なのです。

問題は _値型_ と _ミュータブルクラス_ をどのように使い分けるかです。これは、言い換えると _Value Semantics_ と _Reference Semantics_ をどのように使い分けるかという問題です。

分類上は、次のような 4 種類の型を考えることができます。

<table>
<tr><th></th><th><em>Value Semantics</em> を持つ</th><th><em>Value Semantics</em> を持たない</th></tr>
<tr><th><em>Reference Semantics</em> を持つ</th><td><strike><em>イミュータブルクラス</em></strike></td><td><em>ミュータブルクラス</em></td></tr>
<tr><th><em>Reference Semantics</em> を持たない</th><td>（普通の） <em>値型</em></td><td><strike>使い勝手が良くない</strike></td></tr>
</table>

しかし、前述のように、 _イミュータブルクラス_ は _値型_ に置き換えれば良いことがほとんどです。また、 ["Value Semantics とは"](./) で見たように、 _Value Semantics_ も _Reference Semantics_ も持たない型は使い勝手が良くありません。そう考えると、 _値型_ か _ミュータブルクラス_ かの選択というのは、 _ミュータブル_ な型に _Value Semantics_ を持たせるのか _Reference Semantics_ を持たせるのかの選択だと言えます。

そのような場合にどちらが望ましいかはケース・バイ・ケースで、一概に線引きをするのは困難です。判断に迷った場合に筆者がおすすめする方法は、まず（ _Value Semantics_ を持つ普通の） _値型_ で実装し、それでうまくいかない場合に _ミュータブルクラス_ （ _Reference Semantics_ ）を検討するというものです。 _Value Semantics_ を持たない型の利用を最小限に留めることで、「意図しない変更」や「整合性の破壊」などの発生機会を最小化し、問題のコントロールが容易になります。

そのときに、 _Value Semantics_ を持つ世界・持たない世界を分離しておくことが重要です。 _Value Semantics_ を持つ世界では共有にまつわる問題を気にする必要がありません。 _Value Semantics_ の世界をどれだけ広げられるかというのが、設計における一つのポイントになるでしょう。これは、 _参照型_ 中心の言語で _イミュータブル_ な世界を分離して、できるだけ広げる努力をするのと本質的に同じことです。

_値型_ には _イミュータブルクラス_ よりも状態の変更が容易であるという特性があります。そのため、 _Value Semantics_ をとるか、変更の容易さを取るかというトレードオフにおいて、 Swift では他の言語よりも _Value Semantics_ を採用するハードルが低いと言えるでしょう。

## まとめ

_値型_ は _ミュータブルクラス_ と _イミュータブルクラス_ のいいとこ取りをした存在で、 Swift はその _値型_ を中心とした言語です。 _値型_ 中心の言語はめずらしいですが、 Swift でそれが可能だった理由は、筆者は次の二つだと考えています。

- Swift の標準ライブラリはコレクションも _値型_
- Swift は _値型_ の使い勝手を向上させる言語仕様が豊富

前者は _Copy-on-Write_ による効率の良い _値型_ コレクションによって、後者は次のような言語仕様によって実現されています。

- 安全な `inout` 引数
- `mutating func`
- Computed Property を介した変更
- 高階関数と `inout` 引数の組み合わせ

Swift は進化の途上であり、 `for-in` ループと `inout` の組み合わせなど、今後さらに _値型_ の使い勝手が改善されていくものと思われます。

ただし、 _値型_ も万能ではありません。 _値型_ と _ミュータブルクラス_ を適切に使い分ける必要があります。どちらを採用すべきかわからないときは、まず _値型_ で始め、問題が生じたら _ミュータブルクラス_ を検討することをおすすめします。

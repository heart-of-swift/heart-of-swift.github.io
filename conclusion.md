---
layout: page
title: おわりに
---

本書では、 Swift の根幹である _Value Semantics_ と _Protocol-oriented Programming_ に焦点を当て、 Swift とはどのような言語なのかを説明してきました。そのことを、 ["はじめに"](/#introduction) では

> Swift の Heart （中心）となる概念を説明することを通して、 Swift の Heart （心）の形を描き出す

と表現しました。本書を通して描いてきた Heart の形は次のようになります。

<div class="summary"><img alt="Heart の形の本書のまとめ" src="img/swift-heart-summary.png" /></div>

### 原点

この図の根底にある、原点とも言えるものが _Value Semantics_ と _値型_ です。

---

- [第 1 章&nbsp;&nbsp;{{ site.data.book.chapters[0].name }}]({{ site.data.book.chapters[0].path }})

#### Value Semantics

<div class="summary-topic"><img alt="Value Semantics" src="img/swift-heart-summary.png" style="transform: scale(3, 3) translate(-32%, -40%);" /></div>

本書で最初に取り上げた、すべての大元にあったものが _Value Semantics_ でした。本書のすべての話は、 _Value Semantics_ が大切だということを前提に、 Swift ではどのようにして _Value Semantics_ 中心のコードが実現されているかという話だと言えます。

そのため、まず初めに _Value Semantics_ とは何かを、次に、 _Value Semantics_ を持たない型が引き起こす問題について説明しました。

---

- [{{ site.data.book.chapters[0].sections[0].name }}]({{ site.data.book.chapters[0].sections[0].path }})
- [{{ site.data.book.chapters[0].sections[1].name }}]({{ site.data.book.chapters[0].sections[1].path }})

#### 値型

<div class="summary-topic"><img alt="値型" src="img/swift-heart-summary.png" style="transform: scale(3, 3) translate(-28%, -34%);" /></div>

_ミュータブルクラス_ （ _ミュータブル_ な _参照型_ ）は _Value Semantics_ を持たないので、多くの言語では _イミュータブルクラス_ （ _イミュータブル_ な _参照型_ ）を使って _Value Semantics_ が実現されています。しかし、 Swift はその代わりに _値型_ を使って _Value Semantics_ を実現します。 _値型_ を使えば、 _イミュータブルクラス_ と比較して、状態変更を伴うコードを簡潔に記述することができます。

_値型_ は、 _ミュータブルクラス_ の持つ状態変更の簡潔さと、 _イミュータブルクラス_ の持つ _Value Semantics_ の利点の双方を兼ね備えたものと考えられます。そのような _値型_ を使って _Value Semantics_ を実現する、それが本書で説明してきた話の原点でした。

---

- [{{ site.data.book.chapters[0].sections[1].name }}]({{ site.data.book.chapters[0].sections[1].path }})

### 値型中心の世界を実現

<div class="summary-topic"><img alt="値型中心の世界を実現" src="img/swift-heart-summary.png" style="transform: scale(1.4, 1.4) translate(17%, -34%);" /></div>

Heart の図には、 _Value Semantics_ と _値型_ を原点として、二つの軸が描かれています。その一つが、左方向に伸びる **「値型中心の世界を実現」** する軸です。

_値型_ を使うことで _ミュータブルクラス_ と _イミュータブルクラス_ のいいとこ取りができました。しかし、コードのすべてを _値型_ で記述するには困難があります。「値型中心の世界を実現」する軸は、そのような困難を乗り越えるために Swift が取り入れた仕組みや提供している機能についての話でした。

---

- [第 1 章&nbsp;&nbsp;{{ site.data.book.chapters[0].name }}]({{ site.data.book.chapters[0].path }})

#### 値型のコレクション

<div class="summary-topic"><img alt="値型のコレクション" src="img/swift-heart-summary.png" style="transform: scale(3, 3) translate(-7%, -29%);" /></div>

そのような困難の一つがコレクションでした。

コレクションは、その性質から多くの言語で _参照型_ として実装されています。コレクションはコードのいたるところで使われるため、コレクションが _参照型_ であることは、 _値型_ だけでコードを書くのが困難であることを意味します。

Swift 標準ライブラリのコレクションは、 _Copy-on-Write_ を用いることで _値型_ として実装されています。そのため、 Swift ではコレクションも含めて値型だけでコードが書けることを説明しました。

---

- [{{ site.data.book.chapters[0].sections[2].name }}]({{ site.data.book.chapters[0].sections[2].path }})

#### 値型を使いやすくする機能

<div class="summary-topic"><img alt="値型を使いやすくする機能" src="img/swift-heart-summary.png" style="transform: scale(3, 3) translate(25%, -15%);" /></div>

もう一つの困難は、状態変更に伴うコードを記述する難しさでした。

単純なケースでは、 _値型_ は _ミュータブルクラス_ と同じように簡潔なコードで状態変更を扱うことができます。しかし、より複雑なケースでは一筋縄では行かないことがあります。

本書では次の四つを取り上げ、どのようなケースで難しさがあり、それを解決するために Swift がどのような機能を提供しているか、それらを使ってどのように問題を解決するのかを説明しました。

- 安全な `inout` 引数
- `mutating func`
- Computed Property を介した変更
- 高階関数と `inout` 引数の組み合わせ

---

- [{{ site.data.book.chapters[0].sections[2].name }}]({{ site.data.book.chapters[0].sections[2].path }})

### 値型前提の抽象化を実現

<div class="summary-topic"><img alt="制約としてのプロトコル" src="img/swift-heart-summary.png" style="transform: scale(1.2, 1.2) translate(-23%, 19%);" /></div>

Heart の図のもう一つの軸が、上方向に伸びる **「値型前提の抽象化を実現」** する軸です。

冗長なコードの重複を避けるために、抽象化は避けては通れないテーマです。 _値型_ 中心のコードでは、 _参照型_ 中心のコードとは異なる方法でコードを抽象化することを説明しました。

---

- [第 2 章&nbsp;&nbsp;{{ site.data.book.chapters[1].name }}]({{ site.data.book.chapters[1].path }})

#### 制約としてのプロトコル

<div class="summary-topic"><img alt="制約としてのプロトコル" src="img/swift-heart-summary.png" style="transform: scale(3, 3) translate(-26%, -17%);" /></div>

_オブジェクト指向プログラミング_ （ _Object-oriented Programming_ ）においては、 _継承_ を元にした _サブタイプポリモーフィズム_ （ _サブタイピング_ ）が抽象化における重要な役割を果たします。しかし、 _値型_ は _継承_ することができません。そこで、 Swift ではプロトコルを用いてコードを抽象化します。

このとき、 _値型_ 中心の Swift では _サブタイプポリモーフィズム_ よりも _パラメトリックポリモーフィズム_ が適していると説明しました。これをプロトコルに焦点を当てて言い換えると、 Swift ではプロトコルを型として用いるよりも、制約として用いるのが適していると言えます。 Swift の標準ライブリでも制約としてのプロトコルが広く使われています。

_Protocol-oriented Programming_ には明確な定義がないですが、本書ではそのようなプロトコルの使われ方に焦点を当てて _Protocol-oriented Programming_ を説明しました。

---

- [{{ site.data.book.chapters[1].sections[0].name }}]({{ site.data.book.chapters[1].sections[0].path }})
- [{{ site.data.book.chapters[1].sections[1].name }}]({{ site.data.book.chapters[1].sections[1].path }})


#### 戻り値の型の抽象化

<div class="summary-topic"><img alt="戻り値の型の抽象化" src="img/swift-heart-summary.png" style="transform: scale(2.8, 2.8) translate(-25%, 4%);" /></div>

_パラメトリックポリモーフィズム_ による抽象化を実現するために _ジェネリクス_ が用いられますが、 _ジェネリクス_ は戻り値の型をうまく抽象化することができません。そのため、 _ジェネリクス_ と対になる概念として、戻り値の型を抽象化する _リバースジェネリクス_ が考案されました。さらに、 _リバースジェネリクス_ を簡潔に記述するための構文として _Opaque Result Type_ が提案されました。

---

- [{{ site.data.book.chapters[1].sections[2].name }}]({{ site.data.book.chapters[1].sections[2].path }})

#### 引数の型の抽象化

<div class="summary-topic"><img alt="引数の型の抽象化" src="img/swift-heart-summary.png" style="transform: scale(2.8, 2.8) translate(-23%, 31%);" /></div>

_Opaque Result Type_ は戻り値の型についてのものでしたが、同じことを引数の型についても考えることができます。それが、 _ジェネリクス_ を簡潔に記述するための構文 _Opaque Argument Type_ でした。

---

- [{{ site.data.book.chapters[1].sections[2].name }}]({{ site.data.book.chapters[1].sections[2].path }})

#### Opaque Type と Existential Type

<div class="summary-topic"><img alt="Opaque Type と Existential Type" src="img/swift-heart-summary.png" style="transform: scale(2.4, 2.4) translate(13%, 16%);" /></div>

_Opaque Argument Type_ と _Opaque Result Type_ をまとめて _Opaque Type_ と呼びますが、 
_Opaque Type_ はプロトコルを制約として用います。これの対になるものに _Existential Type_ があり、 _Existential Type_ はプロトコルを型として用います。

_Opaque Type_ には `some` 、 _Existential Type_ には `any` というキーワードを修飾子として用いることで、「制約として」・「型として」のプロトコルの違いを明確にすることができました。

---

- [{{ site.data.book.chapters[1].sections[2].name }}]({{ site.data.book.chapters[1].sections[2].path }})

### まとめ

このように、本書は _Value Semantics_ と _値型_ を原点に、「値型中心の世界」と「値型前提の抽象化」の実現という二つの軸で、 Swift という言語の Heart を説明してきました。

Swift は決して難解な言語ではないですが、 _参照型_ 中心の言語が多い中では、 _値型_ を中心とした特徴的な言語だと言えます。言語を使いこなすには、その言語独自の考え方を理解することが欠かせません。

本書が Swift という言語を理解する手助けになったなら幸いです。

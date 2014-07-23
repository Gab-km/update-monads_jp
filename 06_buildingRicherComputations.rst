..
   Building richer computations
   ============================

より豊かな計算の構築
====================

.. One of the key things about F# computation expressions that I emphasized in `my previous blog post <http://tomasp.net/blog/2013/computation-zoo-padl>`_ and in the `PADL 2014 paper <http://tomasp.net/academic/papers/computation-zoo/>`_ is that computation expressions provide rich syntax that includes resource management (the ``use`` keyword), exception handling or loops (``for`` and ``while``) - in simple words, it mirrors the normal syntax of F#.

私が `以前のブログ投稿 <http://tomasp.net/blog/2013/computation-zoo-padl>`_ と `PADL 2014 の論文 <http://tomasp.net/academic/papers/computation-zoo/>`_ で強調していた F# のコンピュテーション式についてキーとなるものの1つは、コンピュテーション式がリソース管理(``use`` キーワード)や、例外処理、ループ(``for`` と ``while``)を含めた豊かな構文をもたらすということです－シンプルな言葉で言うと、これは普通の F# の構文そのままです。

.. So far, we have not used any of these for update monads. All these additional constructs have to be provided in the computation builder (so that the author can define them in the most suitable way). The great thing about update monads (for F#) is that we have just a single computation builder and so we can define a number of additional operations to enable richer syntax.

今までのところ、これらを何一つ Update モナドで使ったことがありません。これら全ての追加構造は(作者が最も都合の良い方法でそれらを定義できるように)コンピュテーションビルダーで提供されるべきものです。(F# 用の)Update モナドについてもっとも素晴らしいことは、私たちには単一のコンピュテーションビルダーがあり、そのためより豊かな構文を可能にするためにたくさんの演算を定義できることです。

.. The following snippet extends ``UpdateBuilder`` defined earlier with more operations. If you're not interested in the details, feel free to skip to the next section. The key idea is that this only has to be written once!

以下のスニペットは先に定義した ``UpdateBuilder`` をより多くの演算で拡張します。もし詳細に興味がなければ、次のセクションに飛んで構いません。キーとなるアイディアは、これは一度だけ書けばよいということです！

..
  /// Extends UpdateBuilder to support additional syntax
  type UpdateBuilder with
    /// Represents monadic computation that returns unit
    /// (e.g. we can now omit 'else' branch in 'if' computation)
    member inline x.Zero() = x.Return(())

    /// Delays a computation with (uncontrolled) side effects
    member inline x.Delay(f) = x.Bind(x.Zero(), f)

    /// Sequential composition of two computations where the
    /// first one has no result (returns a unit value)
    member inline x.Combine(c1, c2) = x.Bind(c1, fun () -> c2)

    /// Enable the 'return!' keyword to return another computation
    member inline x.ReturnFrom(m : UpdateMonad<'S, 'P, 'T>) = m

    /// Ensure that resource 'r' is disposed of at the end of the
    /// computation specified by the function 'f'
    member inline x.Using(r,f) = UM(fun s ->
      use rr = r in let (UM g) = f rr in g s)

    /// Support 'for' loop - runs body 'f' for each element in 'sq'
    member inline x.For(sq:seq<'V>, f:'V -> UpdateMonad<'S, 'P, unit>) =
      let rec loop (en:System.Collections.Generic.IEnumerator<_>) =
        if en.MoveNext() then x.Bind(f en.Current, fun _ -> loop en)
        else x.Zero()
      x.Using(sq.GetEnumerator(), loop)

    /// Supports 'while' loop - run body 'f' until condition 't' holds
    member inline x.While(t, f:unit -> UpdateMonad<'S, 'P, unit>) =
      let rec loop () =
        if t() then x.Bind(f(), loop)
        else x.Zero()
      loop()

.. code-block:: fsharp

  /// 追加の構文をサポートするために UpdateBuilder を拡張する
  type UpdateBuilder with
    /// ユニットを返すモナド的計算を表現する
    /// (例えば、'if' 計算で 'else' 節を省略できるようになります)
    member inline x.Zero() = x.Return(())

    /// (制御されていない)副作用のある計算を遅らせる
    member inline x.Delay(f) = x.Bind(x.Zero(), f)

    /// 1番目が結果を持たない(ユニットの値を返す)2つの計算を
    /// 順番に合成する
    member inline x.Combine(c1, c2) = x.Bind(c1, fun () -> c2)

    /// もう一つの計算を返すために'return!' キーワードを利用可能にする
    member inline x.ReturnFrom(m : UpdateMonad<'S, 'P, 'T>) = m

    /// 関数 'f' で指定された計算の最後でリソース 'r' が破棄される
    /// ことを保証する
    member inline x.Using(r,f) = UM(fun s ->
      use rr = r in let (UM g) = f rr in g s)

    /// 'for' ループをサポートする－'sq' の各要素に対し 'f' を実行する
    member inline x.For(sq:seq<'V>, f:'V -> UpdateMonad<'S, 'P, unit>) =
      let rec loop (en:System.Collections.Generic.IEnumerator<_>) =
        if en.MoveNext() then x.Bind(f en.Current, fun _ -> loop en)
        else x.Zero()
      x.Using(sq.GetEnumerator(), loop)

    /// 'while' ループをサポートする－条件 't' が成り立つまで 'f' を実行する
    member inline x.While(t, f:unit -> UpdateMonad<'S, 'P, unit>) =
      let rec loop () =
        if t() then x.Bind(f(), loop)
        else x.Zero()
      loop()

.. You can find more details about these operations in the `F# Computation Zoo paper <http://tomasp.net/academic/papers/computation-zoo/>`_ or in the `F# langauge specification <http://fsharp.org/about/index.html#specification>`_. In fact, the defitions mostly follow the samples from the F# specification. It is worth noting that all the members are marked as ``inline``, which allows us to use *static member constrains* and to write code that will work for any update monad (as defined by a pair of *update* and *state* types).

これらの演算についてもっと詳しい内容は `F# Computation Zoo の論文 <http://tomasp.net/academic/papers/computation-zoo/>`_ や `F# 言語仕様 <http://fsharp.org/about/index.html#specification>`_ で見つけることができます。実際、この定義はほぼ F# の仕様からのサンプルに従っています。コメントしておく価値があることとしては、全てのメンバーが ``inline`` としてマークされており、これにより *静的メンバー制約* が使えるようになり、(*update* と *state* 型のペアとして定義された)どんな Update モナドでも動作するコードが書けるようになります。

.. Let's look at a trivial example using the writer computation:

Writer コンピュテーションを使った自明な例を見てみましょう:

..
  /// Logs numbers from 1 to 10
  let logNumbers = update {
    for i in 1 .. 10 do
      do! write i }

.. code-block:: fsharp

  /// 1 から 10 までの数字をログに書き込む
  let logNumbers = update {
    for i in 1 .. 10 do
      do! write i }

.. As expected, when we run the computation using ``writeRun``, the result is a tuple containing a list with numbers from 1 to 10 and a unit value. The computation does not explicitly return and so the ``Zero`` member is automatically used.

予想通り、 ``writeRun`` を使った計算を実行すると、結果は 1 から 10 までの数字のリストとユニット値を持つタプルとなります。この計算は明示的にリターンしていないので、 ``Zero`` メンバーが自動的に使われます。

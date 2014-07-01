.. Implementing update monads in F#
   --------------------------------

F# で Update モナドを実装する
-----------------------------

.. To implement this idea in F#, we can use `static member constraints <http://msdn.microsoft.com/en-us/library/dd233203.aspx>`_. If you do not know about static member constrains in F#, you can think of it as a form of duck typing (or light-weight type classes). If a type defines certain members, then the code will compile and run fine, otherwise you'll get a compile-time error. We will require the user to define two types representing ``State`` and ``Update``, respectively. The ``Update`` type will need to define the three operations. An abstract definition (not valid F# code) would look like this:

F# でこのアイディアを実装するために、私たちは `静的メンバー制約 <http://msdn.microsoft.com/en-us/library/dd233203.aspx>`_ を使うことができます。F# の静的メンバー制約についてご存じない場合は、ダックタイピングのようなもの(または軽量な型クラス)だと思っておくといいでしょう。もしある型がとあるメンバーを定義していれば、コードがちゃんとコンパイルと実行ができ、そうでなければコンパイル時エラーが発生します。私たちはユーザーに ``State`` と ``Update`` を表す2つの型をそれぞれ定義するように求めます。 ``Update`` 型は3つの演算を定義する必要があります。抽象的な定義(ちゃんとした F# のコードではありません)はこのようになります:

.. code-block:: fsharp

  type State
  type Update =
    static Unit    : Update
    static Combine : Update * Update -> Update
    static Apply   : State * Update -> State

.. Invocation of members via static member constraints is not entirely easy (the feature is used mainly by library implementors for things like generic numerical code). But the idea is that we can define inline functions (``unit``, ``++`` and ``apply``) that invoke the corresponding operation on the type (specified either explicitly or via type inference).

静的メンバー制約によるメンバー呼び出しは、すべてが簡単というわけではありません(その機能は主にジェネリックな数値型のコードのような用途のライブラリを実装する方々に利用されます)。しかし、(明示的に、または型インターフェイスを通じて特定された)型に対応した演算を呼び出すインライン関数(``unit`` 、 ``++`` および ``apply``)を定義することができるのがこのアイディアなのです。

.. If you're not familiar with F#, you can freely skip over this definition, just remember that we now have functions ``unit`` and ``apply`` and an operator ``++``:

もし F# に馴染みがないのであれば、 ``unit`` 関数、 ``apply`` 関数、 ``++`` 演算子があるよということだけ覚えておいて、この定義は気軽に飛ばして構いません:

.. /// Returns the value of 'Unit' property on the ^S type
   let inline unit< ^S when ^S :
       (static member Unit : ^S)> () : ^S =
     (^S : (static member Unit : ^S) ())

   /// Invokes Combine operation on a pair of ^S values
   let inline (++)< ^S when ^S :
       (static member Combine : ^S * ^S -> ^S )> a b : ^S =
     (^S : (static member Combine : ^S * ^S -> ^S) (a, b))

   /// Invokes Apply operation on state and update ^S * ^U
   let inline apply< ^S, ^U when ^U :
       (static member Apply : ^S * ^U -> ^S )> s a : ^S =
     (^U : (static member Apply : ^S * ^U -> ^S) (s, a))

.. code-block:: fsharp

  /// ^S 型の 'Unit' プロパティの値を返す
  let inline unit< ^S when ^S :
      (static member Unit : ^S)> () : ^S =
    (^S : (static member Unit : ^S) ())

  /// ^S 型の値のペアに対する Combine 演算を呼び出す
  let inline (++)< ^S when ^S :
      (static member Combine : ^S * ^S -> ^S )> a b : ^S =
    (^S : (static member Combine : ^S * ^S -> ^S) (a, b))

  /// 状態と更新のペア ^S * ^U に対する Apply 演算を呼び出す
  let inline apply< ^S, ^U when ^U :
      (static member Apply : ^S * ^U -> ^S )> s a : ^S =
    (^U : (static member Apply : ^S * ^U -> ^S) (s, a))

.. The last thing that we need to do before we can start playing with some update monads is to implement the monadic operators. In F#, this is done by defining a *computation builder* - a type that has ``Bind`` and ``Return`` operations (as well as some others that we'll see later). The compiler then automatically translates a block ``update { .. }`` using operations ``update.Return`` and ``update.Bind``.

Update モナドで遊び始める前に私たちがするべきことの最後は、モナド的な演算を実装することです。F# では、 *コンピュテーションビルダー* ― ``Bind`` と ``Return`` という演算(他のいくつかも後ほど見てみましょう)を持った型 ― を定義することで達成されます。そしてコンパイラは自動的に ``update.Return`` 演算と ``update.Bind`` 演算を使っている ``update { .. }`` ブロックを翻訳します。

.. The computation builder is a normal class with members. Because we are using static member constraints and inline functions, we need to mark the members as ``inline`` too:

コンピュテーションビルダーはメンバーを持った普通のクラスです。私たちは静的メンバー制約とインライン関数を使っているため、私たちはメンバーにもまた ``inline`` を付ける必要があります。

.. type UpdateBuilder() =
     /// Returns the specified value, together
     /// with empty update obtained using 'unit'
     member inline x.Return(v) : UpdateMonad<'S, 'U, 'T> =
       UM (fun s -> (unit(),v))

     /// Compose two update monad computations
     member inline x.Bind(UM u1, f:'T -> UpdateMonad<'S, 'U, 'R>) =
       UM (fun s ->
         // Run the first computation to get first update
         // 'u1', then run 'f' to get second computation
         let (u1, x) = u1 s
         let (UM u2) = f x
         // Apply 'u1' to original state & run second computation
         // then return result with combined state updates
         let (u2, y) = u2 (apply s u1)
         (u1 ++ u2, y))

   /// Instance of the computation builder
   /// that defines the update { .. } block
   let update = UpdateBuilder()

.. code-block:: fsharp

  type UpdateBuilder() =
    /// 特定の値と一緒に 'unit' を使って得た空の更新を返す
    member inline x.Return(v) : UpdateMonad<'S, 'U, 'T> =
      UM (fun s -> (unit(),v))

    /// 2つの Update モナドの計算を合成する
    member inline x.Bind(UM u1, f:'T -> UpdateMonad<'S, 'U, 'R>) =
      UM (fun s ->
        // 最初の更新 'u1' を取得するために最初の計算を実行し、
        // 2つ目の計算を取得するために 'f' を実行する
        let (u1, x) = u1 s
        let (UM u2) = f x
        // 元の状態に 'u1' を適用し、2つ目の計算を実行して
        // 状態と更新を合わせた結果を返す
        let (u2, y) = u2 (apply s u1)
        (u1 ++ u2, y)

  /// update { .. } ブロックを定義する
  /// コンピュテーションビルダーのインスタンス
  let update = UpdateBuilder()

.. The implementation of the ``Return`` operation is quite simple - we return the specified value and call ``unit()`` to get the unit of the monoid of updates - as a result, we get a computation that returns the value without performing any update on the state.

``Return`` 演算の実装は本当にシンプルです ― 特定の値を返し、更新のモノイドの単位元を取得するために unit() を呼び出します ― その結果、私たちは状態を更新することなく値を返す計算を取得します。

.. The ``Bind`` member is more interesting - it runs the first computation which returns a value ``x`` and an update ``u1``. The second computation needs to be run in an updated state and so we use ``apply s u1`` to calculate a new state that reflects the update. After running the second computation, we get the final resulting value ``y`` and a second update ``u2``. The result of the computation combines the two updates using ``u1 ++ u2``.

``Bind`` メンバーはもっと興味深いです ― 値 ``x`` と更新 ``u1`` を返す最初の計算を実行します。2番目の計算は更新された状態に対して実行される必要があるので、更新を反映した新しい状態を計算するために ``aply s u1`` を使います。2番目の計算を実行したあと、値 ``y`` と2つ目の更新 ``u2`` を最終結果として取得します。計算結果は ``u1 ++ u2`` を用いて2つの更新を結合します。

.. How does this actually work? Let's start by looking at reader and writer monads (which are special cases of the update monad.

これは実際どのように機能するんでしょうか？(Update モナドの特別なケースである)Reader と Writer モナドを確認することで始めてみましょう。

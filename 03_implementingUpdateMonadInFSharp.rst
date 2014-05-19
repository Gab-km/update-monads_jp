F# で Update モナドを実装する
-----------------------------

F# でこのアイディアを実装するために、私たちは `static member constraints <http://msdn.microsoft.com/en-us/library/dd233203.aspx>`_ を使うことができます。F# の static member constraints についてご存じない場合は、ダックタイピングのようなもの(または軽量な型クラス)だと思っておくといいでしょう。もしある型がとあるメンバーを定義していれば、コードがちゃんとコンパイルと実行ができ、そうでなければコンパイル時エラーが発生します。私たちはユーザーに ``State`` と ``Update`` を表す2つの型をそれぞれ定義するように求めます。 ``Update`` 型は3つの演算を定義する必要があります。抽象的な定義(ちゃんとした F# のコードではありません)はこのようになります:

.. code-block:: fsharp

  type State
  type Update =
    static Unit    : Update
    static Combine : Update * Update -> Update
    static Apply   : State * Update -> State

static member constraints によるメンバー呼び出しは、すべてが簡単というわけではありません(その機能は主にジェネリックな数値型のコードのような用途のライブラリを実装する方々に利用されます)。しかし、(明示的に、または型インターフェイスを通じて特定された)型に対応した演算を呼び出すインライン関数(``unit`` 、 ``++`` および ``apply``)を定義することができるのがこのアイディアなのです。

もし F# に馴染みがないのであれば、 ``unit`` 関数、 ``apply`` 関数、 ``++`` 演算子があるよということだけ覚えておいて、この定義は気軽に飛ばして構いません:

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

Update モナドで遊び始める前に私たちがするべきことの最後は、モナド的な演算を実装することです。F# では、 *コンピュテーションビルダー* ― ``Bind`` と ``Return`` という演算(他のいくつかも後ほど見てみましょう)を持った型 ― を定義することで達成されます。そしてコンパイラは自動的に ``update.Return`` 演算と ``update.Bind`` 演算を使っている ``update { .. }`` ブロックを翻訳します。

コンピュテーションビルダーはメンバーを持った普通のクラスです。私たちは static member constraints とインライン関数を使っているため、私たちはメンバーにもまた ``inline`` を付ける必要があります。

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

``Return`` 演算の実装は本当にシンプルです ― 特定の値を返し、更新のモノイドの単位元を取得するために unit() を呼び出します ― その結果、私たちは状態を更新することなく値を返す計算を取得します。

``Bind`` メンバーはもっと興味深いです ― 値 ``x`` と更新 ``u1`` を返す最初の計算を実行します。2番目の計算は更新された状態に対して実行される必要があるので、更新を反映した新しい状態を計算するために ``aply s u1`` を使います。2番目の計算を実行したあと、値 ``y`` と2つ目の更新 ``u2`` を最終結果として取得します。計算結果は ``u1 ++ u2`` を用いて2つの更新を結合します。

これは実際どのように機能するんでしょうか？(Update モナドの特別なケースである)Reader と Writer モナドを確認することで始めてみましょう。

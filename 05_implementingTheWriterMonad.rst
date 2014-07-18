..
   -----------------------------
   Implementing the writer monad
   -----------------------------

-------------------
Writer モナドの実装
-------------------

.. Similarly to the reader monad, the writer monad is just a simple special case of the update monad. This time, the *state* is trivial and all the interesting things are happening in the updates. The type of the usual writer monad is ``'TState * 'T`` and so if we want to make this a special case of update monad, we can define the type as ``unit -> 'TState * 'T``.

Reader モナドと同様に、Writer モナドは Update モナドのシンプルで特別なケースに過ぎません。今回は、 *state* が自明であること、および全ての興味深いことが update の中で起こっています。普通の Writer モナドの型は ``'TState * 'T`` ですので、これを Update モナドの特別なケースにしたいのであれば、私たちはこの型を ``unit -> 'TState * 'T`` として定義できます。

..
   Writer state and update
   -----------------------

Writer の状態と更新
-------------------

.. The state needs to be a monoid (with unit and composition) so that we can compose the states of multiple sub-computations. The following example uses a list as a concrete example. We define a (writer) monad that keeps a list of ``'TLog`` values and returns that as the result (more generally, we could use an arbitrary monoid instead of a list):

複数の下位の計算の状態を合成することが出来るように、状態は(単位元と合成を持った)モノイドである必要があります。以下の例は具体的な例としてリストを使っています。私たちは ``'TLog`` の値を保持し、結果としてそのリストを返す(Writer)モナドを定義します(より一般的には、私たちはリストの代わりに任意のモノイドを定義することが出来ます):

.. /// Writer monad has no readable state
   type WriterState = NoState

   /// Updates of writer monad form a list
   type WriterUpdate<'TLog> =
     | Log of list<'TLog>
     /// Returns the empty log (monoid unit)
     static member Unit = Log []
     /// Combines two logs (operation of the monoid)
     static member Combine(Log a, Log b) = Log(List.append a b)
     /// Applying updates to state does not affect the state
     static member Apply(NoState, _) = NoState

.. code-block:: fsharp

  /// Writer モナドは読み込み可能な状態を持たない
  type WriterState = NoState

  /// Writer モナドの更新がリストを形成する
  type WriterUpdate<'TLog> =
    | Log of list<'TLog>
    /// 空のログを返す(モノイドの単位元)
    static member Unit = Log []
    /// 2つのログを結合する(モノイドの演算)
    static member Combine(Log a, Log b) = Log(List.append a b)
    /// 状態に更新を適用することはその状態に影響を与えない
    static member Apply(NoState, _) = NoState

.. The writer monad appears (in some informal sense) dual to the earlier reader monad. The state (that can be read) is always empty and is represented by the ``NoState`` value, while all the interesting aspects are captured by the ``WriterUpdate`` type - which is a list of values produced by the computation. The updates of a writer monad have to form a monoid - here, we use a list that concatenates all logged values. You could easily change the definition to implement other monoids (e.g. to keep the last produced value).

Writer モナドは先の Reader モナドに対して(いくぶん形式的でない点で)2つの面を見せます。(読み取り可能な)状態は常に空で ``NoState`` という値で表現され、全ての興味深い側面が ``WriterUpdate`` 型ーこの型は計算によって生成された値のリストです。Writer モナドの更新はモノイドを成す必要がありますーに捉えられますが、私たちはすべてのログの値を連結したリストを使います。他のモノイド(例えば、最後に生成した値を保持する)を実装する定義に簡単に変更できます。

..
   Writer monad primitives
   -----------------------

Writer モナドのプリミティブ
---------------------------

.. Similarly to the previous example, we now need two primitives - one to add new element to the log (``write`` of the writer monad) and one to run a computation and extract the result and the log:

前の例と同様に、2つのプリミティブー1つはログに新しい要素を追加するもの(Writer モナドの ``write``)、もう1つは計算を実行し結果とログを抽出するものーが必要です:

..
  /// Writes the specified value to the log
  let write v = UM (fun s -> (Log [v], ()))
  /// Runs a "writer monad computation" and returns
  /// the log, together with the final result
  let writeRun (UM f) = let (Log l, v) = f NoState in l, v

.. code-block:: fsharp

  /// ログに特定の値を書き込む
  let write v = UM (fun s -> (Log [v], ()))
  /// "Writer モナドの計算"を実行し、最終結果と
  /// 一緒にログを返す
  let writeRun (UM f) = let (Log l, v) = f NoState in l, v

.. The ``write`` function creates a singleton list containing the specified value ``Log [v]`` as the update and returns the unit value as the result of the computation. When combined with other computations, the updates are concatenated and so this will become a part of the list ``l`` in the result ``(Log l, v)`` that is made accessible in the ``writerRun`` function.

``writer`` 関数は更新として特定の値 ``Log [v]`` を含む単独のリストを作り、計算の結果として単位元を返します。他の計算と合成させた時、更新は連結され、 ``(Log l, v)`` の中にあるリスト ``l`` の一部になるため、 ``writerRun`` 関数の中でアクセス可能になります

..
   Sample writer computations
   --------------------------

Writer の計算サンプル
---------------------

.. Let's have a look at a sample computation using the new definitions - the remarkable thing (from the practical F# programming perspective) is that we wrap the computation in the ``update { .. }`` block just like in the previous example. But this time, we'll use the ``write`` primitive to write 20 and then 10 to the log and the F# compiler correctly infers that we are using ``WriterState`` and ``WriterUpdate`` types:

新しい定義を使った計算のサンプルを見てみましょうー(実践的な F# プログラミングの視点から)特筆すべきことは、ちょうど前の例のように ``update { .. }`` ブロックの中に計算をラップしたことです。しかし今回、20とそれから10をログに書き込むために ``write`` プリミティブを使い、F# コンパイラは私たちが ``WriterState`` と ``WriterUpdate`` 型を使っていることを正しく推論します:

..
  /// Writes '20' to the log and returns "world"
  let demo3 = update {
    do! write 20
    return "world" }
  /// Calls 'demo3' and then writes 10 to the log
  let demo4 = update {
    let! w = demo3
    do! write 10
    return "Hello " + w }

  /// Returns "Hello world" with 20 and 10 in the log
  demo4 |> writeRun

.. code-block:: fsharp

  /// '20' をログに書き"world"を返す
  let demo3 = update {
    do! write 20
    return "world" }
  /// 'demo3'を呼び出し、10をログに書き込む
  let demo4 = update {
    let! w = demo3
    do! write 10
    return "Hello " + w }

  /// ログにある20と10と一緒に"Hello world"を返す
  demo4 |> writeRun

.. If you run the code, the ``demo3`` computation first writes 20 to the log, which is then combined (using the ``++`` operator that invokes ``WriterUpdate.Combine``) with the value 10 written in ``demo4``.

このコードを実行すると、 ``demo3`` の計算が最初に 20 をログに書き込み、 ``demo4`` で書き込まれた値 10 が(``WriterUpdate.Combine`` を呼び出す ``++`` 演算子を使って)合成されます。

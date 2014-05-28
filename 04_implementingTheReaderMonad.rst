..
   -----------------------------
   Implementing the reader monad
   -----------------------------

====================
 Readerモナドの実装
====================

.. The reader monad keeps some state, but it does not give us any way of modifying it. In terms of update monads, this means that there is some state, but the monoid of updates is trivial - in principle, we can just use ``unit`` as the type of updates. You can see that when looking at the type too - the type of reader monad is ``'TState -> 'T``. To get a type with a structure matching to update monads, we can use an equivalent type ``'TState -> unit * 'T``.

リーダーもなどはいくつかの状態を保持しますが、それらを更新する術をもちません。Updateモナドの文脈で言うならば、いくつか状態は存在するものの、更新を行うモノイドは原則的には取るに足らないもので、単に ``Unit`` を更新の型として用いればいいということになります。このことは、型を見ればわかります -Readerモナドの方は ``'TState -> T`` となっています-。Updateモナドに対応する構造にするため、同様の意味を持つ型として ``'TState -> unit * 'T`` を用いることができます。


..
   Reader state and update
   -----------------------



読み取り状態と更新
==================

.. note::
   ここどう訳するべき？？Reader状態とUpdate、とかおかしいけど日本語だけってのもわけわからん。

.. In practice, we still need to define a type for updates, so that we can provide the required static members. We use a single-case discriminated union with just one value ``NoUpdate``:

実際の所、必要な静的メンバを提供できるように、まだ更新のための型を定義する必要があります。

..
   .. code-block:: fsharp

     /// The state of the reader is 'int'
     type ReaderState = int
     /// Trivial monoid of updates
     type ReaderUpdate =
       | NoUpdate
       static member Unit = NoUpdate
       static member Combine(NoUpdate, NoUpdate) = NoUpdate
       static member Apply(s, NoUpdate) = s

.. code-block:: fsharp

  /// Readerの状態は 'int'
  type ReaderState = int
  /// 更新用の単純なMonoid
  type ReaderUpdate =
    | NoUpdate
    static member Unit = NoUpdate
    static member Combine(NoUpdate, NoUpdate) = NoUpdate
    static member Apply(s, NoUpdate) = s

.. None of the operations on the ``ReaderUpdate`` type does anything interesting. Both unit and combine simply returns the only possible value and the apply operation returns the state without a change.

``ReaderUpdate``に関わる操作で興味深いものはありません。 ``unit`` も ``combine`` も単純に取りうるただひとつの値を返すだけですし、 ``apply`` 操作は状態を変更なく返すだけです。

..
   Reader monad primitives
   -----------------------

Readerモナド プリミティブ
=========================

.. Next, we need a primitive that allows us to read the state and a run operation that executes a computation implemented using the reader monad (given the value of the read-only state). The operations look as follows:

.. note::
   * Computation = 式 でOK？？
   * primitive = 訳せない・・・。とりあえずPrimitiveで。

次に、（読み取り専用な値を受け取る）Readerモナドを用いて実装された式を実行するための、プリミティブな操作が必要となります。それは以下のようになります。

..
   .. code-block:: fsharp

     /// Read the current state (int) and return it as 'int'
     let read = UM (fun (s:ReaderState) ->
       (NoUpdate, s))
     /// Run computation and return the result
     let readRun (s:ReaderState) (UM f) = f s |> snd

.. code-block:: fsharp

  /// 現在の状態（int）を読み取り、それを'int'として返す
  let read = UM (fun (s:ReaderState) ->
    (NoUpdate, s))
  /// 計算を実行し結果を返す
  let readRun (s:ReaderState) (UM f) = f s |> snd


.. When you look at the type of computations (hover the mouse pointer over the ``read`` identifier), you can see a parameterized update monad type. The ``read`` primitive has a type ``UpdateMonad<ReaderState, ReaderUpdate, ReaderState>``. This means that we have an update monad that uses ``ReaderState`` and ``ReaderUpdate`` as the *act* (specifying the computation details) and, when executed, produces a value of ``ReaderState``.

式の方を確認すると（マウスを ``read`` の上にホバーさせてみてください）、パラメータ化されたUpdateモナド型になっているのがわかります。 ``read`` プリミティブは ``UpdateMonad<ReaderState, ReaderUpdate, ReaderState>`` 型です。これは ``ReaderState`` と ``ReaderUpdate`` を （計算の詳細を指定する） *act* として用い、実行時には ``ReaderState`` を生成するようなupdateモナドを定義したことを意味します。

Sample reader computations
--------------------------

Now we can use the ``update { .. }`` block together with the ``read`` primitive to write computations that can read an immutable state. The following basic example reads the state and adds one (in ``demo1``), and then adds 1 again in ``demo2``:

.. code-block:: fsharp

  /// Returns state + 1
  let demo1 = update {
    let! v = read
    return v + 1 }
  /// Returns the result of demo1 + 1
  let demo2 = update {
    let! v = demo1
    return v + 1 }

  // Run it with state 40
  demo2 |> readRun 40

If you run the code, you'll see that the result is 42. The interesting thing about this approach is that we only had to define two types. The ``update { .. }`` computation works for all update monads and so we get the computation builder "for free". However, thanks to the parameterization, the computation really represents an immutable state - there is no way to mutate it.

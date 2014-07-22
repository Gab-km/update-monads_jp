..
   Implementing the state monad
   ============================

Stateモナドの実装
=================

..
   Interestingly, standard state monad is *not* a special case of update monads. However, we can define a computation that implements the same functionality - a computation with state that we can read and write.

興味深いことに、通常のStateモナドはUpdateモナドの特別系 *ではありません* 。しかし、同様の機能 - 読み書き可能な状態を伴った計算 - を実装した計算を定義することはできます。

..
   States and updates
   ------------------

状態と更新
----------

..
   In this final example, both the type representing *state* and the type representing *update* will have a useful role. We make both of the types generic over the value they carry. State is simply a wrapper containing the value (current state). Update can be of two kinds - we have an empty update (do nothing) and an update to set the state:


この最後の例では、 *状態* を表す型 ``State`` と *更新* を表現する型 ``Update``  両方が、意味のある役割を持つことになります。それらの型を自身が保持している値に対してジェネリックにします。 ``State`` は単に含んでいる値（現在の状態）のラッパーです。 ``Update`` は2種類 - 空の更新（何もしない）と状態を設定する更新  - を取り得ます。

..
   .. code-block:: fsharp

     /// Wraps a state of type 'T
     type StateState<'T> = State of 'T

     /// Represents updates on state of type 'T
     type StateUpdate<'T> =
       | Set of 'T | SetNop
       /// Empty update - do not change the state
       static member Unit = SetNop
       /// Combine updates - return the latest (rightmost) 'Set' update
       static member Combine(a, b) =
	 match a, b with
	 | SetNop, v | v, SetNop -> v
	 | Set a, Set b -> Set b
       /// Apply update to a state - the 'Set' update changes the state
       static member Apply(s, p) =
	 match p with SetNop -> s | Set s -> State s

.. code-block:: fsharp

  /// 型'Tの状態をラップします
  type StateState<'T> = State of 'T

  /// 型'Tの状態に対する更新を表します
  type StateUpdate<'T> =
    | Set of 'T | SetNop
    /// 空の更新 - 何も状態を変更しません
    static member Unit = SetNop
    /// 更新の結合 - 最新の（最も右にある） 'Set'更新を返します
    static member Combine(a, b) =
      match a, b with
      | SetNop, v | v, SetNop -> v
      | Set a, Set b -> Set b
    /// 状態に対して更新を適用します - 'Set'更新が状態を変更します
    static member Apply(s, p) =
      match p with SetNop -> s | Set s -> State s


..
   This definition is a bit more interesting than the previous two, because there is some interaction between the *states* and *updates*. In perticular, when the update is ``Set v`` (we want to replace the current state with a new one), the ``Apply`` member returns a new state instead of the original. For the ``Unit`` member, we need an update ``SetNop`` which simply means that we want to keep the original state (and so ``Apply`` just returns the original value in this case).

この定義は前の2つに比べるとより興味深いものとなっています、なぜなら *状態* と *更新* の間にいくつかの相互作用があるからです。特に、更新が ``Set v`` （現在の状態を新しいもので置き換えようとします） の時には、 ``Apply`` メンバは元とは異なる状態を返します。 ``Unit`` メンバについては、 単に元の状態を保持しておくという ``SetNop`` の更新が必要になります。（ですのでこの場合は ``Apply`` はただ単に元の値を返します。）

..
   Another notable thing is the ``Combine`` operation - it takes two updates (which may be either empty updates or set updates) and produces a single one. If you read a composition ``a1 ++ a2 ++ .. ++ an`` as a sequence of state updates (either ``Set`` or ``SetNop``), then the ``Combine`` operation returns the last ``Set`` update in the sequence (or ``SetNop`` if ther are no ``Set`` updates). In other words, it builds an update that sets the last state that was set during the whole sequence.

もう一つの特筆すべき点は、 ``Combine`` 操作 - 2つの更新（両方とも空の更新かも知れませんし通常の更新かもしれません）を受け取り一つの更新を返します - です。合成 ``a1 ++ a2 ++ .. ++ an`` を状態更新のシーケンス（ ``Set`` でも ``SetNope`` でも構いません ）として読み取った場合、 ``Combine`` 操作はシーケンス中の最後の ``Set`` 更新（一つも ``Set`` 更新がない場合は ``SetNop`` ）を返します。言い換えると、シーケンス全体の中での最後の状態を設定するような更新を構築するのです。

..
   State monad primitives
   ----------------------

State モナドプリミティブ
----------------------------

..
   Now that we have the type definitions, it is quite easy to add the usual primitives:

さあ、型の定義はできましたので、通常のプリミティブを追加するのはかなり簡単になります。

..
   .. code-block:: fsharp

     /// Set the state to the specified value
     let set s = UM (fun _ -> (Set s,()))
     /// Get the current state
     let get = UM (fun (State s) -> (SetNop, s))
     /// Run a computation using a specified initial state
     let setRun s (UM f) = f (State s) |> snd

.. code-block:: fsharp

  /// 指定された値に状態を設定する
  let set s = UM (fun _ -> (Set s,()))
  /// 現在の状態を取得する
  let get = UM (fun (State s) -> (SetNop, s))
  /// 初期状態を設定して計算を実行する
  let setRun s (UM f) = f (State s) |> snd


..
   The ``set`` operation is a bit different than the usual one for state monad. It ignores the state and it builds an *update* that tells the computation to set the new state. The ``get`` operation reads the state and returns it - but as it does not intend to change it, it returns ``SetNop`` as the update.

``set`` 操作は一般的なStateモナドのそれとは少し違います。状態を無視し、新しい状態を設定するための計算を表す *更新* を構築します。 ``get`` オペレーションは状態を読み取ってそれを返します - ただし何も変更しない場合には、更新として ``SetNop`` を返します。

..
   Sample stateful computation
   ---------------------------

状態を持った計算のサンプル
--------------------------

..
   If you made it this far in the article, you can already expect how the example will look! We'll again use the ``update { .. }`` computation. This time, we define a computation ``demo5`` that increments the state and call it from a loop in ``demo6``:

ここまで読んできた方なら、次の例がどんな風になるのか予想できるでしょう！もう一度 ``update { .. }`` コンピュテーション式を使います。今回は、 ``demo5`` という、状態をインクリメントし ``demo6`` のループの中から呼ぶコンピュテーション式を定義します。

..
   .. code-block:: fsharp

     /// Increments the state by one
     let demo5 = update {
       let! v = get
       do! set (v + 1) }
     /// Call 'demo5' repeatedly in a loop
     /// and then return the final state
     let demo6 = update {
       for i in 1 .. 10 do
	 do! demo5
       return! get }
     // Run the sample with initial state 0
     demo6 |> setRun 0

.. code-block:: fsharp

  /// 状態を1ずつインクリメントする
  let demo5 = update {
    let! v = get
    do! set (v + 1) }
  /// demo5をループの中で反復して呼び
  /// 最後の状態を返す
  let demo6 = update {
    for i in 1 .. 10 do
      do! demo5
    return! get }
  // サンプルを初期状態 0 で実行させる
  demo6 |> setRun 0

..
   Running the code yields 10 as expected - we start with zero and then increment the state ten times. Since we extended the definition of the ``UpdateBuilder`` (in the previous section), we now got a few nice things for free - we can use the ``for`` loop and write computations (like ``demo5``) without explicit ``return`` if they just need to modify the state.

コードを実行すると、予想通り10という結果を得ます - ゼロから始まり、その状態を10回インクリメントしたわけです。 我々は ``UpdateBuilder`` の定義を拡張しましたので（前の章で行いました）,
いくつかの素敵な特典にタダ乗りで来ています - ``for`` ループを使うことや、（ ``demo5`` のように）ただ状態を変更したいだけの際には、明示的に ``return`` を書かなくても計算を記述することができるのです。

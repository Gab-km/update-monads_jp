.. Introducing update monads
   -------------------------

Update モナドの紹介
-------------------

.. Before looking at the definition of update monads, it is useful to review the three monads that we want to unify. The `update monads paper <http://cs.ioc.ee/~tarmo/papers/types13.pdf>`_ has more details and you can also find other tutorials that implement these in F# - here, we'll only look at the type definitions:

Update モナドの定義を見る前に、私たちがひとまとめにしたい3つのモナドをレビューすると都合がいいですね。 `Update モナドの論文 <http://cs.ioc.ee/~tarmo/papers/types13.pdf>`_ はより詳細で、これらを F# で実装した別のチュートリアルも見つけることができます ― では、型の定義だけ見てみることにしましょう:

.. /// Given a readonly state, produces a value
   type Reader<'TState, 'T> = 'TState -> 'T
   /// Produces a value together with additional state
   type Writer<'TState, 'T> = 'TState * 'T
   /// Given state, produces new state & a value
   type State<'TState, 'T>  = 'TState -> 'TState * 'T

.. code-block:: fsharp

  /// 与えられた読み取り専用の状態から、値を生成する
  type Reader<'TState, 'T> = 'TState -> 'T
  /// 追加の状態と一緒に値を生成する
  type Writer<'TState, 'T> = 'TState * 'T
  /// 与えられた状態に対し、新しい状態と値を生成する
  type State<'TState, 'T>  = 'TState -> 'TState * 'T

.. If you look at the definitions, it looks like *reader* and *writer* are both a versions of the *state* with some aspect missing - reader does not produce a new state and writer does not take previous state.

この定義を見たら、 *Reader* と *Writer* はどちらも *State* に何らかの特徴が欠けたバージョン ― Reader は新しい状態を生成せず、Writer は前の状態を受け取らない ― のように見えるでしょう。

.. How can we define a parameterized computation type that allows leaving one or the other out? The idea of *update monads* is quite simple. The trick is that we'll take two different types - one representing the *state* we can read and another representing the *updates* that specify how to change the state:

どれかを対象外にできるようなパラメータ化されたコンピュテーション型をどのように定義したらよいでしょうか？ *Update モナド* のアイディアはまさにシンプルです。トリックは2つの異なる型を受け取ることです ― 1つは読み取りできる *状態* を表し、もう1つはどのように状態を変更するか特定する *更新* を表します:

.. /// Represents an update monad - given a state, produce
   /// value and an update that can be applied to the state
   type UpdateMonad<'TState, 'TUpdate, 'T> =
     UM of ('TState -> 'TUpdate * 'T)

.. code-block:: fsharp

  /// Update モナドを表す - 与えられた状態に対し、
  /// 値とその状態に適用される更新を生成する
  type UpdateMonad<'TState, 'TUpdate, 'T> =
    UM of ('TState -> 'TUpdate * 'T)

.. To make the F# implementation a bit nicer, this is not defined as a type alias, but as a new type labeled with ``UM`` (this makes sure that the infered types will always use the name ``UpdateMonad`` for the type, rather than its definition).

F# の実装をもうちょっといい感じにするために、これは型エイリアスとしてではなく、``UM`` とラベルされた新しい型(これにより、推論された型がその定義名ではなく、常に ``UpdateMonad`` を型名として使うことになります)として定義されています。

.. To make this work, we also need some operations on the types representing states and updates. In particular, we need:

これがちゃんと動くようにするため、私たちは状態と更新を表す型に対していくつかの演算も必要となります。特に、私たちが必要なのは以下のものです:

.. * **Unit update** which represents that no update is applied.
   * **Composition** on updates that allows combining multiple updates on the state into a single update.
   * **Application** that takes a state and an update and applies the update on the state, producing new state as the result.

* 何の更新も適用されないことを表す **更新の単位元**
* ある状態における複数の更新を結合し、１つの更新にする **更新の結合**
* 状態と更新を受け取って状態に更新を適用し、結果として新しい状態を生成する **適用**

.. In more formal terms, the type of updates needs to be a monoid (with unit and composition). In the paper, the two types (sets) together with the operations are called *act* and are defined as (**S**,(**P**,∘,⊕),↓) where ∘ is the unit, ⊕ is composition and ↓ is the application.

より形式的な用語だと、更新の型は(単位元と結合を持つ)モノイドである必要があります。この論文では、2つの型(集合)と演算を一緒にしたものを *act* と呼び、 ∘ を単位元、 ⊕ を合成、そして ↓ を適用として (**S**,(**P**,∘,⊕),↓) として定義されています。

..    Note on naming

      In the last case, I'm using different naming than the original paper. In the paper, applying update **u** to a state **s** is written as **s↓u**. You can see the "**↓u**" part as an action that transforms state and so the authors use the name action. I'm going to use *apply* instead because I refer more to the operation **↓** than to the entire (partially-applied action).

  ネーミングについての注釈

  最後のケースで、私は元の論文とは異なるネーミングを使っています。この論文では、状態 **s** に更新 **u** を適用することを **s↓u** と書いています。 **"↓u"** の部分を状態を変化させる *作用* として見ることができるので、著者は *action* という名前を使っています。私は全体(部分適用された操作)よりもこの演算 **↓** により言及しているので、代わりに *apply* を使おうと思います。

.. ==============================================
   Stateful computations in F# with update monads
   ==============================================

=================================================
Update モナドを使った F# における状態を持った計算
=================================================

:原文: `Stateful computations in F# with update monads - Tomas Petricek's blog <http://tomasp.net/blog/2014/update-monads/index.html>`_
:原著者: `Tomas Petricek <twitter.com/tomaspetricek>`_

.. Most discussions about monads, even in F#, start by looking at the well-known standard monads for handling state from Haskell. The *reader* monad gives us a way to propagate some read-only state, the *writer* monad makes it possible to (imperatively) produce values such as logs and the *state* monad encapsulates state that can be read and changed.

F# においても、モナドについてのほとんどの議論は、Haskell で状態を扱うためのよく知られた標準的なモナドを見てみることから始まります。 *Reader* モナドはいくつかの読み取り専用な状態を伝播する方法を提供し、 *Writer* モナドは(手続き的に)ログのような値を生成できるようにし、 *State* モナドは読み取ったり変更したりすることができる状態を隠蔽します。

.. These are no doubt useful in Haskell, but I never found them as important for F#. The first reason is that F# supports state and mutation and often it is just easier to use a mutable state. The second reason is that you have to implement three different computation builders for them. This does not fit very well with the F# style where you need to name the computation explicitly, e.g. by writing ``async { ... }`` (See also my `recent blog about the F# Computation Zoo paper <http://tomasp.net/blog/2013/computation-zoo-padl/>`_).

これらは Haskell では疑いようもなく便利なんですが、F# でも重要なものだと思った試しがありません。第一の理由が F# は状態と変化をサポートし、可変な状態を使うのがたんに簡単だというのがしょっちゅうあるからです。第二の理由はそれらのモナド用に3つの異なるコンピュテーションビルダーを実装しなくちゃあいけないということです。これは明示的に計算に名前をつける、例えば ``async { ... }`` と書く必要があるという F# のスタイルにあまりうまく合いません。(`The F# Computation Zoo の論文についての最近のブログ記事 <http://tomasp.net/blog/2013/computation-zoo-padl/>`_ も参照してください)

.. When visiting the Tallinn university in December (thanks to James Chapman, Juhan Ernits & Tarmo Uustalu for hosting me!), I discovered the work on update monads by Danel Ahman and Tarmo Uustalu on `update monads <http://cs.ioc.ee/~tarmo/papers/types13.pdf>`_, which elegantly unifies *reader*, *writer* and *state* monads using a single abstraction.

12月にタリン大学を訪れた時(私をホストしてくれた James Capman、Juhan Ernits そして Tarmo Uustalu に感謝を！)、 `Update monads の論文 <http://cs.ioc.ee/~tarmo/papers/types13.pdf>`_ で Danel Ahman と Tarmo Uustalu による *update monads* の研究成果を見つけましたが、それは単一の抽象で *Reader* 、 *Writer* そして *State* モナドをエレガントにまとめるものでした。

.. In this article, I implement the idea of +update monads* in F#. Update monads are parameterized by *acts* that capture the operations that can be done on the state. This means that we can define just a single computation expression ``update { ... }`` and use it for writing computations of all three aforementioned kinds.

本記事で、私は F# で *Update モナド* のアイディアを実装します。Update モナドは state 上で完了した演算をキャプチャする *acts* によってパラメタライズされます。どういうことかというと、私たちはたった1つのコンピュテーション式 ``update { ... }`` を定義でき、前述した3種全ての計算を書くために使うことができる、ということです。

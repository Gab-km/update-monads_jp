=================================================
Update モナドを使った F# における状態を持った計算
=================================================

:原文: `Stateful computations in F# with update monads - Tomas Petricek's blog <http://tomasp.net/blog/2014/update-monads/index.html>`_
:原著者: `Tomas Petricek <twitter.com/tomaspetricek>`_

F# においても、モナドについてのほとんどの議論は、Haskell で状態を扱うためのよく知られた標準的なモナドを見てみることから始まります。 *Reader* モナドはいくつかの読み取り専用な状態を伝播する方法を提供し、 *Writer* モナドは(手続き的に)ログのような値を生成できるようにし、 *State* モナドは読み取ったり変更したりすることができる状態を隠蔽します。

これらは Haskell では疑いようもなく便利なんですが、F# でも重要なものだと思った試しがありません。第一の理由が F# は状態と変化をサポートし、可変な状態を使うのがたんに簡単だというのがしょっちゅうあるからです。第二の理由はそれらのモナド用に3つの異なるコンピュテーションビルダーを実装しなくちゃあいけないということです。これは明示的に計算に名前をつける、例えば ``async { ... }`` と書く必要があるという F# のスタイルにあまりうまく合いません。(`The F# Computation Zoo の論文についての最近のブログ記事 <http://tomasp.net/blog/2013/computation-zoo-padl/>`_ も参照してください)

12月にタリン大学を訪れた時(私をホストしてくれた James Capman、Juhan Ernits そして Tarmo Uustalu に感謝を！)、 `Update monads の論文 <http://cs.ioc.ee/~tarmo/papers/types13.pdf>`_ の Danel Ahman と Tarmo Uustalu による *update monads* の仕事を発見しましたが、それは1つの抽象を用いて *Reader* 、 *Writer* そして *State* モナドをエレガントにまとめていました。

本記事で、私は F# で *Update モナド* のアイディアを実装します。Update モナドは state 上で完了した演算をキャプチャする *acts* によってパラメタライズされます。どういうことかというと、私たちはたった1つのコンピュテーション式 ``update { ... }`` を定義でき、前述した3種全ての計算を書くために使うことができる、ということです。

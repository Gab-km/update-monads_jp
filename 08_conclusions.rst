..
   Conclusions
   ===========

 結び
======

..
   People coming to F# from the Haskell background often dislike the fact that F# does not let you write code polymorphic over monads and that computation expressions always explicitly state the type of computations such as ``async { .. }``. I think there are good reasons for this and tried to explain some of them in `a recent blog post and PADL paper <http://tomasp.net/blog/2013/computation-zoo-padl>`_.

Haskellのバックグラウンドを持つ人々がF#をみるとき、F#がモナドを用いて多態的にコードを書かせてくれず、コンピュテーション式が、 ``async { .. }`` のように常に計算の方を明示的に示さなければいけないことを、よく嫌います。これには良い理由があると私は考えていて、それらの幾つかについては `最近のBlog投稿とPADAの論文 <http://tomasp.net/blog/2013/computation-zoo-padl>`_ で説明を試みています。

..
   As a result, using reader, writer and state monads in F# was always a bit cumbersome. In this blog post, I looked at an F# implementation of the recent idea called *update monads* (see `the original paper (PDF) <http://cs.ioc.ee/~tarmo/papers/types13.pdf>`_), which unifies the three state-related monads into a single type. This works very nicely with F# - we can define just a single computation builder for all state-related computations and then define a concrete state-related monad by defining two simple types. I used the approach to define a reader monad, writer monad useful for logging and a state monad (that keeps a state and allows changing it).

結果から言えば、Reader・Writer・Stateの各モナドをF#で使うのは、常に多少面倒を伴います。このBlog投稿では、 *Updateモナド* と名付けられた、3つの状態に関連したモナドを一つの型に統合する最近のアイデア （詳しくは `オリジナルの論文 (PDF) <http://cs.ioc.ee/~tarmo/papers/types13.pdf>`_ を読んで下さい ） のF#実装を見ますた。これはF#上で非常に上手く動きます - たった一つのコンピュテーション式ビルダーを、状態に関連した全ての計算用に定義することができ、実際の状態に関連したモナドの定義を、2つの単純な型を決めるだけでできてしまうのです。私はこのアプローチを、Readerモナドと、ログに有用なWriterモナドと、Stateモナド（状態を保持し変更できます）を定義するのに用いました。

..
   I guess that making update monads part of standard library and standard programming style in Haskell will be tricky because of historical reasons. However, for F# libraries that try to make purely functional programming easier, I think that update monads are the way to go.

私の考えでは、UpdateモナドをHaskellの標準的なライブラリやプログラミングスタイルに取り込むのは、歴史的な理由によりトリッキーになるでしょう。しかしながら、純粋関数型プログラミングをより簡単にしようとしているF#のライブらいにとっては、Updateモナドは取りうる未知の一つではないかと思っています。

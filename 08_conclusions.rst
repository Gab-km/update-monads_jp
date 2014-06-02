Conclusions
===========

People coming to F# from the Haskell background often dislike the fact that F# does not let you write code polymorphic over monads and that computation expressions always explicitly state the type of computations such as ``async { .. }``. I think there are good reasons for this and tried to explain some of them in `a recent blog post and PADL paper <http://tomasp.net/blog/2013/computation-zoo-padl>`_.

As a result, using reader, writer and state monads in F# was always a bit cumbersome. In this blog post, I looked at an F# implementation of the recent idea called *update monads* (see `the original paper (PDF) <http://cs.ioc.ee/~tarmo/papers/types13.pdf>`_), which unifies the three state-related monads into a single type. This works very nicely with F# - we can define just a single computation builder for all state-related computations and then define a concrete state-related monad by defining two simple types. I used the approach to define a reader monad, writer monad useful for logging and a state monad (that keeps a state and allows changing it).

I guess that making update monads part of standard library and standard programming style in Haskell will be tricky because of historical reasons. However, for F# libraries that try to make purely functional programming easier, I think that update monads are the way to go.

..
   -----------------------------
   Implementing the reader monad
   -----------------------------

====================
 Readerモナドの実装
====================

The reader monad keeps some state, but it does not give us any way of modifying it. In terms of update monads, this means that there is some state, but the monoid of updates is trivial - in principle, we can just use ``unit`` as the type of updates. You can see that when looking at the type too - the type of reader monad is ``'TState -> 'T``. To get a type with a structure matching to update monads, we can use an equivalent type ``'TState -> unit * 'T``.

Reader state and update
-----------------------

In practice, we still need to define a type for updates, so that we can provide the required static members. We use a single-case discriminated union with just one value ``NoUpdate``:

.. code-block:: fsharp

  /// The state of the reader is 'int'
  type ReaderState = int
  /// Trivial monoid of updates
  type ReaderUpdate =
    | NoUpdate
    static member Unit = NoUpdate
    static member Combine(NoUpdate, NoUpdate) = NoUpdate
    static member Apply(s, NoUpdate) = s

None of the operations on the ``ReaderUpdate`` type does anything interesting. Both unit and combine simply returns the only possible value and the apply operation returns the state without a change.

Reader monad primitives
-----------------------

Next, we need a primitive that allows us to read the state and a run operation that executes a computation implemented using the reader monad (given the value of the read-only state). The operations look as follows:

.. code-block:: fsharp

  /// Read the current state (int) and return it as 'int'
  let read = UM (fun (s:ReaderState) ->
    (NoUpdate, s))
  /// Run computation and return the result
  let readRun (s:ReaderState) (UM f) = f s |> snd

When you look at the type of computations (hover the mouse pointer over the ``read`` identifier), you can see a parameterized update monad type. The ``read`` primitive has a type ``UpdateMonad<ReaderState, ReaderUpdate, ReaderState>``. This means that we have an update monad that uses ``ReaderState`` and ``ReaderUpdate`` as the *act* (specifying the computation details) and, when executed, produces a value of ``ReaderState``.

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

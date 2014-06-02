Building richer computations
============================

One of the key things about F# computation expressions that I emphasized in `my previous blog post <http://tomasp.net/blog/2013/computation-zoo-padl>`_ and in the `PADL 2014 paper <http://tomasp.net/academic/papers/computation-zoo/>`_ is that computation expressions provide rich syntax that includes resource management (the ``use`` keyword), exception handling or loops (``for`` and ``while``) - in simple words, it mirrors the normal syntax of F#.

So far, we have not used any of these for update monads. All these additional constructs have to be provided in the computation builder (so that the author can define them in the most suitable way). The great thing about update monads (for F#) is that we have just a single computation builder and so we can define a number of additional operations to enable richer syntax.

The following snippet extends ``UpdateBuilder`` defined earlier with more operations. If you're not interested in the details, feel free to skip to the next section. The key idea is that this only has to be written once!

.. code-block:: fsharp

/// Extends UpdateBuilder to support additional syntax
  type UpdateBuilder with
    /// Represents monadic computation that returns unit
    /// (e.g. we can now omit 'else' branch in 'if' computation)
    member inline x.Zero() = x.Return(())

    /// Delays a computation with (uncontrolled) side effects
    member inline x.Delay(f) = x.Bind(x.Zero(), f)

    /// Sequential composition of two computations where the
    /// first one has no result (returns a unit value)
    member inline x.Combine(c1, c2) = x.Bind(c1, fun () -> c2)

    /// Enable the 'return!' keyword to return another computation
    member inline x.ReturnFrom(m : UpdateMonad<'S, 'P, 'T>) = m

    /// Ensure that resource 'r' is disposed of at the end of the
    /// computation specified by the function 'f'
    member inline x.Using(r,f) = UM(fun s ->
      use rr = r in let (UM g) = f rr in g s)

    /// Support 'for' loop - runs body 'f' for each element in 'sq'
    member inline x.For(sq:seq<'V>, f:'V -> UpdateMonad<'S, 'P, unit>) =
      let rec loop (en:System.Collections.Generic.IEnumerator<_>) =
        if en.MoveNext() then x.Bind(f en.Current, fun _ -> loop en)
        else x.Zero()
      x.Using(sq.GetEnumerator(), loop)

    /// Supports 'while' loop - run body 'f' until condition 't' holds
    member inline x.While(t, f:unit -> UpdateMonad<'S, 'P, unit>) =
      let rec loop () =
        if t() then x.Bind(f(), loop)
        else x.Zero()
      loop()

You can find more details about these operations in the `F# Computation Zoo paper <http://tomasp.net/academic/papers/computation-zoo/>`_ or in the `F# langauge specification <http://fsharp.org/about/index.html#specification>`_. In fact, the defitions mostly follow the samples from the F# specification. It is worth noting that all the members are marked as ``inline``, which allows us to use *static member constrains* and to write code that will work for any update monad (as defined by a pair of *update* and *state* types).

Let's look at a trivial example using the writer computation:

.. code-block:: fsharp

  /// Logs numbers from 1 to 10
  let logNumbers = update {
    for i in 1 .. 10 do
      do! write i }

As expected, when we run the computation using ``writeRun``, the result is a tuple containing a list with numbers from 1 to 10 and a unit value. The computation does not explicitly return and so the ``Zero`` member is automatically used.

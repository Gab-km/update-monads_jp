-----------------------------
Implementing the writer monad
-----------------------------

Similarly to the reader monad, the writer monad is just a simple special case of the update monad. This time, the *state* is trivial and all the interesting things are happening in the updates. The type of the usual writer monad is ``'TState * 'T`` and so if we want to make this a special case of update monad, we can define the type as ``unit -> 'TState * 'T``.

Writer state and update
-----------------------

The state needs to be a monoid (with unit and composition) so that we can compose the states of multiple sub-computations. The following example uses a list as a concrete example. We define a (writer) monad that keeps a list of ``'TLog`` values and returns that as the result (more generally, we could use an arbitrary monoid instead of a list):

.. code-block:: fsharp

  /// Writer monad has no readable state
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

The writer monad appears (in some informal sense) dual to the earlier reader monad. The state (that can be read) is always empty and is represented by the ``NoState`` value, while all the interesting aspects are captured by the ``WriterUpdate`` type - which is a list of values produced by the computation. The updates of a writer monad have to form a monoid - here, we use a list that concatenates all logged values. You could easily change the definition to implement other monoids (e.g. to keep the last produced value).

Writer monad primitives
-----------------------

Similarly to the previous example, we now need two primitives - one to add new element to the log (``write`` of the writer monad) and one to run a computation and extract the result and the log:

.. code-block:: fsharp

  /// Writes the specified value to the log
  let write v = UM (fun s -> (Log [v], ()))
  /// Runs a "writer monad computation" and returns
  /// the log, together with the final result
  let writeRun (UM f) = let (Log l, v) = f NoState in l, v

The ``write`` function creates a singleton list containing the specified value ``Log [v]`` as the update and returns the unit value as the result of the computation. When combined with other computations, the updates are concatenated and so this will become a part of the list ``l`` in the result ``(Log l, v)`` that is made accessible in the ``writerRun`` function.

Sample writer computations
--------------------------

Let's have a look at a sample computation using the new definitions - the remarkable thing (from the practical F# programming perspective) is that we wrap the computation in the ``update { .. }`` block just like in the previous example. But this time, we'll use the ``write`` primitive to write 20 and then 10 to the log and the F# compiler correctly infers that we are using ``WriterState`` and ``WriterUpdate`` types:

.. code-block:: fsharp

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

If you run the code, the ``demo3`` computation first writes 20 to the log, which is then combined (using the ``++`` operator that invokes ``WriterUpdate.Combine``) with the value 10 written in ``demo4``.

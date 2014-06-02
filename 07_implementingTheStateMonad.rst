Implementing the state monad
============================

Interestingly, standard state monad is *not* a special case of update monads. However, we can define a computation that implements the same functionality - a computation with state that we can read and write.

States and updates
------------------

In this final example, both the type representing *state* and the type representing *update* will have a useful role. We make both of the types generic over the value they carry. State is simply a wrapper containing the value (current state). Update can be of two kinds - we have an empty update (do nothing) and an update to set the state:

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

This definition is a bit more interesting than the previous two, because there is some interaction between the *states* and *updates*. In perticular, when the update is ``Set v`` (we want to replace the current state with a new one), the ``Apply`` member returns a new state instead of the original. For the ``Unit`` member, we need an update ``SetNop`` which simply means that we want to keep the original state (and so ``Apply`` just returns the original value in this case).

Another notable thing is the ``Combine`` operation - it takes two updates (which may be either empty updates or set updates) and produces a single one. If you read a composition ``a1 ++ a2 ++ .. ++ an`` as a sequence of state updates (either ``Set`` or ``SetNop``), then the ``Combine`` operation returns the last ``Set`` update in the sequence (or ``SetNop`` if ther are no ``Set`` updates). In other words, it builds an update that sets the last state that was set during the whole sequence.

State monad primitives
----------------------

Now that we have the type definitions, it is quite easy to add the usual primitives:

.. code-block:: fsharp

  /// Set the state to the specified value
  let set s = UM (fun _ -> (Set s,()))
  /// Get the current state
  let get = UM (fun (State s) -> (SetNop, s))
  /// Run a computation using a specified initial state
  let setRun s (UM f) = f (State s) |> snd

The ``set`` operation is a bit different than the usual one for state monad. It ignores the state and it builds an *update* that tells the computation to set the new state. The ``get`` operation reads the state and returns it - but as it does not intend to change it, it returns ``SetNop`` as the update.

Sample stateful computation
---------------------------

If you made it this far in the article, you can already expect how the example will look! We'll again use the ``update { .. }`` computation. This time, we define a computation ``demo5`` that increments the state and call it from a loop in ``demo6``:

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

Running the code yields 10 as expected - we start with zero and then increment the state ten times. Since we extended the definition of the ``UpdateBuilder`` (in the previous section), we now got a few nice things for free - we can use the ``for`` loop and write computations (like ``demo5``) without explicit ``return`` if they just need to modify the state.

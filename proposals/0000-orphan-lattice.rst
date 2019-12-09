Orphan Lattice
==============

.. author:: John Ericson
.. date-accepted:: Leave blank. This will be filled in when the proposal is accepted.
.. proposal-number:: Leave blank. This will be filled in when the proposal is
                     accepted.
.. ticket-url:: Leave blank. This will eventually be filled with the
                ticket URL which will track the progress of the
                implementation of the feature.
.. implemented:: Leave blank. This will be filled in with the first GHC version which
                 implements the described feature.
.. highlight:: haskell
.. header:: This proposal is `discussed at this pull request <https://github.com/ghc-proposals/ghc-proposals/pull/0>`_.
            **After creating the pull request, edit this file again, update the
            number in the link, and delete this bold sentence.**
.. contents::

Linearized, especially arbitrarily so, dependencies are terrible.
Orphans avoid linearization, and in doing so get us closer to a normal form.
Cabal keeps track of coherence.

Motivation
----------

As Ed Kmett laments in his `FAQ for Coda`_, Haskell today forces dependencies to be linearized.
First picture the traditional data types x functionality plane used to illustrate the expression problem.
In avoiding orphan instances, we are left with a chain of dependencies where each one adds perhaps just a single type or instance, but then is forced to cover all the newly created "area" in new instances.
You can imaging dependency chain as a confused weft that cannot decide which axis the warp should go in, and so meanders around as an ugly and paranoid space-filling curve.
This is bad because the quadratically increasing size of packages towards the end makes adding new functionality unscalable.
Either you write larger and larger new packages at the end of the chain, or you are left with larger downstream refactors after slipping in a more judicious package "mid chain".

But if we write orphans, we can fill out a semilattice incrementally.
Make a new type, make a new class, fill out the missing instances in other packages later.
Of course, we want that quadratic new functionality in the end, and this proposal doesn't goes as far as Coda in proposing tactics to fill in missing instances automatically.
But by at least allowing the development to happen linearly even as desired development grows super-linearly, we reclaim some ability to grow the ecosystem.

This is not a new observation.
There is a reddit thread (TODO find) about this where Ed mentions embracing orphans some how, and @isovector mentions his `https://github.com/isovector/wide-open-world`_.
I'm not quite sure what exact plans Ed has, but I can say that ``wide-open-world`` would be a quadratically growing monorepo that, while avoiding and silly linearities in dependencies, nevertheless sequentializes development with the git history of one repo replacing the cabal dependency history as the culprit.
I like how package repositories and version bounds keep things "eventually consistent", even before it was cool, and I don't want to advocate for such a dramatic shift in our tools and social processes at this time.

The trick is really simple.
Instances are anti-modular, but so are packages if you squint just right.
Just as there must only be one instance globally in the program, so there must be one version of a package in the package solution.
We simply piggy-back on the package manager's ability to solve with constraints and throw more coherence constraints at it.
GHC currently calls modules which contains orphan instances "orphans modules".
Well, declare them and call them "adopting modules", because they adopt the orphan instances the parent modules didn't provide, and simply require that Cabal ensure no set of parents' offspring are adopted twice.
Coherence is preserved.

Theory
------

OK, it is a bit more complicated than let by, because

 - due to flexible instances and multi-parameter type classes not every instance is clearly born of one class and one type.

 - due to reexported modules not every type constructor and class definition has a canonical location.

 - backpack *shuddgers*

Not all of this can be dealt with, but we can make progress.

``import``s and custody
~~~~~~~~~~~~~~~~~~~~~~~

Imports (ignoring ``hs-boot`` files) are acyclic.
We can take their reflexive-transitive closure to get a partial order.
We can ignore any modules that don't provide types, classes, or instances, removing them from the relation.

The core feature of this proposal is we wish to construct an n-ary partial semi-lattice.
The intuition is every set of nodes has 0 or 1 unique joins (versus an arbitrary finite partial order where it arbitrary joins).
TODO formal definition.

The use of our lattice will be to index the distinguished modules where instances are allowed to reside.
Given a simple ``C T`` head of an instance, the instance goes in ``m(C) /\ m(T)``, where ``m`` maps a definition to the module it is defined in.
In the non-orphan cases we have today, either ``m(C) <= m(T)`` or ``m(T) <= m(C)``, so the join is whatever the "downstream" module is.
However, if neither module transitively imports each other, a third module that importants both can be declared as the canonical join.
The ``C T`` instance goes in there.
This covers the binary case.

Remember a ``C T`` constraint isn't necessarily resolved with a ``C T`` instance.
There might instead be a ``forall t. C t`` instance.
In this unary case, the instance must be defined in ``/\{m(c)} = m(c)``, the module that defined the class, and another overlapping instance couldn't possibly be immediately written downstream without immediate catching it, but this simple example presages the phenomena arising from more complex instance heads.

Dually, we can see when trying to resolve a ``forall t. C t`` constraint we must look in ``m(c)`` for the instance.
We can formalize this by pretending that all type variables are defined in a bottom module imported by all others.
Now the lookup

Complex instance heads
~~~~~~~~~~~~~~~~~~~~~~

Instead of simple Haskell 98, we have a instance head structure of::

  data InstanceHead = MkInstanceHead Class [RoseTree (Either TyCon Var)]

Where ``RoseTree`` is defined as::

  data RoseTree a = MkRoseTree a [RoseTree a]

GHC currently says that any ``TyCon`` in the arguments to the class can rightfully own the instance, but means avoiding orphans isn't good enough to get coherence as there are multiple valid claimants to the instance.
``C T0 T1`` for example could be "owned" by ``/\{m(C), m(T0)}`` or ``/\{m(C), m(T1)}``.

Instance priority
~~~~~~~~~~~~~~~~~

We cannot fully resolve the problem, but at least we provide partial priority, with a partial order on instance heads.

First of all, desugar repeated vars as distinct vars with equality constraints.
But then recall we only care about instance *heads*, so we can ignore those new equality constraints that go in the context.
This means that::

  v0, v1 :: Var
  -------------
  v0 <= v1

Essentially, we only have on ``Var`` then, which we will also make the bottom element::

  v :: Var, x :: Either TyCon Var
  -----------------------------
  Left v <= x

Next, we add depth subtyping to ``List`` and ``RoseTree``::

  --------
  [] <= []

  x <= y
  lx <= ly
  --------------------
  (x : lx) <= (y : ly)

  x <= y
  lx <= ly
  ----------------------------------
  MkRoseTree x lx <= MkRoseTree y ly

And keep the bare variable bottom again::

  v :: Var
  r :: RoseTree (Either TyCon Var)
  ----------------------------------
  MkRoseTree (Left v) [] <= r

An finally to put it all together::

  args0 <= args1
  ------------------------------------------------
  MkInstanceHead c args0 <= MkInstanceHead c args1

Blaming incoherence
~~~~~~~~~~~~~~~~~~~

So with that definition, what can we do?


Proposed Change Specification
-----------------------------

#. Add a new pragma

   ::

     <ORPHANS_FROM> ::= {-# ORPHANS_FROM <parents-list> #-}

     <parents-list-item> ::= <module>
     <parents-list> ::= <parents-list-item>, (, <parents-list-item>)+)

   to go at the top of a module (like ``LANGUAGE`` pragmas).
   The parent form a set of at least two elements (note the ``+`` vs ``*``).
   (They should all be distinct, but if the precedent is for duplicate entries in sets to just have no effect, that's fine too.
   TODO confirm what the precedent is.)

   The parent modules must not be too related, i.e. no parent of them should directly or transitively import another one.

   The effect of this flag is to allow the orphans of these parents in the current module.
   Specifically, it allow
   orphans that wouldn't be orphans if all the definitions from the parent modules were moved into the current module,
   provided those orphans wouldn't be allowed if a proper subset of the parents was used instead.

   With ``-XPackageImports``, also allow

   ::

     <parents-list-item> ::= <module>
                          |  <package> <module>

   where package IDs are quoted just like package in imports, and ``"this"`` again specially refers to the current component.

#. Extend cabal files with new syntax::

     orphans:
       <package>
       <package>
       ...
       <module>:
         <package> <module>
         <package> <module>
         ...
       ...

   This goes within components.
   (Same position as e.g. a ``build-depends`` stanza.)
   If a module from the current package is to be used, ``"this"`` must be used rather than the package's name.
   TODO non-library components.
   The outer package list limits restricts what packages can be used in the inner lists in a specific way:
   Every inner list can either contain `"this"` and any (not necessarily subset) of the listed packages, or exactly the entire set of provided packages.

#. Add a flag to GHC (TODO syntax) to encode the above information.
   GHC will require that the specified orphan modules have the exactly matching ``{-# ORPHANS_FROM ... #-}`` pragma.
   Where the pragma has no explicit package, the providing package must be resolved to match what the Cabal file says.

#. Cabal, or any other package manager, must deem invalid solutions were the parent sets of all orphan-adopting modules are not distinct.
   It must also further restrict the parents' package relatedness as GHC restricts the parent modules themselves:
   The parent packages must be distinct and no parent package should directly or transitively import one another.
   It must also deem invalid solutions where within some parent set one parent depends on another.
   This is the "no incest" rule.

Specify the change in precise, comprehensive yet concise language. Avoid words
like "should" or "could". Strive for a complete definition. Your specification
may include,

* BNF grammar and semantics of any new syntactic constructs
* the types and semantics of any new library interfaces
* how the proposed change interacts with existing language or compiler
  features, in case that is otherwise ambiguous

Note, however, that this section need not describe details of the
implementation of the feature or examples. The proposal is merely supposed to
give a conceptual specification of the new feature and its behavior.


Examples
--------
This section illustrates the specification through the use of examples of the
language change proposed. It is best to exemplify each point made in the
specification, though perhaps one example can cover several points. Contrived
examples are OK here. If the Motivation section describes something that is
hard to do without this proposal, this is a good place to show how easy that
thing is to do with the proposal.


Effect and Interactions
-----------------------

- The rule saying that no proper subset of parent moduless would suffice means the class and arugments of every instance much make use of all of the specified parents.
  This in tern means that we don't need to worry if the parent modules of two orphan-providing modules overlap.

- The "unique custody" rule ensures that two modules cannot provide overlapping orphan instances.

- The "no incest" rule ensures that orphan modules' instances cannot overlap with instances defined next to the parent modules' classes or types.


Costs and Drawbacks
-------------------

Orphans are, contrary to popular belief, not the only source of incoherence in the type system.
See [Backpack '19] for details.


Alternatives
------------

I'm sure loads of things have been proposed over the years.
I have not followed much of the discussion.
Perhaps I am ignorant of my idea having been proposed before, and then rejected.


Unresolved Questions
--------------------

- Should parent only come in pairs, instead of sets of arbitrary >= 2 size?
  I can't think of anything that goes wrong without the restriction, and so I don't propose it, but we may still want to be defensive.


Implementation Plan
-------------------
(Optional) If accepted who will implement the change? Which other resources
and prerequisites are required for implementation?


Endorsements
-------------
(Optional) This section provides an opportunty for any third parties to express their
support for the proposal, and to say why they would like to see it adopted.
It is not mandatory for have any endorsements at all, but the more substantial
the proposal is, the more desirable it is to offer evidence that there is
significant demand from the community.  This section is one way to provide
such evidence.


.. _`FAQ for Coda`: https://github.com/ekmett/coda#what-is-all-this-about

.. [Backpack '19]: https://people.mpi-sws.org/%7Eskilpat/papers/kilpatrick-thesis-nov-2019-publication.pdf

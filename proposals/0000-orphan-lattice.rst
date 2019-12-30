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
The intuition is every set of nodes has 0 or 1 unique suprema (versus an arbitrary finite partial order where each has arbitrarily many suprema).
TODO formal definition.

The use of our lattice will be to index the distinguished modules where instances are allowed to reside.
Given a simple ``C0 T`` head of an instance, the instance goes in ``m(C) /\ m(T)``, where ``m`` maps a definition to the module it is defined in.
In the non-orphan cases we have today, either ``m(C) <= m(T)`` or ``m(T) <= m(C)``, so the join is whatever the "downstream" module is.
However, if neither module transitively imports each other, a third module that importants both can be declared as the canonical join.
The ``C0 T`` instance goes in there.
That third module may not be imported by all other modules that import ``C`` and ``T``, so it is not a regular semilattice join, but none of those other modules may define ``instance C0 T``, so we can forget they are suprema from the underlying ``import`` partial order and only worry about them insofar as they declare other types, classes, and instances.
This means our underlying partial order really doesn't just have the nodes filtered as already described, but also has the (generating) edges filtered to just those imports which are needed to bind classes and type constructors in instance heads.
All this covers the simple binary case.

Remember a ``C0 T`` constraint isn't necessarily resolved with a ``C0 T`` instance.
There might instead be a ``forall t. C0 t`` instance.
In this unary case, the instance must be defined in ``/\{m(c)} = m(c)``, the module that defined the class, and another overlapping instance couldn't possibly be immediately written downstream without immediate catching it, but this simple example presages the phenomena arising from more complex instance heads.

Dually, we can see when trying to resolve a ``forall t. C0 t`` constraint we must look in ``m(c)`` for the instance.
We can formalize this by pretending that all type variables are defined in a bottom module imported by all others.
That means that ``m(var) <= x`` for all variables.

Complex instance heads
~~~~~~~~~~~~~~~~~~~~~~

GHC Haskell isn't Haskell 98, and instance heads can rich applications of type constructors and variables, along with functional dependencies.
At last functional dependencies are easy to deal with; We only care about the "non-determined" ones in the instance head, which is to say we can pretend the rest were turned into associated type families and moved out of the instance head.
From here onward, by instance head we mean non-determined portion of the instance head.
This matches GHC's current behavior.

Anyways, GHC currently says that any type constructor in the arguments to the class can rightfully own the instance, but means avoiding orphans isn't good enough to get coherence as there are multiple valid claimants to the instance.
Today, we could have both ``forall a. C1 T0 a`` and ``forall a. C1 a T1`` declared in ``m(T0)`` and ``m(T1)``, respectively.
We actually *cannot* fix that problem.
But we can at least stop it from getting worse.
If we again take GHC's existing rules and naively adapt them,
``C1 T0 T1`` for example could be "owned" by ``/\'{C1, T0}`` or ``/\'{C1, T1}``, where ``/\' = /\ . fmap m``, a definition we will continue to use for brevity.
This is worse than before, as previously ``m(T0)`` and ``m(T1)`` would have to import ``C`` themselves, and so the overlap between ``C1 T0 T1`` and ``forall a. C1 T0 a`` or ``forall a. C1 a T1`` would be immediately caught.
But, we can fix this by saying the class and *all* the type constructors in the instance head.
What this means is that ``C1 T0 T1`` must go in ``/\'{C1, T0, T1}``, which must either transitively import for coincide with ``/\'{C1, T0}`` and ``/\'{C1, T1}``.
This plugs the leak, ensuring that an overlapping ``C1 T0 T1`` instance will be eagerly caught just like today.

Just to recap, this means that any instance (head) must transitively import all modules where more general instances (according to substitutability) are allowed to defined.
``m`` is a homomorphism from the partial order of instance heads (by substitutability) to the partial order of modules associated with our n-ary partial semi-lattice.
Where ``mset0`` is a subset of ``mset1``, ``/\mset0`` <= ``/\mset1`` if both exist.
That means the instance being defined in the join of the modules of the class and all type constructors is sufficient.

Modules, components, packages
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

So far, the lattice machinary has been described in terms of modules and their imports.
But the original premise of this proposal is that Cabal would coordinate which suprema modules are the canonicalized as joins in the n-ary partial semilattice.
Cabal knows of modules existence, but not their imports, so what bridges the gap?

The first thing we want to do is add some extra structure to our algebras.
We can map the underlying relations from modules to components by throwing away the specific module and just remembering what component defines it.
And we can do the same thing throwing away the component and just remembering the package that contains it.
These maps are necessarily partial order homomorphisms, because we already demand we can build entire packages at a time, but they aren't naturally guaranteed to be n-ary partial semilattice homormophisms.

Well, for both Cabal's and human programmers' sake, it is easier if we require that this be the case.
Thankfully, the demands that this introduces are fairly predictable and reasonable sounding.
They are:

  #. The join of modules from the same component must also be in that component.
  #. All join of modules from the same set of components must all be in a component declared to be the join of those components.
  #. The join of components from the same package must be in that package.
  #. All join of components from the same set of packages must all be in a package declared to be the join of those packages.

As one can see, we are just repeating the structure at every level.
It's a bit odd that we have this large heterogeneously named hierarchy.
If we ever get hierarchical modules, we would then have an arbitrary deep homogeneous hierarchy, and want similar rules to connect sibling modules and children to parents.
This would more succinctly describe the principle that is repeated in the 4 rules above.

Proposed Change Specification
-----------------------------

WIP

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

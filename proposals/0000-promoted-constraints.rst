Promoted Constraints
========================

.. author:: John Ericson (@Ericson2314)
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
.. sectnum::
.. contents::

This proposal sketches some changes to core, which finally provide a desugaring `-XDataContexts`, and a foundation for constrained type families.

Motivation
----------

There's two independent loose ends we'd like to tie up.

``-XDataContexts``
~~~~~~~~~~~~~~~~~

Data contexts may be changed or removed, so this might not seem important.
But if we better understand how they work today, we can better make that decision.

Constrained type families
~~~~~~~~~~~~~~~~~~~~~~~~~

Constrained type families are essential for type families to desugar to promoted functions.
They should hopefully also remedy up some long standing weirdness in type families.
To properly define them, however, it is useful to first have a notion of promoted constraints to build off of.

Proposed Change Specification
-----------------------------

1. Constraints exist at the kind level.
   Constraints are resolved to quantifiers (local constraints) and instances just as they are in terms
   [The quantifiers are distinct, see below, but the mechanism is the same.]
   [All instances are automatically promoted.]
   ``=>`` Can be used in kinds, including the surface syntax.

2. In core only (*not* exposed in the surface syntax), we have these 2 new quantifiers:

     - ``forall (_ :: C) =>``, invisible, irrelevant, and dependent
     - ``foreach (_ :: C) =>``, invisible, relevant, and dependent.

   All 3 quantifiers are paired with lambdas: ``\ @( :: C) =>``.
   [TODO strawman syntax for relevancy and dependence of lambda.]

   By invisible we mean there will be no explicit argument, this is true for all constraints.
   By relevant/irrelevant, we mean whether a dictionary exists at run time.
   By dependent, we mean whether a promoted constraint is bound in scope for the rest of the type, as opposed to the binding with this type.
   [Note that plain ``=>`` is *not* dependent, no constraint to the right of the ``=>`` is discharged.]

3. Data contexts desugar to extra quantifers on constructors::
    data C... => T... = A ... | B ... | ...
    
    ===>
    
    data T... where
      A :: forall (_ :: C...) => ... -> T
      B :: forall (_ :: C...) => ... -> T
      ...

Examples
--------

Go read `Effect and Interactions`_ first.

This section illustrates the specification through the use of examples of the
language change proposed. It is best to exemplify each point made in the
specification, though perhaps one example can cover several points. Contrived
examples are OK here. If the Motivation section describes something that is
hard to do without this proposal, this is a good place to show how easy that
thing is to do with the proposal.

Effect and Interactions
-----------------------

Operation semantics
~~~~~~~~~~~~~~~~~~~

The operational sematics are *not* changed.
In particular, nothing at the type/kind level actually depends on a value of a (compile time or run time) dictionary.
There are no new stuck terms, i.e. evaluation continues as normal inside quantifiers and lambdas.
Unification / type checking is free to take advantage of this.

Constrained type families should boil down to actually making an operational semantics change on top of this.
I don't think they need a static semantics change.

Existential quantification
~~~~~~~~~~~~~~~~~~~~~~~~

What this hell does this mean?

::
  data Foo :: Type where
    MkFoo :: forall (_ :: C) => Foo

Or, how does relevant/irellevant relate to existential/universal?
That latter distinction doesn't really exist in Haskell.
But, we can intuit effectively a universal variable can revealed through unification or substitution at compile time, while an existential can only be revelead though pattern matching at run time.
So an erased existential is a fun thing, as it cannot be directly revealed at run time.

In the above case, we have a skolem and erased dictionary.
We can use it to satisfy some typing rules, but we will never get any information out of it, should the operational semantics make use of promoted dictionary in the future.
However, by coherence it will unify with any not-so-ofuscated dictionary with the same type.
The unification holds inducively, which is good as dictionaries are not real things in the type system (we only have constraints).
If, in the future we have a stuck term blocked on this skolem,
and then we get a concrete dictionary with the same constraint-type in scope, we can un-stick our term.

Does that desugaring really not break code?
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

I really do think so!
Right now, types data contexts require the constraint to use a constructor, but do not provide that constraint what the constructor is pattern matched.
This is silly because introduction and elimination should be dual, but we didn't know how else to eliminate the dictionary while making ``-XDataContexts`` do anything at all.

Well, by in-effect stratifying the amount of constraint-satisfiedness we have (at the type level? is it relevant?) we can recover the duality.
It conveniently collapses to the non-duality when, other than on the data constructors themselves, we do not have the fancy new quantifiers at our disposal, preserving back-compat. Proof:

#. Introduction requires an erased constraint.
   We have no ``forall (_ :: C) =>`` syntax, so we use a plain ``=>``, which thereby becomes the effective surface-syntax requirement.
   There is no need to manually promote the local instance, nor do we need to use ``foreach (_ :: C) =>`` as the case is to the left of the ``:: ... => ...`` as per usual.

#. Elimination provides an erased constraint.
   But, lacking syntax or constraint kinds, there is nothing we can do with it but call another such constructor.
   We need to separately provide the regular ``=>``-bound constraint to get anything done.

Future of ``-XDataContexts``
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

In [TODO get exact paper], data contexts become this magic coeffect propagating the constraint to instances.
This, v.s. what is described, allows purely more programs, so is backwards compatible.

In the constraint contexts proposal, data contexts are necessary for using families in ADTs. But the implicit ``forall (_ :: C) =>` quantifier also puts the irrelevant constraint in scope over the rest of the fields, just where it needs to be to use a family.

Costs and Drawbacks
-------------------
Even @goldfirere thinks this is too many quantifiers! :D

Alternatives
------------
We could have plain ``C =>`` be ``foreach (_ :: C) =>``, so there is no way to stop dependent use of a constraint.

Unresolved Questions
--------------------
Somebody needs to go right some judgements.

Implementation Plan
-------------------
Once we have some real judgements, I could help pitch in implementing.
But I suspect this will take a good amount of time from the people that change the type system more than me.

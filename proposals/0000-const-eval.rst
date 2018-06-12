Notes on reStructuredText - delete this section before submitting
==================================================================

The proposals are submitted in reStructuredText format.  To get inline code, enclose text in double backticks, ``like this``.  To get block code, use a double colon and indent by at least one space

::

 like this
 and

 this too

To get hyperlinks, use backticks, angle brackets, and an underscore `like this <http://www.haskell.org/>`_.


Constant evaluation
===================

.. proposal-number:: Leave blank. This will be filled in when the proposal is
                     accepted.
.. trac-ticket:: Leave blank. This will eventually be filled with the Trac
                 ticket number which will track the progress of the
                 implementation of the feature.
.. implemented:: Leave blank. This will be filled in with the first GHC version which
                 implements the described feature.
.. highlight:: haskell
.. header:: This proposal is `discussed at this pull request <https://github.com/ghc-proposals/ghc-proposals/pull/0>`_.
            **After creating the pull request, edit this file again, update the
            number in the link, and delete this bold sentence.**
.. sectnum::
.. contents::

This formalizes a notion compile-time evaluation for zero-cost abstractions, and certain backend functionality.
Expressions may be "constant", meaning they can be fully evaluated at compile time, or "constant-erased" meaning they additionally must be erased at run time.
Additional "relevant, yet erased" quantifiers are introduced for sake of abstractions with constants, which arguably fill in gaps in the existing Dependent Haskell plans.


Motivation
------------

Guaranteed Optimization
~~~~~~~~~~~~~~~~~~~~~~~

The essence of most optimization is partial evaluation:
Inlining is (lazy) value-abstraction elimination.
Specialization is lazy type-abstraction elimination.
constant propagation is the full evaluation of sub-expressions.
Even other rewrite rules that don't stem directly from the dynamic semantics still often only are applicable after some partial evaluation.

Performing a single reduction is trivial.
But each reduction opens the door to potentially more reductions, making exploring the space of possible partial evaluation a formidable search problem.
Supercompilation especially runs afoul of this cost, but even more common optimization passes impose some unpredictability.
When unoptimized builds are fast enough, this is fine: extra performance is a happy surprise.
But when they aren't, the programmer is put in a stressful and uncertain situation---not unlike reasoning about functional correctness without the aid of types.

From a compiler-author's standpoint, type systems are *the* way to get away with a local/compositions analysis instead of a global analysis.
Here is no exception.
We can mark terms "constant" if they can be evaluated at compile time, and then the partial partial evaluator knows that exploring a "constant" subterm will always make progress.

At first glance, this is just a fancy way of describing the constant-folding pass that just about every compiler has.
The key, however, is identifiers can also be marked constant if we know they will only be substituted for constant terms.






Here you should describe in greater detail the motivation for the change. This
should include concrete examples of the shortcomings of the current
state of things.

Backend Necessity
~~~~~~~~~~~~~~~~~


Give a strong reason for why the community needs this change. Describe the use case as clearly as possible and give an example. Explain how the status quo is insufficient or not ideal.


Proposed Change Specification
-----------------------------

"Constance" of terms
~~~~~~~~~~~~~~~~~~~~~~~

Just about any compiler, GHC included, has the functionality to identify closed terms as constants.
We propose extending this analysis in an effect system where every term is deemed either constant or non-constant, and any term

As mentioned, all our existing quantifiers will introduce normal non-constant bindings.
We will need a new family of quantifiers

``forall a :: k -.`` asdf ``forall a :: k -.``

Constant-qualifiers
~~~~~~~~~~~~~~~~~~~




Specify the change in precise, comprehensive yet concise language. Avoid words like should or could. Strive for a complete definition. Your specification may include,

* grammar and semantics of any new syntactic constructs
* the types and semantics of any new library interfaces
* how the proposed change interacts with existing language or compiler features, in case that is otherwise ambiguous

Note, however, that this section need not describe details of the implementation of the feature. The proposal is merely supposed to give a conceptual specification of the new feature and its behavior.


Effect and Interactions
-----------------------
Detail how the proposed change addresses the original problem raised in the motivation.

Discuss possibly contentious interactions with existing language or compiler features.

``RuntimeRep``
~~~~~~~~~~~~~~

Currently, we have a rule "No variable may have a levity-polymorphic type".
We can relax this so that the kinds of the types of variables can contain variables as long as those are bound with a specializing quantifier.
This will make greatly increase the expressiveness of code working with exotic runtime represenations.

Costs and Drawbacks
-------------------
Give an estimate on development and maintenance costs. List how this effects learnability of the language for novice users. Define and list any remaining drawbacks that cannot be resolved.


Alternatives
------------
List existing alternatives to your proposed change as they currently exist and discuss why they are insufficient.


Unresolved questions
--------------------
Explicitly list any remaining issues that remain in the conceptual design and specification. Be upfront and trust that the community will help. Please do not list *implementation* issues.

Hopefully this section will be empty by the time the proposal is brought to the steering committee.


Implementation Plan
-------------------
(Optional) If accepted who will implement the change? Which other ressources and prerequisites are required for implementation?

Notes on reStructuredText - delete this section before submitting
==================================================================

The proposals are submitted in reStructuredText format.  To get inline code, enclose text in double backticks, ``like this``.  To get block code, use a double colon and indent by at least one space

::

 like this
 and

 this too

To get hyperlinks, use backticks, angle brackets, and an underscore `like this <http://www.haskell.org/>`_.


Phase-separated Template Haskell
================================


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

Redo Template Haskell with phase separation so that it is suitable to use with all manner of bootstrapping and cross compilation.
This permanently removes a crucial roadblock against obscure platforms becoming first-class citizens of the Haskell ecosystem, and also is the first step towards obviating the C Preprocessor.


Motivation
------------

Cross compilation and TH currently don't mix.
The best workarounds are iserv, running TH on a delegated external process with the compiler's target architecture,
and the upcoming ability for GHC to import previously dumped splices.



This proposal picks up `<http://blog.ezyang.com/2016/07/what-template-haskell-gets-wrong-and-racket-gets-right/>`_, and my old email `<https://mail.haskell.org/pipermail/ghc-devs/2016-January/011064.html>`_.
The former and maybe the latter might be useful.

Proposed Change Specification
-----------------------------
Specify the change in precise, comprehensive yet concise language. Avoid words like should or could. Strive for a complete definition. Your specification may include,

* grammar and semantics of any new syntactic constructs
* the types and semantics of any new library interfaces
* how the proposed change interacts with existing language or compiler features, in case that is otherwise ambiguous

Note, however, that this section need not describe details of the implementation of the feature. The proposal is merely supposed to give a conceptual specification of the new feature and its behavior.

These changes constitute a new ``PhasedTemplateHaskell`` extension, as they are incompatible with TH as it exists today.
``PhasedTemplateHaskell`` and ``TemplateHaskell`` will be disallowed in the same module.

Phase separation
~~~~~~~~~~~~~~~~

The essence of phase-separated macro systems is a duplication of binding name spaces to create an indexed family.
binding constructs become indexed, with identifier resolution limited to the usual namespaces, now families, applied to that index. For example, whereas
::
  data Foo   = Nil | Bar Foo
  data Foo a = Nil | Bar (Foo a)
is a name-clash (at different types) today,
::
  $[0] data Foo   = Nil | Bar Foo
  $[1] data Foo a = Nil | Bar (Foo a)
is fine tomorrow, where ``$[integer-literal]`` at the beginning of the definition is used to chose the phase.
The first ``Foo`` resolves to ``Foo`` in the 0 (type) namespace, with type ``Type``, while the second ``Foo`` resolves to ``Foo`` in the 1 (type) namespace, with a type ``Type -> Type``.

The most interesting binding form is ``import``. Given some integer constant ``n``:
::
  $[n] import Foo
imports ``Foo``\'s ``k + n``` namespaces into the current module's ``n`` namespaces.
As such, the ``n`` index is best thought of as an offset.
Firstly, this is because all namespaces in parallel are shifted.
Secondly, this is because given that we don't know (for sake of composition) how the current module will be imported, it is global to all modules just relative.

Infinite module graphs, like cyclic ones, are prohibited. An simple example would be:
::
  module Main where
  $[1] import Main
Unlike their cyclic counterparts, there is no ``.hs-boot``-like escape hatch that allows for this.
It is just illegal.

For convenience and backwards compatibility, we provide the following 2 sugars:
::
  binding-construct
  ==> $[0] binding-construct
::
  $binding-construct
  ==> $[1] binding-construct

For practical purposes, we only need 2 phases.
That said, both the implementation and pedagogy are clarified by allowing namespaces to be freely generated through shifted binding constructs
Instead, we can enforce post-hoc constraints on the space of generated namespaces *in actuality* for sake of ecosystem cohesion.

Quoting and splicing
~~~~~~~~~~~~~~~~~~~~

The core of any macro system is quoting and splicing.
In Template Haskell the core quoting constructs are ``[e| |]``, ``[d| |]``, ``[t| |]``, ``[p| |]``, while the core splicing construct is ``$(..)``.
To connect phases and macros, we require that quoted syntax in phase ``n`` be resolved in phase ``n - 1``, while a splicing expression in stage ``n`` is resolved (and evaluated according) to phase ``n + 1``.
That means:
::
  {- stage n -}  [| {- stage n - 1 -} [| {- stage n - 2 -} ... |] |]
::
  {- stage n -}  $( {- stage n + 1 -} $( {- stage n + 2 -} ... ) )

Quasi-quoting is in Haskell is misnamed and has nothing to do with quasi-quoting in Lisp.
That said, quasi-quoting is relevant in that the provided parsing function is resolved in stage ``n + 1`` like the splicing constructs.

Typed template Haskell unfortunately has no easy phases analog.
The problem is the phantom type argument would need to come from a different stage as the ``TExpr`` to which it is applied.
We will hold off on this for now.

Cabal integration
~~~~~~~~~~~~~~~~~

Effect and Interactions
-----------------------

The big boon for cross compilation stems from the fact that that splicing is the only thing which forces (fully general) evaluation during compilation.

With this change, spliced expressions

Detail how the proposed change addresses the original problem raised in the motivation.

Discuss possibly contentious interactions with existing language or compiler features.


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

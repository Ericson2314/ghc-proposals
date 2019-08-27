Modularize GHC
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
.. sectnum::
.. contents::

Meta-proposal for a number of different efforts to make GHC easier to develop and use as a library.
The interactions are subtle and I believe deserve a unified treatment.

Motivation
----------

Haskell prides itself on virtues such as code reuse, modularity, abstraction, and composition.
GHC, one of the oldest code bases still in wide use, does not live up to those virtues.
There has long been interest in refactoring GHC to make it live up to this, but the path is long and tangled.

There are number of different efforts in this direction, but they are not entirely parallel.
Individual MRs and proposals do a poor job of capturing the overall plan.
Moreover, while the pieces are individually motivated, their real prize is all the work together.

First, This proposal kicks things off by proposing an initial plan, in the hopes that the proposal process will give it a wide audience.
But the exact plan will inevitably change, so it secondarily proposes keeping a wiki page up to date with the current plan and current status.
Strictly speaking, since the MRs and other proposals are reviewed and approved individually, the wiki page is the more actionable part of this proposal.
But it's not worth writing a proposal *just* for that when anyone can create a wiki page.
I (@Ericson2314) am therefore more interested in the preliminary consensus part, nebulous as it may be.
[See `#236`_, the first example of a such a "big picture" proposal, for comparison.]

Proposed Change Specification
-----------------------------

A wiki page will be kept with the following rough plan and each item's progress.
Additionally, acceptance of this proposal indicates a rough agreement that these rather large refactors are worth pursuing.
The MRs and other proposals are still reviewed and approved individually, and the plan (and its wiki page) will change accordingly.

The plan is broken into topics, with individual proposals and MRs, along the dependencies with each topic, included as sublists.

Multi-target GHC
~~~~~~~~~~~~~~~~

GHC should allow the platform its compiling for to be configured at run time.
This is here because it unblocks other topics; not trying to smuggle features among tech debt.

* [X] No more target info baked in ``Config.hs`` (it's now in ``settings``).

* [X] No more ``*_TARGET_*`` macros.

* [ ] Remove ``#include "MachDeps.h"`` from the compiler proper.

  * Most difficult to do with the primops.
    See `GHC #16964`_ for details.

* [ ] Extend CLI to make overriding the target easier than changing ``settings``.

Build stage 1 GHC with plain Cabal
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

A large amount of the complexity that Make and Hadrian deal with can and should be gotten rid of.
Building stage 1 like a normal project would make GHC development far more accessible, and the GHC API easier to use from other projects, especially when a temporary fork is needed.
Hadrian is still needed for multi-stage bootstrapping, but long term perhaps could disentangle itself from GHC entirely, and rebrand itself as a Cabal reimplementation.
Lastly, a pure Cabal, if not entirely automatic, build process could potentially speed up the deprecation of ``make``.

* [ ] Convert code-gen programs to TH

  * [ ] Get TH working in GHC without complicating bootstrap requirements

    * [ ] Requires stage hygiene for TH (`#243`_) to not break cross compiled GHC.

      * [ ] Requires multi-package GHC (`#243`_) to pun stages as separate modules for hygiene.

    * [ ] Requires naive core interpreter (`#162`_) to deal with ABI changes from stage0.

* [ ] Push ``configure.ac`` deeper.
  Individual components care about configuring different things;
  the compiler itself shouldn't need much configuration at all with most choices punted to run time.
  Cabal projects support a ``build-type: configure`` which makes integration relatively seamless.
  Make and Hadrian do want to ensure that different components inspecting the same thing get the same result.
  They still need an overall configure step, so we could use ``aclocal.m4`` to share code between the old and new ``configure.ac``.

Abstract over ``DynFlags``
~~~~~~~~~~~~~~~~~~~~~~~~~~

``DynFlags`` has no meaning.
It is just a roll up of every configuration option anything needs ever.
This leaks information to each logical part of the compiler, and prevents adding additional components with more configuration options.

The simplest thing to do is make purpose-specific configuration records, and ``Has*`` classes that allow projecting those records.
Then every ``DynFlags`` can be replaced with a type parameter with the relevant constraints.
``DynFlags`` would only be used by the entry point to satisfy all the constraints.



Reduce hs-boot tangle
~~~~~~~~~~~~~~~~~~~~~

`hs-boot` files have annoyed everyone that works on GHC.
Of all that could be said about them, perhaps most important is nothing much checks their growth over time.
As such, GHC tends towards a greater *un*separation of concerns.
Increasing entropy is always a problem with code maintenance, but this makes it more acute.

Secondarily, every `hs-boot` is poignantly a near-miss opportunity for modularity.
The abstract interface indicates that only certain details matter, but the knot is already tied so theirs no opportunity to plug in something different.

Examples
--------

Not much to show besides plan and links.

Effect and Interactions
-----------------------

Covered in "Proposed Change Specification" as the effects and interactions, rather than changes themselves, are precisely what's being proposed here.

Costs and Drawbacks
-------------------

- The changes that strive to get rid of the ``hs-boot`` tangle will make the types bigger (if not much fancier).
  This is in my view an unavoidable trade-off with tying the not at the module vs value level.

- Proliferation of dictionaries means GHC will probably loose performance without ``-fexpose-all-unfoldings``.
  I think this is a small price to pay for modularity though.

Alternatives
------------
List existing alternatives to your proposed change as they currently exist and
discuss why they are insufficient.


Unresolved Questions
--------------------

Every step of the way new things will come up.


Implementation Plan
-------------------

I (@Ericson2314) will port this document over to the wiki page.
The individual proposals MRs will continued to be pushed by their current authors, but I hope with this document others would feel more able to contributors.

.. _`#162`: https://github.com/ghc-proposals/ghc-proposals/issues/162
.. _`#236`: https://github.com/ghc-proposals/ghc-proposals/pull/236
.. _`#243`: https://github.com/ghc-proposals/ghc-proposals/pull/243
.. _`#263`: https://github.com/ghc-proposals/ghc-proposals/pull/263

.. _`GHC #16964`: https://gitlab.haskell.org/ghc/ghc/issues/16964

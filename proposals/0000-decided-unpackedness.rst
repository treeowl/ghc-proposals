Add true unpackedness to Generic metadata
=========================================

.. proposal-number:: Leave blank. This will be filled in when the proposal is
                     accepted.
.. trac-ticket:: Leave blank. This will eventually be filled with the Trac
                 ticket number which will track the progress of the
                 implementation of the feature.
.. implemented:: Leave blank. This will be filled in with the first GHC version which
                 implements the described feature.
.. highlight:: haskell
.. header:: This proposal is `discussed at this pull request <https://github.com/ghc-proposals/ghc-proposals/pull/150>`_.
.. sectnum::
.. contents::

The ``Generic`` representation of a type gives a lot of useful information
about its structure. Unfortunately, it's missing an operationally critical
piece: whether each field is unpacked.


Motivation
------------
My `proposal to replace atomicModifyMutVar# <https://github.com/ghc-proposals/ghc-proposals/pull/149>`_
provides a primitive atomic operation that works for any record whose first
field is lifted. For example, it works for ``(a, b)``, or ``(a,b,c)``, or
``data Foo = Foo Int Char# Int#``. We can express this using generics: ::

 type family Leftmost (a :: Type -> Type) :: Type where
   Leftmost (M1 i ('MetaData _ _ _ 'True) f) = Leftmost' f
   Leftmost (M1 i ('MetaSel _ _ _ 'DecidedUnpack) f) = Leftmost' f
   Leftmost (M1 i c f) = Leftmost f
   Leftmost (f :*: g) = Leftmost f
   Leftmost (K1 i c) = c
 
   Leftmost (f :+: g) = TypeError ('Text "Sum types cannot be used with atomicModifyIORefG")
   Leftmost U1 = TypeError ('Text "atomicModifyIORefG expects a record with at least one field")
   Leftmost V1 = TypeError ('Text "atomicModifyIORefG expects a record with at least one field")
 
 -- Dig through newtypes and unpacked things
 type family Leftmost' (a :: Type -> Type) :: Type where
   Leftmost' (M1 i c f) = Leftmost' f
   Leftmost' (K1 i c) = Leftmost (Rep c)

 atomicModifyIORefG :: a ~ Leftmost (Rep r) => IORef a -> (a -> r) -> IO (a, r)
 atomicModifyIORefG (IORef (STRef ref)) f = IO $ \s ->
   case atomicModifyMutVar2# ref f s of
     (# s', old, new #) -> (# s', (old, new) #)

This is safe as long as the ``Generic`` instances are derived or otherwise
legitimate.

Proposed Change Specification
-----------------------------

Replace the ``SourceUnpackedness`` field with a ``DecidedUnpackedness`` one.
The source unpackedness information isn't worth much anyway, and almost nobody
uses it, so I think breakage would be minimal.

Effect and Interactions
-----------------------
Detail how the proposed change addresses the original problem raised in the motivation.

Discuss possibly contentious interactions with existing language or compiler features. 


Costs and Drawbacks
-------------------
Give an estimate on development and maintenance costs. List how this effects learnability of the language for novice users. Define and list any remaining drawbacks that cannot be resolved.


Alternatives
------------

We could add a separate ``DecidedUnpackedness`` field to the ``MetaSel``
constructor. This is arguably the cleanest way, but it would break all code
that looks at ``MetaSel``. Ouch!


Unresolved questions
--------------------
Explicitly list any remaining issues that remain in the conceptual design and specification. Be upfront and trust that the community will help. Please do not list *implementation* issues.

Hopefully this section will be empty by the time the proposal is brought to the steering committee.


Implementation Plan
-------------------
(Optional) If accepted who will implement the change? Which other ressources and prerequisites are required for implementation?

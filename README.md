# Haskell Notes

> Someone told me that you don't just learn Haskell in the usual way (by
> programming), you *study* Haskell ... &mdash;[The Beauty of Haskell][1]

## Types and Kinds

Every value has a *type*, and every type has a *kind*. Kinds are the "types of
types". A type that can serve as the type of values has kind `*`.

```
λ> let x = 42 :: Int
λ> :type x
x :: Int
λ> :kind Int
Int :: *
```

*Type constructors* are functions from types to types.
`Maybe` and `[]` (List) have kind `* -> *`: they take a type of kind `*` as
an argument and construct a new type of kind `*`.

```
λ> :k Maybe
Maybe :: * -> *
λ> :k Maybe Int
Maybe Int :: *
λ> :k []
[] :: * -> *
λ> :k [Int]
[Int] :: *
```

Multi-parameter type constructors can be partially applied, just like multi-parameter
functions. `Either`, for example, which has kind `* -> * -> *`, returns a type
constructor of kind `* -> *` when applied to a type argument of kind `*`.

```
λ> :k Either
Either :: * -> * -> *
λ> :k Either String
Either String :: * -> *
λ> :k Either String Int
Either String Int :: *
```

What happens when `Either` is applied to `Maybe`?

```
λ> :k Either Maybe

<interactive>:1:8:
    Expecting one more argument to ‘Maybe’
    The first argument of ‘Either’ should have kind ‘*’,
      but ‘Maybe’ has kind ‘* -> *’
    In a type in a GHCi command: Either Maybe
```

[Type Constructors and Kinds][2]:
> Just like values are distinguished by their types, types are distinguished by
> their kinds, and just like ill-typed values result in type errors, ill-kinded
> types result in kind errors.

The kind of `Either` is `* -> (* -> *)`, not `(* -> *) -> *`. A type
constructor of the latter kind is a *higher-order* type constructor, one that
expects a type constructor as an argument.

```
λ> :k Functor
Functor :: (* -> *) -> Constraint
λ> :k Functor Maybe
Functor Maybe :: Constraint
```











<!--References-->

[1]: http://jabberwocky.eu/2014/04/25/the-beauty-of-haskell
[2]: https://leanpub.com/purescript/read#leanpub-auto-type-constructors-and-kinds

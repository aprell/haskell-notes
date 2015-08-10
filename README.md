# Haskell Notes

> Someone told me that you don't just learn Haskell in the usual way (by
> programming), you *study* Haskell ... &mdash;[The Beauty of Haskell][1]

## Types and Kinds

Every value has a *type*, and every type has a *kind*. Kinds are the "types of
types".

```haskell
λ> let x = 42 :: Int
λ> :type x
x :: Int
λ> :kind Int
Int :: *
```

[Type Constructors and Kinds][2]:
> Just like values are distinguished by their types, types are distinguished by
> their kinds, and just like ill-typed values result in type errors, ill-kinded
> types result in kind errors.

Every type that can serve as the type of values has kind `*`.

```haskell
λ> :k Maybe
Maybe :: * -> *
λ> :k Maybe Int
Maybe Int :: *
λ> :k []
[] :: * -> *
λ> :k [Int]
[Int] :: *
```

`Maybe` is a *type constructor*: a function that takes a type of kind `*` as
an argument and returns another type of kind `*`. List (`[]`) is another
example of a type constructor: it takes a type `a` and constructs a new type
`[a]`.

```haskell
λ> :k Functor
Functor :: (* -> *) -> Constraint
λ> :k Functor Maybe
Functor Maybe :: Constraint
```

`Functor` is a *higher-order* type constructor.










<!--References-->

[1]: http://jabberwocky.eu/2014/04/25/the-beauty-of-haskell
[2]: https://leanpub.com/purescript/read#leanpub-auto-type-constructors-and-kinds

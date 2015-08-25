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

Multi-parameter type constructors can be partially applied, just like
multi-parameter functions. `Either`, for example, which has kind `* -> * ->
*`, returns a type constructor when applied to a type.

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

Note that `Functor Maybe` has kind `Constraint`, which implies that it cannot
be used as a value type. Many (unary) type constructors are in fact instances
of `Functor`.

```
λ> :info Functor
class Functor (f :: * -> *) where
  fmap :: (a -> b) -> f a -> f b
  (GHC.Base.<$) :: a -> f b -> f a
    -- Defined in ‘GHC.Base’
instance Functor Maybe -- Defined in ‘Data.Maybe’
instance Functor (Either a) -- Defined in ‘Data.Either’
instance Functor [] -- Defined in ‘GHC.Base’
instance Functor IO -- Defined in ‘GHC.Base’
instance Functor ((->) r) -- Defined in ‘GHC.Base’
instance Functor ((,) a) -- Defined in ‘GHC.Base’
```

Wait, `(->) r` is a functor? What does it mean to map a function over a
function? Recall the signature of `fmap`.

```
λ> :t fmap
fmap :: Functor f => (a -> b) -> f a -> f b
```

Specializing `f` to `(->) r` gives `(a -> b) -> (->) r a -> (->) r b`, which
is equivalent to `(a -> b) -> (r -> a) -> (r -> b)` in infix form. This
signature looks familiar.

```
λ> :t (.)
(.) :: (b -> c) -> (a -> b) -> a -> c
```

In fact, the `Functor` instance for `(->) r` is simply:

```haskell
instance Functor ((->) r) where
	fmap = (.)
```

```
λ> let f = fmap (*3) (+2)
λ> :t f
f :: Num a => a -> a
λ> f 1
9
λ> let g = (*3) . (+2)
λ> g 1
9
```

Let us repeat the exercise with the pair functor. Substituting `(,) c` for `f`
gives `(a -> b) -> (,) c a -> (,) c b`, which is equivalent to `(a -> b) ->
(c,a) -> (c,b)` in infix form. We see that `fmap` leaves the first element of
a pair unchanged.

```
λ> fmap (+1) (2,3)
(2,4)
```

If we want `fmap` to behave differently, we can write a `newtype` wrapper and
make it an instance of `Functor`.

```haskell
newtype Pair a b = Pair { fromPair :: (b, a) }

instance Functor (Pair a) where
    fmap f (Pair (b, a)) = Pair (f b, a)
```

```
λ> fromPair . fmap (+1) $ Pair (2,3)
(3,3)
```

Every proper instance of `Functor` should satisfy the [functor laws][3]. The
second functor law requires that `fmap (f . g) == fmap f . fmap g`.

```
λ> ((*3) . (+2)) `fmap` Just 1
Just 9
λ> (*3) `fmap` (+2) `fmap` Just 1
Just 9
```

There is actually an infix synonym for `fmap`.

```
λ> :i (<$>)
(<$>) :: Functor f => (a -> b) -> f a -> f b
    -- Defined in ‘Data.Functor’
infixl 4 <$>
```

Think function application (`$`) with context (`< >`).

```
λ> (*3) $ (+2) $ 1
9
λ> (*3) <$> (+2) <$> Just 1
Just 9
```




<!--References-->

[1]: http://jabberwocky.eu/2014/04/25/the-beauty-of-haskell
[2]: https://leanpub.com/purescript/read#leanpub-auto-type-constructors-and-kinds
[3]: https://hackage.haskell.org/package/base-4.8.1.0/docs/Data-Functor.html

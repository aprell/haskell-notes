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
λ> :m +Test.QuickCheck
λ> let f = (*3)
λ> let g = (+2)
λ> let prop x = fmap (f . g) x == (fmap f . fmap g) x
λ> quickCheck (prop :: Maybe Int -> Bool)
+++ OK, passed 100 tests.
```

So far we have lifted only single-argument functions. Given a function of type
`a -> b`, `fmap` returns a lifted version of the function that takes a value
of type `f a` and returns a value of type `f b`. Now suppose we want to lift a
function of type `a -> (b -> c)` (parentheses added for clarity). What we get
is a function of type `f a -> f (b -> c)`, one that takes a value of type `f
a` and returns a value of type `f (b -> c)`. The lifted function's result is a
single-argument function inside a functor. Guess what happens if we try to
lift a function of type `a -> (b -> c -> d)`. How do we apply functions of
type `f (b -> c)` or `f (b -> c -> d)`? Enter `Applicative`.

```
λ> :i Applicative
class Functor f => Applicative (f :: * -> *) where
  pure :: a -> f a
  (<*>) :: f (a -> b) -> f a -> f b
  (*>) :: f a -> f b -> f b
  (<*) :: f a -> f b -> f a
    -- Defined in ‘Control.Applicative’
instance Applicative [] -- Defined in ‘Control.Applicative’
instance Applicative ZipList -- Defined in ‘Control.Applicative’
instance Monad m => Applicative (WrappedMonad m)
  -- Defined in ‘Control.Applicative’
instance Control.Arrow.Arrow a => Applicative (WrappedArrow a b)
  -- Defined in ‘Control.Applicative’
instance Applicative Maybe -- Defined in ‘Control.Applicative’
instance Applicative IO -- Defined in ‘Control.Applicative’
instance Applicative (Either e) -- Defined in ‘Control.Applicative’
instance Data.Monoid.Monoid m => Applicative (Const m)
  -- Defined in ‘Control.Applicative’
instance Applicative ((->) a) -- Defined in ‘Control.Applicative’
instance Data.Monoid.Monoid a => Applicative ((,) a)
  -- Defined in ‘Control.Applicative’
```

Suppose we want to add two values of type `Maybe a`, say `Just 1` and `Just
2`. We can start with `fmap (+) $ Just 1`, which produces `Just (+1) :: Num a
=> Maybe (a -> a)`. What we need to complete the operation is a function of type
`Maybe (a -> b) -> Maybe a -> Maybe b` that applies the `(+1)` wrapped
in `Just` to `Just 2`. Something like this:

```haskell
Just f `apply` x = fmap f x
Nothing `apply` _ = Nothing
```

```
λ> (+) `fmap` Just 1 `apply` Just 2
Just 3
```

The function we are looking for is the application operator `<*>` from
`Applicative`. Every instance of `Applicative`, including `Maybe`, must
implement `<*>`.

```
λ> (+) `fmap` Just 1 <*> Just 2
Just 3
```

There is actually an infix synonym for `fmap`, which lets us write:

```
λ> (+) <$> Just 1 <*> Just 2
Just 3
```

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

Instead of using `fmap`, we can use `pure` to lift values.

```
λ> :t pure (+)
pure (+) :: (Num a, Applicative f) => f (a -> a -> a)
λ> :t pure (+) <*> Just 1
pure (+) <*> Just 1 :: Num a => Maybe (a -> a)
λ> :t pure (+) <*> Just 1 <*> Just 2
pure (+) <*> Just 1 <*> Just 2 :: Num b => Maybe b
λ> pure (+) <*> Just 1 <*> Just 2
Just 3
```

Notice how `(+) <$> Just 1` and `pure (+) <*> Just 1` produce the same result.
In fact, we expect that `f <$> x == pure f <*> x`.

```
λ> :m +Test.QuickCheck
λ> let f = (+1)
λ> let prop x = (f <$> x) == (pure f <*> x)
λ> quickCheck (prop :: Maybe Int -> Bool)
+++ OK, passed 100 tests.
```

[Typeclassopedia][4]:
> In other words, we can decompose `fmap` into two more atomic operations:
> injection into a context, and application within a context.

Also, notice how `f <$> x <*> y` or `pure f <*> x <*> y` resemble `f x y`. The
use of operators can be hidden behind utility functions.

```
λ> liftA2 (+) (Just 1) (Just 2)
Just 3
```









<!--References-->

[1]: http://jabberwocky.eu/2014/04/25/the-beauty-of-haskell
[2]: https://leanpub.com/purescript/read#leanpub-auto-type-constructors-and-kinds
[3]: https://hackage.haskell.org/package/base-4.8.1.0/docs/Data-Functor.html
[4]: https://wiki.haskell.org/Typeclassopedia#Applicative

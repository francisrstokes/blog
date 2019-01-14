---
layout: post
title: "Exploring The Peano Axioms With Algebraic Data Types"
date: 2019-01-12 09:00:00 +0100
comments: true
categories: mathematics peano javascript functional-programming adt algebraic
---

The *Peano Axioms* are <a href="https://en.wikipedia.org/wiki/Peano_axioms" _target="blank">8 simple rules</a> that can define the natural number system. What are the natural numbers? They're are just the numbers *0, 1, 2, 3, ...* and so on, up to infinity.

This system is interesting because in mathematics, you need to be rigorous; You can't go around saying numbers (or any mathematical object) *just are*, you have be able to really describe what you mean when you talk about *zeroness* or *oneness*.

## The Axioms

The Peano Axioms describe a *system* with two elements: An *object* `0`, and a *function* `S`. I'm using these words object and function quite loosely. As you will see below, `0` could actually be any symbol you choose, and the function `S` (known as the successor function) also essentially acts as just a symbol - that is to say, it is not necessary to define `S` function further than what is shown below.

These are the axioms as stated on wikipedia:

1. `0` is a natural number
2. For every natural number `x`, `x = x` (reflexivity)
3. For all natural numbers `x` and `y`, if `x = y`, then `y = x` (symmetry)
4. For all natural numbers `x`, `y` and `z`, if `x = y` and `y = z`, then `x = z` (transitivity)
5. For all `a` and `b`, if `b` is a natural number and `a = b`, then `a` is also a natural number (closure under equality)
6. For every natural number `n`, `S(n)` is a natural number
7. For all natural numbers `m` and `n`, `m = n` **if and only if** `S(m) = S(n)` (injection)
8. For every natural number `n`, `S(n) = 0` is **false**. That is, there is no natural number whose successor is `0`

What these axioms state is that we have an initial number `0`, which is not the successor of any other number (i.e. it is the smallest), and that we can create new numbers by applying a successor function to any number. So there is a successor of `0` called `S(0)`, and a successor to that called `S(S(0))`, and so on.

Practically speaking, you can read a Peano encoded natural number as simply being the number of `S`s it has. So `S(0)` is `1`, `S(S(0))` is `2`, and so on.

Here's a table of the first few Peano numbers



| *N* | *Peano* |
|:--------:|:--------------:|
| 0      | `0`            |
| 1      | `S(0)`         |
| 2      | `S(S(0))`      |
| 3      | `S(S(S(0)))`   |
| 4      | `S(S(S(S(0))))`|


## Algebraic Data Types

We can represent this system in *types* by utilising something called an <a href="https://en.wikipedia.org/wiki/Algebraic_data_type" target="_blank">Algebraic Data Type</a>. An *Algebraic type* is either:

- A **product** type, like a <a href="https://en.wikipedia.org/wiki/Record_(computer_science)" target="_blank">record/object/struct</a> or a <a href="https://en.wikipedia.org/wiki/Tuple" target="_blank">tuple</a>
- A **sum** type, which is a *union* of other types

For this article we will focus on the **sum type**. There are many common sum types that we use every day. `Boolean` is a sum type of the types `True` and `False`. In Haskell it might be defined as


```haskell
data Boolean = True | False
```

In JavaScript, we can use a library like `daggy` or `algebraic-types` to create a sum type. For these examples I will use `algebraic-types` because it provides a mechanism to properly define recursive types.

```javascript
const {createUnionType} = require('algebraic-types');

const Boolean = createUnionType('Boolean', {
  True: [],
  False: []
});
```

`True` and `False` are both types that belong to the type `Boolean`. This is a simple but powerful concept, because it implies that meaning can be *encoded* directly on the type level. To illustrate this further, let's take a breif look at a slightly more complex sum type, `Maybe`, which encodes possible nothingness.

In Haskell:

```haskell
data Maybe a = Just a | Nothing
```

In JavaScript:

```javascript
const {createUnionType, variable} = require('algebraic-types');

const Maybe = createUnionType('Maybe', {
  Nothing: [],
  Just: [variable('a')]
});
```

This should be read *"A `Maybe` of 'a' is either `Just` 'a' or it is `Nothing`"*, where 'a' can be *any* value. You never need to write a function or method that returns `null` to indicate no value because you can do it on the type level.

One more thing that we can do with sum types is define *recursive* structures. A binary tree is laughably trival to implement as a sum type:

In Haskell:

```haskell
data BinaryTree a = Leaf a | Branch BinaryTree BinaryTree;
```

In JavaScript:

```javascript
const {createUnionType, variable, recursive} = require('algebraic-types');

const BinaryTree = createUnionType('BinaryTree', {
  Leaf: [variable('a')],
  Branch: [recursive('left'), recursive('right')]
});
```

Which is to say a `BinaryTree` is either `Leaf` of some value 'a', or it's a `Branch` of two more `BinaryTree`s (both of which can be either a `Leaf` or a `Branch`).

## Back to Peano

Now we have a basic understanding of sum types, we can apply the concepts of *encoding meaning* and *recursive definition* to creating a type for Peano Numbers. I will use JavaScript for the rest of the examples, but the translation to Haskell is trival.

```javascript
const Peano = createUnionType('Peano', {
  O: [],
  S: [recursive('x')]
});
```

The `Peano` type is made of two types: `O`, which has no variables of its own, and `S`, which has one variable that must be another `Peano` type. The recursive definition ensures that we can only have *axiomatically* correct constructions:

```javascript
const {S, O} = Peano;

// All the following are valid
const zero = O();
const one = S(zero);
const two = S(one);
const three = S(S(S(O())));

// But this is not
const invalid = S("hello");
// -> Error: Variable x on constructor S must be a Peano
```

So just by creating this type, we have the first axiom down. The next three axioms are related to the properties of *equality* in this system. We can encode them by adding a method `eq` to prototype of the `Peano` constructor. Since methods are defined on the `Peano` type rather than the on the *child types* `S` or `O`, there must be a mechanism to check which type the `Peano` actually is inside a method. `algebraic-types` adds a method called `match` to every type you create with it, which can perform this operation (called pattern matching). We can use this to build a recursive definition for `eq`.

```javascript
Peano.prototype.eq = function (y) {
  const x = this;

  return x.match({
    O: () => y.match({
      O: () => true,
      S: () => false
    }),
    S: innerX => y.match({
      O: () => false,
      S: innerY => innerX.eq(innerY)
    })
  });
}

const zero = O();
const one = S(zero);

const x = S(one);
const y = S(S(zero));
const z = S(S(O()));

// Reflexivity
x.eq(x); // -> true

// Symmetry
x.eq(y); // -> true
y.eq(x); // -> true

// Transitivity
x.eq(y); // -> true
y.eq(z); // -> true
x.eq(z); // -> true
```

The rest of the axioms fall out of either the type defintion itself, or the implementation of equality.

Let's break down the recursive definition for equality a little bit, as the same principle will be used to define more operations like addition and multiplication.

To compare if two `Peano`s, `x` and `y` are equal, we simply have to look at their types with *pattern-matching*. If `x` is a `O` and `y` is an `S` of *anything*, then they cannot be equal, but if `y` is also a `O` then we can say that they *are* equal. If both `x` and `y` are an `S` of something, then we must recursively call `eq` on their inner values. If they are indeed equal, this process will continue until both inner values reach `O`. If they are not equal, then one of the values will reach `O` while the other is an `S` of something, which we have already defined as non-equal.

## More operations

### Addition

Even though we can construct `Peano` numbers, and check if they are equal to each other, this can be described as neither particularly useful nor particularly interesting. What would be interesting is the ability to add one `Peano` to another, yielding a new `Peano`.

```javascript
Peano.prototype.add = function (y) {
  const x = this;

  return x.match({
    O: () => y,
    S: innerX => S(y).add(innerX)
  });
};

// 2 + 3
S(S(O())).add(S(S(S(O()))));
// -> S(S(S(S(S(O())))))
```

Adding `x` and `y` is another recursive operation. Since for any number we know that `y + 0 = y`, *pattern-matching* a `O` value for `x` means that we can just return the other value (even if that value was a zero itself). If the value of `x` is an `S` however, and we increase the other `y` by wrapping it in an `S`, and add that to the inner value of `x`, then the effect is to add one to `y` and minus one from `x`. `x` will eventually match `O` and the final value of `y` will be returned.

This is kind of like have two piles of blocks that you want to add. If you just move one block at a time from the first pile into the second pile until there are no blocks left in the first pile, then the second pile will *of course* contain the number of blocks in the second pile plus the number of blocks in the first.

### Subtraction

A very similar trick can be used for subtraction, but first it's good to mention that subtraction in the Peano Axioms is not really *well defined*, since there are no negative numbers (axiom #8). For this reason we will say that an expression like `4 - 5` in our world will simply equal `0`, rather than `-1`.

Going back to the idea of two piles, if we remove one block from each pile until one of the piles is empty, then whatever is left on the other pile is the result of `(Pile #1) - (Pile #2)`. You can try it for yourself if you need some convincing.

```javascript
Peano.prototype.sub = function (y) {
  const x = this;

  return x.match({
    O: () => y,
    S: innerX => y.match({
      O: => S(innerX),
      S: => innerX.sub(y)
    })
  });
};

// 3 - 2
S(S(S(O()))).sub(S(S(O())));
// -> S(O())
```

Notice that we need to put `x` back into an `S` if `y` is a `O`, because otherwise we would be off by one.

### Multiplication

Multiplication is just glorified addition, so we should be able to build it onto of the existing `add` method.

```javascript
Peano.prototype.mul = function (y) {
  const x = this;

  return x.match({
    O: () => O(),
    S: innerX => y.match({
      O: () => O(),
      S: innerY => S(innerX).add(
        innerY.mul(S(innerX))
      )
    })
  });
};
```

Anything times `0` is always `0`, so in both those cases are easy. To multiply two numbers larger than zero we just keep adding `x`s while `y` is greater than `0`, unwrapping a level of `y` every time.

## Converting to and from Integers

It could be useful (as far as this system is useful in a practical sense) to be able to convert `Peano` numbers to regular `Number`s, and to convert positive integers to `Peano` numbers. We can write a `toInteger` method like:

```javascript
Peano.prototype.toInteger = function () {
  const x = this;
  return x.match({
    O: () => 0,
    S: innerX => 1 + innerX.toInteger()
  });
};

S(S(S(S(O())))).toInteger();
// -> 4
```

Which is basically just a recursive operation that unfolds to something like `1 + 1 + 1 + 1 + ... + 0`.

`fromInteger` can be a static method on the `Peano` object which works in the other direction, by recursively wrapping `n` layers of `S` around an `O`.

```javascript
Peano.fromInteger = (n) => {
  const f = (total, n) => {
    if (n > 0) {
      return f(S(total), n - 1);
    }
    return total;
  };

  return f(O(), n);
};

Peano.fromInt(5);
// -> S(S(S(S(S(O())))))
```

This has the nice effect of not blowing up if we throw in a negative number - we'd simply get `0` in those cases.

Now we can perform *advanced* calculations like

```javascript
const three = S(S(S(O())));

Peano
  .fromInteger(50)
  .mul(three)
  .sub(S(three))
  .add(O())
  .toInteger();
// -> 146
```

## Conclusion

The Peano Axioms have fascinated me ever since I learned about them because they are a digestible piece of mathematics that is somehow apart from the rote concepts you learn while making your way through the education system. The system shines a subtle light on the real underpinnings of mathematics; the rigor of defining the properties of a mathematical object. At school you might be tricked into thinking that numbers *just are*, but really they are just one of many mathematical objects with different kinds of properties. If you're like me, and perhaps never knew about this side of math, you can check out the field of <a href="https://en.wikipedia.org/wiki/Abstract_algebra" target="_blank">Abstract Algebra</a>, that classifies objects and systems of this kind based on the properties they have.

But from a practical development perspective, as a number system, this isn't particularly useful in everyday life. That said, you might find some real value in *modeling with types*. It's a really powerful concept when applied correctly, especially when combined with functions that operate on types. You can discern a lot about a program by just looking at the *signatures* of such functions - what types do they take as parameters and what types do they return. If you've read this article coming from the JavaScript world, I advise you to checkout Haskell (especially if you've enjoyed the benefits of TypeScript). It takes programming with types to a fine art.

<a href="https://twitter.com/fstokesman" target="_blank">If you've found this article interesting, I'd really love to hear from you about it! Send me tweet @fstokesman</a>

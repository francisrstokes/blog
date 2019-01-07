---
layout: post
title: "Infinite Data Structures In JavaScript"
date: 2019-01-07 11:39:22 +0100
comments: true
categories: javascript haskell functional-programming
---

_Languages like Haskell are able to deal directly with Infinite Lists, but can that ability be brought to the world of JavaScript?_

------

{% img center /images/inf-structure.png 600 600 'image' 'images' %}

In the real world we deal with “infinite” ideas all the time. All the positive numbers is an example of an infinite concept we wouldn’t bat an eye at. But typically when we are programming we have to think of these infinite concepts in a finite way. You can’t have an Array of all the positive numbers (at least not in JavaScript!).

What I want to introduce in this article is the idea of an **Infinite List** data structure, which can represent some never ending sequence, and let’s us use common operations like ***map*** and ***filter*** to modify and create new sequences.

```javascript
const allPositiveNumbers = /* somehow we can create this */;

// We can create new Infinite structures using map and filter
const allEvenNumbers = allPositiveNumbers.filter(x => x % 2 === 0);
const allOddNumbers = allEvenNumbers.map(x => x - 1);
```

When we want to actually *use the data*, we can just ***take*** some concrete amount of it.

```javascript
allPositiveNumbers.take(10);
// -> [ 1, 2, 3, 4, 5, 6, 7, 8, 9, 10 ]

allEvenNumbers.take(10);
// -> [ 2, 4, 6, 8, 10, 12, 14, 16, 18, 20 ]

allOddNumbers.take(10);
// -> [ 1, 3, 5, 7, 9, 11, 13, 15, 17, 19 ]
```

# Creating an Infinite List

To create the kind of structure defined above, we first need some way to just *describe* an infinite thing without needing to actually evaluate it (which would kill the computer *really fast*). To do that we first need to understand 2 things:

- The *iterator pattern*
- *Generator* functions

## The iterator pattern

Let’s start with the *iterator pattern* first. The *iterator pattern* is a kind of design pattern where you can produce a lot of values, one at a time. It’s basically just an object with a ***.next()*** method. When that method is called, it returns another object with 2 keys: ***value*** and ***done***. As we call ***.next()***, the ***done*** property indicates whether the iterator has more values to give us, and the ***value*** is of course the value. Below is a simple iterator that returns values 0 to 3:

```javascript
const createIterator = () => {
  let x = 0;
  return {
    next: () => {
      if (x > 3) {
        return { value: undefined, done: true };
      } else {
        const currentValue = x;
        x += 1;
        return { value: currentValue, done: false }; 
      }
    }
  }
};

const iterator = createIterator();
iterator.next();
// -> { value: 0, done: false }

iterator.next();
// -> { value: 1, done: false }

iterator.next();
// -> { value: 2, done: false }

iterator.next();
// -> { value: 3, done: false }

iterator.next();
// -> { value: undefined, done: true }
```

This kind of pattern has a *first-class* place in JavaScript. ***Arrays, Objects, Maps, and Sets*** can all implement the iterator pattern by conforming to an interface called [**Iterable**](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Guide/Iterators_and_Generators#Iterables).

## Generator functions

Generator functions are a special kind of function in JavaScript which is declared with the __function* syntax__. Generator functions are used to create *iterator* objects (ones with a ***.next()*** method), but in much clearer and more concise way. Below is a finite generator that creates an equivalent iterator:

```javascript
const createIterator = function* () {
  let x = 0;
  while (x < 4) {
    yield x;
    x += 1;
  }
};
```

Here, the ***yield*** keyword is used to indicate the value that is returned every time the iterator is called. You can think of the function pausing after every yield, and picking up where it left off when the iterator’s ***.next()*** method is called again.

All it would take to make this an infinite generator is to turn the ***while (x < 4)*** loop into a ***while (true)*** loop. For a better feel, here’s an infinite generator for the famous fibonacci sequence:

```javascript
const createFibSeqIterator = function* () {
  let a = 0;
  let b = 1;
  while (true) {
    yield b;
    const tmp = a;
    a = b;
    b = b + tmp;
  }
};

const fibSeq = createFibSeqIterator();

fibSeq.next() // -> { value: 1, done: false }
fibSeq.next() // -> { value: 1, done: false }
fibSeq.next() // -> { value: 2, done: false }
fibSeq.next() // -> { value: 3, done: false }
fibSeq.next() // -> { value: 5, done: false }
```

## Putting it together

Iterators seem like a representation of something infinite because it lets us ask for the ***.next()*** element indefinitely. Furthermore, a generator seems like a good way to specify the iterator, because we can write succinct algorithms for infinite series without the boilerplate of manually crafting iterators.

But this still isn’t enough, because as powerful as generators are, they’re not really compositional. If I wanted to to create a generator where I filtered to all the fibonacci numbers that ended with a 5, I can’t easily use my existing createFibSeqIterator() to do that — i.e. I can’t compose the idea of the original generator + some new operation on it.

To remedy this, we first need to encapsulate the generator into a data type, which we can do with a class:

```javascript
class Infinite {
  constructor (generatorFn) {
    this.generator = generatorFn;
  }
}
```

It’s on this class that we will implement our operations like ***filter***, ***map***, and ***take***.

## Infinitely Lazy

You might be puzzled when thinking about how we could implement an operation such as ***filter***. The answer is *simple*: we do it lazily. Instead of actually trying to filter our list, we just make a note inside the ***Infinite*** class. Then when the user wants concretely ***.take()*** some elements of it, we can do the actual business of filtering then.

```javascript
class Infinite {
  constructor (generatorFn) {
    this.generator = generatorFn;
    this.transformations = [];
  }

  filter(filterFn) {
    const newInfinite = new Infinite(this.generator);
    newInfinite.transformations = this.transformations.slice();
    newInfinite.transformations.push({
      type: 'filter',
      fn: filterFn
    });

    return newInfinite;
  }
}
```

The ***Infinite*** class gets a new ***transformations*** property, and ***filter*** creates a new Infinite with the same generator and transformations array, and pushes a transformation into the list.

We now have all the components we need now to write a ***.take()*** that will make the list concrete.

```javascript
class Infinite {
  constructor (generatorFn) {
    this.generator = generatorFn;
    this.transformations = [];
  }

  filter(filterFn) {
    const newInfinite = new Infinite(this.generator);
    newInfinite.transformations = this.transformations.slice();
    newInfinite.transformations.push({
      type: 'filter',
      fn: filterFn
    });

    return newInfinite;
  }

  take(n) {
    const iterator = this.generator();
    const concrete = new Array(n);
    let index = 0;

    while (index < n) {
      const {value, done} = iterator.next();

      if (done) {
        // The generator wasn't infinite, return what we got up until this point
        return concrete;
      }

      // Loop over the transformations and apply them to the value
      let x = value;
      let filtered = false;
      for (let i = 0; i < this.transformations.length; i++) {
        const T = this.transformations[i];

        if (T.type === 'filter') {
          if (!T.fn(x)) {
            filtered = true;
          }
        }
      }

      // After the transformations are complete, if we didn't filter, then we can
      // x to the concrete list
      if (!filtered) {
        concrete[index] = x;
        index++;
      }
    }

    return concrete;
  }
}
```

When we call ***.take(n)***, we can create an iterator out of the generator, and then initialize a fixed-length array with ***n*** elements. This will be our concrete list. An index variable can be used to keep track of how many concrete values we’ve collected so far. Using a while loop, we can get a value out of the iterator, and then run our list of transformations on it. If one of those transformations is a ***filter***, and it doesn’t pass the test, we simply don’t add it to our concrete list, and repeat the loop. When we’ve collected ***n*** elements, we’re done and can return the concrete list.

Let’s see how that looks with the fibonacci example from before:

```javascript
const fibonacciSequence = new Infinite(function* () {
  let a = 0;
  let b = 1;
  while (true) {
    yield b;
    const tmp = a;
    a = b;
    b = b + tmp;
  }
});

const fibsEndingWith5 = fibonacciSequence.filter(x => {
  const str = x.toString();
  return str[str.length - 1] === '5';
});

fibonacciSequence.take(5);
// -> [ 1, 1, 2, 3, 5 ]

fibsEndingWith5.take(5);
// -> [ 5, 55, 6765, 75025, 9227465 ]
```

To implement ***map***, we could use much the same approach as with ***filter***. The map method itself just clones the current ***Infinite*** and adds a new transformation to the list. Inside take, it’s enough to just add an ***else-if*** inside the transformation loop.

## Conclusion

In this article we’ve explored the *iterator pattern* and *generator functions* in order to build a compositional and *lazily-evaluated* Infinite List data structure.

The ***Infinite*** List in this article is a bit limited however, because it lacks some operations that would make it truly useful, like a map implementation, dependent filtering, or the ability to combine Infinites together (like pairing every fibonacci number with a prime number, for example).

For these I created *lazy-infinite*, a more powerful ***Infinite List*** structure, that conforms to the [Fantasy-Land](https://github.com/fantasyland/fantasy-land) specification. [I’d love for you to take a look on github](https://github.com/francisrstokes/Lazy-Infinite-List), or to give it a try in your next project with

*npm i lazy-infinite*

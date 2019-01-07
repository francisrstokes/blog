---
layout: post
title: "Arcsecond: Parsing In JavaScript Made Easy"
date: 2019-01-07 14:20:18 +0100
comments: true
categories: javascript haskell functional-programming
---

{% img center /images/gandalf.png 600 600 'image' 'images' %}


I recently starting making a serious attempt at learning Haskell (anyone who has tried before will probably sympathise that it usually takes a couple of tries to crack it). Amongst the many cool things Haskell has to offer is an amazing parsing library that comes with the standard set of packages called [**Parsec**](https://wiki.haskell.org/Parsec)**,** which lets you describe how to parse complex grammars in what essentially looks like natural language.

Here is how a CSV parser is implemented using Parsec. Don’t worry if you don’t understand all the syntax, the point is that the whole parser is specified in just four lines.

```haskell
-- Taken from http://book.realworldhaskell.org/read/using-parsec.html

import Text.ParserCombinators.Parsec

csvFile = endBy line eol
line = sepBy cell (char ',')
cell = many (noneOf ",\n")
eol = char '\n'

parseCSV :: String -> Either ParseError [[String]]
parseCSV input = parse csvFile "(unknown)" input
```

If you want to read more checkout [http://book.realworldhaskell.org](http://book.realworldhaskell.org)

This post is not about Haskell however, [but rather a library I wrote called **arcsecond** which is based on Parsec](https://github.com/francisrstokes/arcsecond), with the goal of bringing that same expressivity to JavaScript.

### Parser combinators

**arcsecond** is a parser combinator library, in which complex parsers can be built by composing simple parsers. The simplest parsers just match specific strings or characters:

```javascript

const { digit, letter, char, str, regex } = require('arcsecond');

// digit will parse any number 0-9
// letter will parse any letter a-z or A-Z

// char and str will parse specific sequences
const getA = char ('a');
const getHello = str ('hello');

// regex will parse any given regular expression
const getLowNumbers = regex (/^[0-4]+/);
```

These can then be combined together with the _combinators_ in the library.

```javascript
const { char, str, sequenceOf, choice } = require('arcsecond');

const worldOrMarsOrVenus = choice ([
  str ('world'),
  str ('mars'),
  str ('venus')
]);

const combinedParser = sequenceOf ([
  str ('hello'),
  char (' '),
  worldOrMarsOrVenus
]);
```

Then the new parsers can be used with text:

```javascript
const { parse, char, str, sequenceOf, choice } = require('arcsecond');

// parse creates a function we can use for parsing text
const parseText = parse (combinedParser);

console.log(
  parseText ('hello world') 
)
// -> [ 'hello', ' ', 'world' ]
```

#### Combinators

The combinators are where it gets cool. In **arcsecond** a combinator is a higher order parser, which takes one or more parsers as its input and gives back a new parser that _combines_ those in some way. If you’ve used _higher order components_ in react like **_connect_**, **_withRouter_**, or **_withStyles_**, then you’re already familiar with the idea.

As shown above, **_sequenceOf_** is a combinator that will parse text using each of the parsers in order, collecting their results into an array.

**_choice_**, on the other hand, will try each of its parsers in order and use the first one that matches. The library contains more, like **_many_**, which takes a parser as an argument and matches as much as it can using that parser, collecting up the results in an array:

```javascript
const { parse, many, choice, str } = require('arcsecond');

const animal = choice ([
  str ('dog'),
  str ('whale'),
  str ('bear'),
  str ('ant')
]);

const manyAnimals = many (animal);

// parse creates a function we can use for parsing text
const parseText = parse (manyAnimals);

console.log(
  parseText ('dogantwhaledogbear') 
)
// -> [ 'dog', 'ant', 'whale', 'dog', 'bear' ]
```

You can use **_sepBy_** to create a parser that matches items separated by something matched by _another parser_.

```javascript
const { parse, sepBy, choice, str, char } = require('arcsecond');

const animal = choice ([
  str ('dog'),
  str ('whale'),
  str ('bear'),
  str ('ant')
]);

const commaSeparated = sepBy (char (','));

const commaSeparatedAnimals = commaSeparated (animal);

// parse creates a function we can use for parsing text
const parseText = parse (commaSeparatedAnimals);

console.log(
  parseText ('dog,ant,whale,dog,bear') 
)
// -> [ 'dog', 'ant', 'whale', 'dog', 'bear' ]
```

**_between_** will let you match items that occur between _two other parsers_.

```javascript
const { parse, between, sepBy, choice, str, char } = require('arcsecond');

const animal = choice ([
  str ('dog'),
  str ('whale'),
  str ('bear'),
  str ('ant')
]);

const betweenBrackets = between (char ('[')) (char (']'));
const commaSeparated = sepBy (char (','));

const commaSeparatedAnimals = commaSeparated (animal);
const arrayOfAnimals = betweenBrackets (commaSeparatedAnimals);

// parse creates a function we can use for parsing text
const parseText = parse (arrayOfAnimals);

console.log(
  parseText ('[dog,ant,whale,dog,bear]') 
)
// -> [ 'dog', 'ant', 'whale', 'dog', 'bear' ]
```

#### Curried Functions

**arcsecond** makes use of curried functions. If you don’t know what a curried function is, give my article “[Making Functional Programming Click](https://hackernoon.com/making-functional-programming-click-836d4715baf2)” a read. If you really can’t be bothered right now, open that in another tab and read this executive summary.

A curried function is one that, if it takes more than one argument, it instead returns a new function that takes the next argument. It’s easier to see with some code:

```javascript
// Normal function, just a single function taking both arguments and returning a result
function normalAdd(a, b) {
  return a + b; 
}

normalAdd(1, 2);
// -> 3


// Curried function, a function that takes the first argument and then returns
// *another function*, which takes the next argument.
function curriedAdd(a) {
  return function(b) {
    return a + b; 
  }
}

curriedAdd(1)(2);
// -> 3

// We can use curried functions to easily make new, more specific functions
const addOne = curriedAdd(1);

addOne(2);
// -> 3
```

As you can see above, **_curriedAdd_** is first called with 1. Since it then returns a function, we can go ahead and call that with 2, which finally returns the actual result.

We can use **_curriedAdd_** to create a new function by calling it with just one argument and then assigning the result to a variable. As a language, JavaScript treats functions as a first class citizen, meaning they can be passed around and assigned to variables.

This principle lies at the heart of **arcsecond,** and every function in the library is defined this way. **_sepBy_** take two parsers — first a separator parser, and second a value parser. Because it is curried, it is easy to create a more specific combinator like **_commaSeparated_** by only supplying the first argument.

> If you’re not used to it, it will probably seem strange. But a good lesson to learn as a software developer is not to have knee-jerk bad reactions to things that you don’t immediately understand. There is usually a reason, and you’ll find a lot more value by discovering that reason rather than dismissing it.

### Error Handling

If you try to parse a string which is not correctly formatted you would expect to get some kind of error message. **arcsecond** uses a special data type called an **_Either_**, which is _either_ a value or an error. It’s like a **_Promise_**, which can be either Rejected or Resolved, but without the implication of being asynchronous. In an **_Either_** however, the “resolved” type is called a **_Right_**, and the “rejected” type is called a **_Left_**.

The return type of **_parse_** is an Either. You can get at the value or error like this:

```javascript
const { parse, sequenceOf, str, char } = require('arcsecond');

const parser = sequenceOf ([
  str ('hello'),
  char (' '),
  str ('world')
]);

const parseResult = parse(parser)('hello world');

// We can use the cata method, which is basically a switch/case but for types
parseResult.cata({
  Left: err => {
    // do something here with the error
  },
  Right: result => {
    // result -> [ 'hello', ' ', 'world' ] 
  }
});
```


However this might not fit well into your codebase if you don’t have more functional code. For that reason there are two other options. The first is to convert the **_Either_** into a **_Promise:_**

```javascript
const { parse, sequenceOf, str, char, toPromise } = require('arcsecond');

const parser = sequenceOf ([
  str ('hello'),
  char (' '),
  str ('world')
]);

const parseResult = parse(parser)('hello world');

toPromise(parseResult)
  .catch(err => {
    // do something here with the error
  })
  .then(result => {
    // result -> [ 'hello', ' ', 'world' ] 
  });
```

Or you can use **_toValue,_** which must be wrapped in a try/catch block:

```javascript
const { parse, sequenceOf, str, char, toValue } = require('arcsecond');

const parser = sequenceOf ([
  str ('hello'),
  char (' '),
  str ('world')
]);

const parseResult = parse(parser)('hello world');

try {
  const result = toValue(parseResult);
  // -> [ 'hello', ' ', 'world' ] 
} catch (err) {
  // do something with err
}
```

### Something more complex: JSON

Let’s put **arcsecond** through its paces by using it to create a parser for JSON.

> [_Click here to skip ahead and see the full JSON parser in one file_](https://gist.github.com/francisrstokes/4d5f5b2de9644cf547799e3ac85fc6e2)

#### Values

JSON only has 7 possible values:

*   String
*   Number
*   true
*   false
*   null
*   Array
*   Object

So to write a JSON parser, we just need to write parsers for all these values.

#### Types

In order for our parser to be useful we need to be able to identify what we’ve parsed, and the best way to do that is to put the results into a data type, which will provide a common interface for us to interact with the JSON tree. Every type has a _type_ name, a _value_, and a _toString_ function to pretty print the structure.

```javascript
// Function to create a single-valued "type constructor"
const makeBasicType = typeName => value => ({
  type: typeName,
  value,
  toString: () => `${typeName}(${value})`
});

// Function to create a multi-valued "type constructor"
const makeMultiType = typeName => values => ({
  type: typeName,
  value: values,
  toString: () => `${typeName}(${values.map(v => v.toString()).join(', ')})`
});



const stringType = makeBasicType('String');
const numberType = makeBasicType('Number');
const booleanType = makeBasicType('Boolean');

const arrayType = makeMultiType('Array');
const objectType = makeMultiType('Object');

// For the object, we can store the key/value pairs in their own type
const keyValuePair = (key, value) => ({
  type: 'KeyValuePair',
  value: [key, value],
  toString: () => `KeyValuePair(${key.toString()}, ${value.toString()})`
});

const nullType = () => ({
  type: 'Null',
  value: null,
  toString: () => 'Null'
});



arrayType([
  stringType("hello"),
  numberType(42),
  stringType("world"),
  numberType(84),
]).toString();
// -> Array(String(hello), Number(42), String(world), Number(84))
```

With our types in hand let’s start with the absolute simplest of parsers: **_true_**, **_false_** and **_null_**. These are just literal strings:

```javascript
const { parse, str, choice } = require('arcsecond');

const nullParser = str ('null') .map(nullType);
const trueParser = str ('true') .map(booleanType);
const falseParser = str ('false') .map(booleanType);

const jsonParser = choice ([
  nullParser,
  trueParser,
  falseParser
]);
```

> The parsers in **arcsecond** have a **_map_** method — just like arrays —which allows you to transform the value the parser matched. With **_map_** we can put the values matched into the data types defined above.

Numbers are a bit more complex. The [JSON specification](https://json.org) has this railroad diagram showing how a number can be parsed:

{% img center /images/json-number.gif 600 600 'image' 'images' %}

The forks in the railroads show optionality, so the simplest number that can be matched is just 0.

Basically a numbers like:

*   _1_
*   _\-0.2_
*   _3.42e2_
*   _\-0.4352E-235_

Are all valid in JSON, while something like _03.46_ is not, because no path in the railroad would allow for that.

```javascript
const {
  choice,
  sequenceOf,
  char,
  digits,
  regex,
  possibly,
  anyOfString
} = require('arcsecond');

// The possibly parser either returns what it matches or null
// We can use the possibly parser to make one that returns empty string
// instead of null, which is more useful in this case
const orEmptyString = parser => possibly (parser) .map(x => (x) ? x : '');

// We also want to get sequences of matches, but then join them back
// together afterwards, instead of being in an array
const joinedSequence = parsers => sequenceOf (parsers) .map(x => x.join(''));

const numberParser = joinedSequence ([
  orEmptyString (char ('-')),
  choice ([
    char ('0'),
    regex (/^[1-9][0-9]*/)
  ]),
  orEmptyString (joinedSequence ([
    char ('.'),
    digits
  ])),
  orEmptyString (joinedSequence([
    anyOfString ('eE'),
    orEmptyString (anyOfString ('-+')),
    digits
  ]))
]) .map (x => numberType(Number(x)));


// Now we can add the numberParser to the jsonParser
const jsonParser = choice ([
  nullParser,
  trueParser,
  falseParser,
  numberParser
]);
```

If you take the time to read through the **_numberParser_**, you’ll see it lines up pretty much 1:1 with the diagram above.

Let’s try strings next. A string is anything between double quotes, but it can also contain escaped quotes.

```javascript
const {
  between,
  many,
  choice,
  sequenceOf,
  char,
  anythingExcept
} = require('arcsecond');

// Like joinedSequence, we want to get many of the same matches and then join them
const joinedMany = parser => many (parser) .map(x => x.join(''));

const betweenQuotes = between (char ('"')) (char ('"'));

const stringParser = betweenQuotes (joinedMany (choice ([
  joinedSequence ([
    char ('\\'),
    char ('"')
  ]),
  anythingExcept (char ('"'))
]))) .map(stringType);

// Adding the stringParser to the jsonParser
const jsonParser = choice ([
  nullParser,
  trueParser,
  falseParser,
  numberParser,
  stringParser
]);
```

The **_anythingExcept_** parser comes in very handy here, and is especially expressive when compared with the image on the JSON spec website.

{% img center /images/json-str.gif 600 600 'image' 'images' %}

That only leaves Array and Object, which can both be pitfalls because they are basically just containers of **_jsonValue._** To illustrate how this might go wrong, we can write the Array parser the “wrong” way first and then see how to address it.

We can use the **_whitespace_** parser — which matches zero or more whitespace characters — to ensure the the array brackets and comma operator allow for any (optional) whitespace that might be there.

```javascript
const {
  between,
  char,
  choice,
  sepBy,
  whitespace
} = require('arcsecond');

const whitespaceSurrounded = between (whitespace) (whitespace)
const betweenSquareBrackets = between (whitespaceSurrounded (char ('['))) (whitespaceSurrounded(char (']')));
const commaSeparated = sepBy (whitespaceSurrounded (char (',')));

const arrayParser = betweenSquareBrackets (commaSeparated (jsonParser)) .map(arrayType);

const jsonParser = choice ([
  nullParser,
  trueParser,
  falseParser,
  numberParser,
  stringParser,
  arrayParser
]);

// ReferenceError: jsonParser is not defined
```

Because **_arrayParser_** is defined in terms of **_jsonParser_**, and **_jsonParser_** is defined in terms of **_arrayParser_**, we run into a _ReferenceError_. If we moved the definition of **_arrayParser_** below **_jsonParser_**, we’d still have the same problem. We can fix this by wrapping the **_jsonParser_** in a special parser, aptly named **_recursiveParser_**. [The argument to **_recursiveParser_** is a thunk](https://en.wikipedia.org/wiki/Thunk), which will allow us to reference variables that are not yet in scope.

```javascript
const {
  between,
  char,
  choice,
  sepBy,
  whitespace,
  recursiveParser
} = require('arcsecond');

const whitespaceSurrounded = between (whitespace) (whitespace)
const betweenSquareBrackets = between (whitespaceSurrounded (char ('['))) (whitespaceSurrounded(char (']')));
const commaSeparated = sepBy (whitespaceSurrounded (char (',')));

// Move this parser to the top, and wrap it in a recursiveParser,
// which takes a function that returns a parser
const jsonParser = recursiveParser (() => choice ([
  nullParser,
  trueParser,
  falseParser,
  numberParser,
  stringParser,
  arrayParser
]));

const arrayParser = betweenSquareBrackets (commaSeparated (jsonParser)) .map(arrayType);
```

Implementing the **_arrayParser_** is actually quite trivial — as simple as JSON values, separated by commas, in square brackets.

Object is only marginally more complex. The values in an object are pairs of strings some other JSON value, with a colon as a separator.

```javascript
const {
  between,
  char,
  choice,
  sepBy,
  whitespace,
  recursiveParser
} = require('arcsecond');

const betweenCurlyBrackets = between (whitespaceSurrounded (char ('{'))) (whitespaceSurrounded(char ('}')));

const jsonParser = recursiveParser (() => choice ([
  nullParser,
  trueParser,
  falseParser,
  numberParser,
  stringParser,
  arrayParser,
  objectParser
]));

const keyValuePairParser = sequenceOf ([
  stringParser,
  whitespaceSurrounded (char (':')),
  jsonParser
]) .map(([key, _, value]) => keyValuePair(key, value));

const objectParser = betweenCurlyBrackets (commaSeparated (keyValuePairParser)) .map(objectType);
```

And that’s it. The **_jsonValue_** can be used to parse a JSON document in it’s entirety. [The full parser can be found here as a gist.](https://gist.github.com/francisrstokes/4d5f5b2de9644cf547799e3ac85fc6e2)

### Bonus Parser: CSV

Since I opened up with a CSV parser in Haskell, let’s see how that would look in **arcsecond**. I’ll keep it minimal and forgo creating data types to hold the values, and some extra strengthening that the Parsec version also doesn’t have.

The result should be an array of arrays — the outer array holds “lines” and the inner arrays contain the elements of the line.

```javascript
const {
  parse,
  char,
  many,
  regex,
  anythingExcept,
  sepBy
} = require('arcsecond');

const joinedMany = parser => many (parser) .map(x => x.join(''));

const cell = joinedMany (anythingExcept (regex (/^[,\n]/)));
const cells = sepBy (char (',')) (cell);
const parser = sepBy (char ('\n')) (cells);

const data = `
1,React JS,"A declarative efficient and flexible JavaScript library for building user interfaces"
2,Vue.js,"Vue.js is a progressive incrementally-adoptable JavaScript framework for building UI on the web."
3,Angular,"One framework. Mobile & desktop."
4,ember.js,"Ember.js - A JavaScript framework for creating ambitious web applications"`;

console.log(
  parse (parser) (data)
);
// -> [
//   [ '' ],
//   [ '1', 'React JS', '"A declarative efficient and flexible JavaScript library for building user interfaces"' ],
//   [ '2', 'Vue.js', '"Vue.js is a progressive incrementally-adoptable JavaScript framework for building UI on the web."' ],
//   [ '3', 'Angular', '"One framework. Mobile & desktop."' ],
//   [ '4', 'ember.js', '"Ember.js - A JavaScript framework for creating ambitious web applications"' ],
// ]
```

### Conclusion

There are quite a few key features of **arcsecond** that didn’t get a mention in this article, including the fact it [can parse context sensitive languages](https://en.wikipedia.org/wiki/Context-sensitive_grammar), and the parsing model is based on a [Fantasy Land -compliant Monad](https://github.com/fantasyland/fantasy-land). My main goal with this project was to bring the same level of expressivity that Parsec has to JavaScript, and I hope I’ve been able to do that.

[Please check out the project on github along with all the API docs and examples](https://github.com/francisrstokes/arcsecond), and think of [**arcsecond**](https://github.com/francisrstokes/arcsecond) the next time you find yourself writing an incomprehensible spaghetti regex — you might be using the wrong tool for the job! You can install the latest version with:

> **npm i arcsecond**

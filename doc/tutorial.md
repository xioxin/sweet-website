---
layout: default
toc: true
use_anchors: true
---

# 介绍

Sweet为JavaScript带来了Scheme和Rust等语言的卫生宏。
宏允许您使用Javascript的语法来修饰您一直想要的语言。

## 安装和入门

Install Sweet with npm:

```sh
$ npm install -g @sweet-js/cli @sweet-js/helpers
```

This globally installs the `sjs` binary, which is used to compile Sweet code.

For example, say you'd like to sweeten JavaScript with a simple hello world macro.
You can write it down as the following:

```js
// sweet_code.js
syntax hi = function (ctx) {
  return #`console.log('hello, world!')`;
};
hi
```

Then, you can use the `sjs` command to compile the sweetened code into plain JavaScript:

```sh
$ sjs sweet_code.js
console.log('hello, world!')
```


## Babel后端


注意sweet使用[babel]（https://babeljs.io/）作为后端。Sweet完成了宏的查找和扩展工作后，生成的代码将通过babel运行。


默认情况下，babel不执行任何转换，因此您需要根据需要配置它。最简单的方法是通过一个[`.babelrc`]（https://babeljs.io/docs/usage/babelrc/）文件。最小配置如下所示：

```js
{
    "presets": ["es2015"]
}
```


通过NPM安装ES2015预设：

```
npm install babel-preset-es2015
```

如果您根本不想使用babel，请将“--no babel”标志传递给“sjs”。

# Sweet Hello
宏是如何工作的？

从某种意义上讲，宏有点像compiletime函数；就像函数一样，宏也有定义和调用，它们一起将代码抽象到一个位置，这样您就不会一直重复自己的代码。

Consider the hello world example again:

```js
syntax hi = function (ctx) {
  return #`console.log('hello, world!')`;
};
hi
```
前三行构成宏定义。关键字“syntax”有点像“let”，因为它在当前块范围内创建了一个新变量。‘syntax’不是为运行时值创建变量，而是为_compileTime值创建一个新变量。在这种情况下，“hi”是绑定到前三行上定义的compileTime函数的变量。

在本例中，`syntax`将变量设置为函数，但变量可以设置为任何javascript值。目前，这一点比较学术化，因为Sweet不提供实际使用编译时间函数以外的任何东西的方法。但是，最终将添加此功能。

一旦定义了宏，就可以调用它。在上面的第三行，只需编写“hi”就可以调用宏。


当Sweet编译器看到绑定到compileTime函数的“hi”标识符时，将调用该函数，并使用其返回值替换调用出现的“hi”。在本例中，这意味着“hi”将替换为“console.log”（“hello，world！”）'.





Compiletime functions defined by `syntax` must return an array of syntax objects. You can easily create these with a _syntax template_. Syntax templates are template literals with a `\#` tag, which create a List (see the [immutable.js docs](https://facebook.github.io/immutable-js/docs/#/List) for its API)
of syntax objects.
由'syntax'定义的compileTime函数必须返回一个语法对象数组。您可以使用\语法模板\轻松创建这些。语法模板是带有`\ `标记的模板文本，它创建了一个列表（请参阅[immutable.js docs]（https://facebook.github.io/immutable js/docs//list）了解其API）

语法对象。

A _syntax object_ is  Sweet's internal representation of syntax. Syntax objects are somewhat like tokens from traditional compilers except that delimiters cause syntax objects to nest. This nesting gives Sweet more structure to work with during compilation. If you are coming from Lisp or Scheme, you can think of them a bit like s-expressions.


# Sweet New

让我们来看一个稍微有趣的例子。

假设您使用的是用于javascript的OO框架，而不是使用“new”，我们想调用一个“create”方法，该方法已被猴修补到“function.prototype”上（不用担心，我不会判断…很多）。与其手动将“new”的所有用法重写为“create”方法，还不如定义一个宏来执行该操作。

```js
syntax new = function (ctx) {
  let ident = ctx.next().value;
  let params = ctx.next().value;
  return #`${ident}.create ${params}`;
};

new Droid('BB-8', 'orange');
```

```js
Droid.create('BB-8', 'orange');
```
在这里，您可以看到宏的“ctx”参数提供了对宏调用站点语法的访问。此参数是名为_macro context_uu的迭代器。

宏上下文的类型为：

```
{
  next: () -> {
    done: boolean,
    value: Syntax
  }
}
```

Each call to `next` returns the successive syntax object in `value` until there is nothing left in which case `done` is set to true. Note that the context is also an iterable so you can use `for-of` and related goodies.

Note that in this example we only call `next` twice even though it looks like there is more than two bits of syntax we want to match. What gives? Well, remember that delimiters cause syntax objects to nest. So, as far as the macro context is concerned there are two syntax objects: `Droid` and a single paren delimiter syntax object containing the three syntax objects `'BB-8'`, `,`, and `'orange'`.

After grabbing both syntax objects with the macro context iterator we can stuff them into a syntax template. Syntax templates allow syntax objects to be used in interpolations so it is straightforward to get our desired result.

# Sweet Let

Ok, time to make some ES2015. Let's say we want to implement `let`.
We only need one new feature you haven't seen yet:

```js
syntax let = function (ctx) {
  let ident = ctx.next().value;
  ctx.next(); // eat `=`
  let init = ctx.expand('expr').value;
  return #`
    (function (${ident}) {
      ${ctx} // <2>
    }(${init}))
  `
};

let bb8 = new Droid('BB-8', 'orange');
console.log(bb8.beep());
```

```js
(function(bb8) {
  console.log(bb8.beep());
})(Droid.create("BB-8", "orange"));
```

Calling `expand` allows us to specify the grammar production we want to match; in this case we are matching an expression. You can think matching against a grammar production a little like matching an implicitly-delimited syntax object; these matches group multiple syntax object together.


# Sweet Cond

One task we often need to perform in a macro is looping over syntax. Sweet helps out with that by supporting ES2015 features like `for-of`. To illustrate, here's a `cond` macro that makes the ternary operator a bit more readable:

```js
import { unwrap, isKeyword } from '@sweet-js/helpers' for syntax;

syntax cond = function (ctx) {
  let bodyCtx = ctx.contextify(ctx.next().value);

  let result = #``;
  for (let stx of bodyCtx) { // <2>
    if (isKeyword(stx) && unwrap(stx).value === 'case') {
      let test = bodyCtx.expand('expr').value;
      // eat `:`
      bodyCtx.next();
      let r = bodyCtx.expand('expr').value;
      result = result.concat(#`${test} ? ${r} :`);
    } else if (isKeyword(stx) && unwrap(stx).value === 'default') {
      // eat `:`
      bodyCtx.next();
      let r = bodyCtx.expand('expr').value;
      result = result.concat(#`${r}`);
    } else {
      throw new Error('unknown syntax: ' + stx);
    }
  }
  return result;
};

let x = null;

let realTypeof = cond {
  case x === null: 'null'
  case Array.isArray(x): 'array'
  case typeof x === 'object': 'object'
  default: typeof x
}
```

```js
var x = null;
var realTypeof = x === null ? "null" :
                 Array.isArray(x) ? "array" :
                 typeof x === "undefined" ? "undefined" : typeof x);
```

Since delimiters nest syntax in Sweet, we need a way to get at the nested syntax. The `contextify` method on the macro context provides this functionality. Calling `contextify` with a delimiter will wrap the syntax inside the delimiter in a new macro context object that you can iterate over.

Utility functions are available for import at `'@sweet-js/helpers'` to inspect the syntax you are processing. Note the `for syntax` on the import declaration, which is required to make the imports available inside of a macro definition. Modules and macros are described in the modules section of this document.

The two imports used by this macro are `isKeyword` and `unwrap`. The `isKeyword` function does what it sounds like: it tells you if a syntax object is a keyword. The `unwrap` function returns the primitive value of a syntax object. In the case of unwrapping a keyword, it returns the string representation (if the syntax was a numeric literal `unwrap` would return a number). Since some syntax objects do not have a reasonable primitive representation (e.g. delimiters), the `unwrap` function ether returns an object with a single `value` property or an empty object.

The full API provided by the helper module is described in the [reference documentation](reference.html).

# Sweet Class

So putting together what we've learned so far, let's make the sweetest of ES2015's features: `class`.

```js
import { unwrap, isIdentifier } from '@sweet-js/helpers' for syntax;

syntax class = function (ctx) {
  let name = ctx.next().value;
  let bodyCtx = ctx.contextify(ctx.next().value);

  // default constructor if none specified
  let construct = #`function ${name} () {}`;
  let result = #``;
  for (let item of bodyCtx) {
    if (isIdentifier(item) && unwrap(item).value === 'constructor') {
      construct = #`
        function ${name} ${bodyCtx.next().value}
        ${bodyCtx.next().value}
      `;
    } else {
      result = result.concat(#`
        ${name}.prototype.${item} = function
            ${bodyCtx.next().value}
            ${bodyCtx.next().value};
      `);
    }
  }
  return construct.concat(result);
};

class Droid {
  constructor(name, color) {
    this.name = name;
    this.color = color;
  }

  rollWithIt(it) {
    return this.name + " is rolling with " + it;
  }
}
```

```js
function Droid(name, color) {
  this.name = name;
  this.color = color;
}

Droid.prototype.rollWithIt = function(it) {
  return this.name + " is rolling with " + it;
};
```

# Sweet Modules

Now that you've created your sweet macros you probably want to share them! Sweet supports this via ES2015 modules:

```js
'lang sweet.js';
export syntax class = function (ctx) {
  // ...
};
```

```js
import { class } from './es2015-macros';

class Droid {
  constructor(name, color) {
    this.name = name;
    this.color = color;
  }

  rollWithIt(it) {
    return this.name + " is rolling with " + it;
  }
}
```

The `'lang sweet.js'` directive lets Sweet know that a module exports macros, so you need it in any module that has an `export syntax` in it. This directive allows Sweet to not bother doing a lot of unnecessary expansion work in modules that do not export syntax bindings. Eventually, this directive will be used for other things such as defining a base language.

## Modules and Phasing

In addition to importing macros Sweet also lets you import runtime code to use in compiletime code.

As you've probably noticed, we have not seen a way to define a function that a macro's compiletime function can use:

```js
let log = msg => console.log(msg);

syntax m = ctx => {
  log('doing some Sweet things'); // ERROR: unbound variable `log`
  // ...
};
```

We get an unbound variable error in the above example because `m`'s definition runs at a different _phase_ than the surrounding code: namely it runs at compiletime while the surrounding code is invoked at runtime.

Sweet solves this problem by allowing you to import values for a particular phase:

```js
'lang sweet.js';

export function log(msg) {
  console.log(msg);
}
```

```js
import { log } from './log.js' for syntax;

syntax m = ctx => {
  log('doing some Sweet things');
  // ...
};
```

Adding `for syntax` to an import statement lets Sweet know that it needs to load the values being imported into the compiletime environment and make them available for macro definitions.

NOTE: Importing for syntax is currently only supported for Sweet modules (i.e. those that begin with `'lang sweet.js'`). Support for non-Sweet modules is coming soon.


# Sweet Operators

In addition to the macros we've seen so far, Sweet allows you to define custom operators. Custom operators are different from macros in that you can specify the precedence and associativity but you can't match arbitrary syntax; the operator definition is invoked with fully expanded expressions for its operands.

Operators are defined via the `operator` keyword:

```js
operator >>= left 1 = (left, right) => {
  return #`${left}.then(${right})`;
};

fetch('/foo.json') >>= resp => { return resp.json() }
                   >>= json => { return processJson(json) }
```

```js
fetch("/foo.json").then(resp => {
  return resp.json();
}).then(json => {
  return processJson(json);
});
```

The associativity can be either `left` or `right` for binary operators and `prefix` or `postfix` for unary operators. The precedence is a number that specifies how tightly the operator should bind. The builtin operators range from a precedence of 0 to 20 and are defined [here](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Operator_Precedence).

The operator definition must return an expression.

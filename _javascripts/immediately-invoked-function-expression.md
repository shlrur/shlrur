---
title: "Immediately-Invoked Function Expression (IIFE)"
date: 2018-09-01
categories:
  - JavaScript
tags:
  - JavaScript
  - Function
  - IIFE
last_modified_at: 2018-09-27T14:00:00+09:00
---

>이 포스트는 [Immediately-Invoked Function Expression (IIFE)](http://benalman.com/news/2010/11/immediately-invoked-function-expression/) 의 글을 번역한 것입니다.

잘 모르시겠지만, 저는 용어 사용에 있어서 까다로운 편입니다. 그래서 유명하지만 오해의 소지가 있는 JavaScript 용어인 "self-executing anonymous function" (혹은 self-invoked anonymous function) 을 매우 자주 접한 후, 제 생각을 article에 정리해야겠다고 생각했습니다.

이 패턴이 실제로 어떻게 작용하는지에 대해 철저한 정보를 제공할 뿐만 아니라, 그것의 이름을 제안하려 합니다. 또한, 되도록 이 포스트의 모든 부분을 읽기를 추천합니다.

이 포스트는 "나는 옳다, 네가 틀렸다" 같은 종류가 아니라는 것을 알아주십시오. 저는 사람들이 복잡한 개념을 쉽게 이해하도록 돕고 싶습니다. 그리고 일관되고 정확한 용어를 사용하는 것이 사람들의 이해를 돕는 가장 쉬운 일 중 하나라고 생각합니다.

## So, what’s this all about, anyways?

JavaScript에서 모든 [function](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Function)은 실행될 때 **execution context**를 만듭니다. Function 내에서 정의된 variable과 function은 그 function 내부에서만 접근 가능하기 때문에, function 호출로 인해서 쉽게 privacy를 제공할 수 있습니다.

```js
  // Because this function returns another function that has access to the
  // "private" var i, the returned function is, effectively, "privileged."

  function makeCounter() {
    // `i` is only accessible inside `makeCounter`.
    var i = 0;

    return function() {
      console.log( ++i );
    };
  }

  // Note that `counter` and `counter2` each have their own scoped `i`.

  var counter = makeCounter();
  counter(); // logs: 1
  counter(); // logs: 2

  var counter2 = makeCounter();
  counter2(); // logs: 1
  counter2(); // logs: 2

  i; // ReferenceError: i is not defined (it only exists inside makeCounter)
```

대부분의 경우, <kbd>makeWhatever</kbd>가 return 하는 instance는 여러 개가 필요하지 않고, 하나로 작업 수행이 가능합니다. 다른 경우에는 값을 명시적으로 return 하지 않기도 합니다.

### The heart of the matter

Function을 <kbd>function foo(){}</kbd> 또는 <kbd>var foo = function(){}</kbd> 같은 어떤 방식으로 정의하든 간에, <kbd>foo()</kbd> 처럼 괄호를 사용해서 호출할 수 있습니다.

```js
  // Because a function defined like so can be invoked by putting () after
  // the function name, like foo(), and because foo is just a reference to
  // the function expression `function() { /* code */ }`...

  var foo = function(){ /* code */ }

  // ...doesn't it stand to reason that the function expression itself can
  // be invoked, just by putting () after it?

  function(){ /* code */ }(); // SyntaxError: Unexpected token (
```

위의 코드를 보면, 함정이 있다는 것을 알 수 있습니다. 구문 분석기(parser)는 global scope 또는 function 내부에서 <kbd>function</kbd> 이라는 keyword를 만나면 기본적으로 [함수 표현식(function expression)](https://developer.mozilla.org/en/JavaScript/Reference/Operators/Special/function)이 아닌 [함수 선언(function declaration/statement)](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function)으로 처리합니다. Parser에게 function expression임을 명시하지 않으면, parser는 이것을 이름이 없는 function declaration이라고 판단하고 [SyntaxError](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/SyntaxError)를 던집니다. 왜냐하면 function declaration은 이름이 필요하기 때문입니다.

### An aside: functions, parens, and SyntaxErrors

흥미롭게도, 만약 그 function의 이름을 지정하고 그 뒤에 괄호를 넣는다면, parser는 또 다른 이유로 SyntaxError를 던집니다. Function expression 뒤에 오는 괄호는 해당 expression의 호출을 의미하지만, function declaration 뒤에 오는 괄호는 앞의 declaration과 완전 별개이며, 단순히 그룹 연산자(평가 우선순위를 제어하는 수단)로 인식하기 때문입니다.

```js
  // While this function declaration is now syntactically valid, it's still
  // a statement, and the following set of parens is invalid because the
  // grouping operator needs to contain an expression.

  function foo(){ /* code */ }(); // SyntaxError: Unexpected token )

  // Now, if you put an expression in the parens, no exception is thrown...
  // but the function isn't executed either, because this:

  function foo(){ /* code */ }( 1 );

  // Is really just equivalent to this, a function declaration followed by a
  // completely unrelated expression:

  function foo(){ /* code */ }

  ( 1 );
```

이에 관해 더 자세한 정보를 원한다면 _Dmitry A. Soshnikov_ 의 [ECMA-262-3 in detail. Chapter 5. Functions](http://dmitrysoshnikov.com/ecmascript/chapter-5-functions/#question-about-surrounding-parentheses) 를 읽어보십시오.

## Immediately-Invoked Function Expression (IIFE)

다행히도 앞에서 보여드린 SyntaxError의 fix는 간단합니다. Parser가 function expression으로 인식하도록 하는 가장 널리 쓰이는 방법은 괄호로 감싸는 것입니다. JavaScript에서 괄호는 function declaration을 포함할 수 없기 때문입니다. 이때 parser는 <kbd>function</kbd> keyword를 만나면 function declaration이 아닌 function expression으로 인식할 것입니다.

```js
  // Either of the following two patterns can be used to immediately invoke
  // a function expression, utilizing the function's execution context to
  // create "privacy."

  (function(){ /* code */ }()); // Crockford recommends this one
  (function(){ /* code */ })(); // But this one works just as well

  // Because the point of the parens or coercing operators is to disambiguate
  // between function expressions and function declarations, they can be
  // omitted when the parser already expects an expression (but please see the
  // "important note" below).

  var i = function(){ return 10; }();
  true && function(){ /* code */ }();
  0, function(){ /* code */ }();

  // If you don't care about the return value, or the possibility of making
  // your code slightly harder to read, you can save a byte by just prefixing
  // the function with a unary operator.

  !function(){ /* code */ }();
  ~function(){ /* code */ }();
  -function(){ /* code */ }();
  +function(){ /* code */ }();

  // Here's another variation, from @kuvos - I'm not sure of the performance
  // implications, if any, of using the `new` keyword, but it works.
  // http://twitter.com/kuvos/status/18209252090847232

  new function(){ /* code */ }
  new function(){ /* code */ }() // Only need parens if passing arguments
```

### An important note about those parens

Parser가 이미 function expression으로 판단해서 추가적인 "모호성 제거" 괄호가 필요하지 않은 경우에도 괄호의 사용은 유용할 수 있습니다. 규칙을 정할 수 있기 때문입니다.(_matter of convention_)

이러한 괄호는 function expression이 즉히 실행되고 variable에 function의 결과가 입력됨을 나타냅니다(function 자체가 입력되는 것이 아닙니다). 이렇게 하면 당신의 코드를 읽는 사람이 매우 긴 function expression을 스크롤 해서 제일 아래를 보고 호출되는지 아닌지를 확인해야 하는 문제를 줄일 수 있습니다.

_경험에 비추어 볼 때 JavaScript parser가 SyntaxError exception을 던지지 않게 하려면 모호하지 않은 코드를 작성하는 것이 기술적으로 필요합니다. 하지만 더 나아가서, 다른 개발자가 "WTFError" exception을 던지지 않게 하려면 모호하지 않은 코드의 작성은 필수입니다._

### Saving state with closures

명명된 식별자에 의해 함수가 호출될 때 arguments를 전달할 수 있는것과 마찬가지로, IIFE에도 arguments를 전달할 수 있습니다. 그리고 다른 function 내에서 정의된 모든 function들은 외부 function에 전달된 모든 argument 및 variable에 접근할 수 있으므로(이러한 관계를 closure라고 합니다.), **Immediately-Invoked Function Expression**를 사용하여 값을 잠그고(lock in) 상태를 효과적으로 저장할 수 있습니다.

_Closure에 대해서 더 알고싶다면, [Closures explained with JavaScript](http://skilldrick.co.uk/2011/04/closures-explained-with-javascript/)를 읽어보세요._

```js
  // This doesn't work like you might think, because the value of `i` never
  // gets locked in. Instead, every link, when clicked (well after the loop
  // has finished executing), alerts the total number of elements, because
  // that's what the value of `i` actually is at that point.

  var elems = document.getElementsByTagName( 'a' );

  for ( var i = 0; i < elems.length; i++ ) {

    elems[ i ].addEventListener( 'click', function(e){
      e.preventDefault();
      alert( 'I am link #' + i );
    }, 'false' );

  }

  // This works, because inside the IIFE, the value of `i` is locked in as
  // `lockedInIndex`. After the loop has finished executing, even though the
  // value of `i` is the total number of elements, inside the IIFE the value
  // of `lockedInIndex` is whatever the value passed into it (`i`) was when
  // the function expression was invoked, so when a link is clicked, the
  // correct value is alerted.

  var elems = document.getElementsByTagName( 'a' );

  for ( var i = 0; i < elems.length; i++ ) {

    (function( lockedInIndex ){

      elems[ i ].addEventListener( 'click', function(e){
        e.preventDefault();
        alert( 'I am link #' + lockedInIndex );
      }, 'false' );

    })( i );

  }

  // You could also use an IIFE like this, encompassing (and returning) only
  // the click handler function, and not the entire `addEventListener`
  // assignment. Either way, while both examples lock in the value using an
  // IIFE, I find the previous example to be more readable.

  var elems = document.getElementsByTagName( 'a' );

  for ( var i = 0; i < elems.length; i++ ) {

    elems[ i ].addEventListener( 'click', (function( lockedInIndex ){
      return function(e){
        e.preventDefault();
        alert( 'I am link #' + lockedInIndex );
      };
    })( i ), 'false' );

  }
```

_마지막 두 가지 예에서 <kbd>lockedInIndex</kbd>는 아무 문제 없이 <kbd>i</kbd>라고 볼 수 있지만, function arguments로 다른 이름을 가진 식별자를 사용하면 개념을 훨씬 쉽게 설명할 수 있습니다._

IIFE의 가장 유리한 부작용(?) 중 하나는 \<이름이 지정되지 않았거나 익명인 function expression이 식별자를 사용하지 않고 즉시 호출되기 때문에 현재 범위를 오염시키지 않고 closure를 사용할 수 있다\>는 것입니다.

### What’s wrong with “Self-executing anonymous function?”

이미 여러번 언급되었지만, 확실하지 않은 경우 "**Immediately-Invoked Function Expression**"를 제안합니다. 약어를 좋아하신다면 "**IIFE**"를 제안합니다.

**Immediately-Invoked Function Expression**이 뭔가요? 바로 호출되는 [function expression](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/function)입니다. 이름 그대로 말이죠.


저는 JavaScript 커뮤니티 회원들이 기사와 프리젠테이션에서 "**Immediately-Invoked Function Expression**"과 "**IIFE**"라는 용어를 사용하는 것을 보고 싶습니다. 왜냐하면 그것이 이 개념을 좀 더 쉽게 이해하도록 만든다고 생각하기 때문입니다.

```js
  // This is a self-executing function. It's a function that executes (or
  // invokes) itself, recursively:

  function foo() { foo(); }

  // This is a self-executing anonymous function. Because it has no
  // identifier, it must use the  the `arguments.callee` property (which
  // specifies the currently executing function) to execute itself.

  var foo = function() { arguments.callee(); };

  // This *might* be a self-executing anonymous function, but only while the
  // `foo` identifier actually references it. If you were to change `foo` to
  // something else, you'd have a "used-to-self-execute" anonymous function.

  var foo = function() { foo(); };

  // Some people call this a "self-executing anonymous function" even though
  // it's not self-executing, because it doesn't invoke itself. It is
  // immediately invoked, however.

  (function(){ /* code */ }());

  // Adding an identifier to a function expression (thus creating a named
  // function expression) can be extremely helpful when debugging. Once named,
  // however, the function is no longer anonymous.

  (function foo(){ /* code */ }());

  // IIFEs can also be self-executing, although this is, perhaps, not the most
  // useful pattern.

  (function(){ arguments.callee(); }());
  (function foo(){ foo(); }());

  // One last thing to note: this will cause an error in BlackBerry 5, because
  // inside a named function expression, that name is undefined. Awesome, huh?

  (function foo(){ foo(); }());
```

위의 예들이 "self-executing"이라는 단어가 불러일으키는 오해를 좀 더 명확하게 이해하는 데 도움이 되길 바랍니다. 왜냐하면 "self-executing" function은 실행되고 있음에도 불구하고, 그 자체가 실행되는 것은 아니기 때문입니다. 또한 **Immediately Invoked Function Expression**은 익명일 수도 있고 이름이 지정될 수도 있기 때문에 "anonymous"는 불필요하게 구체적인 용어입니다. 그리고 제가 "executed"보다 "invoked"를 더 선호하는 것은 단순히 구술([alliteration](https://en.wikipedia.org/wiki/Alliteration))의 문제입니다. 저는 "**IIFE**"가 "IEFE"보다 더 멋있고 멋지게 들립니다.

이게 다입니다. 저의 big idea입니다.

_Fun fact: <kbd>arguments.callee</kbd>가 [ECMAScript 5 strict mode에서는 더이상 사용하지 않기](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Strict_mode#Differences_in_functions) 때문에 "self-executing anonymous function"을 ECMAScript 5 strict mode에서 사용하기란 기술적으로 불가능합니다._

### A final aside: The Module Pattern

만약 JavaScript에서 Module pattern에 익숙하지 않다면 아래 코드의 첫 예제를 보십시오. Function이 아닌 object가 반환됩니다.(아래의 예제와 같이 일반적으로 singleton으로 구현합니다)

```js
  // Create an anonymous function expression that gets invoked immediately,
  // and assign its *return value* to a variable. This approach "cuts out the
  // middleman" of the named `makeWhatever` function reference.
  // 
  // As explained in the above "important note," even though parens are not
  // required around this function expression, they should still be used as a
  // matter of convention to help clarify that the variable is being set to
  // the function's *result* and not the function itself.

  var counter = (function(){
    var i = 0;

    return {
      get: function(){
        return i;
      },
      set: function( val ){
        i = val;
      },
      increment: function() {
        return ++i;
      }
    };
  }());

  // `counter` is an object with properties, which in this case happen to be
  // methods.

  counter.get(); // 0
  counter.set( 3 );
  counter.increment(); // 4
  counter.increment(); // 5

  counter.i; // undefined (`i` is not a property of the returned object)
  i; // ReferenceError: i is not defined (it only exists inside the closure)
```

Module pattern의 접근법은 매우 강력할뿐만 아니라 엄청나게 간단합니다. 짧은 코드로, method와 property 들을 namespace화 할 수 있고, global scope의 오염을 최소화하면서 privacy를 만드는 방식으로 _module_ 을 구성할 수 있습니다.

## Further reading

이 article이 유익하며 당신의 물음에 답이 되길 바랍니다. 물론 더 많은 물음이 생겨났겠지만, 아래의 article을 읽으면 function과 module pattern에 대해서 더 많은 것들을 알 수 있을 것입니다.

* [ECMA-262-3 in detail. Chapter 5. Functions.](http://dmitrysoshnikov.com/ecmascript/chapter-5-functions/#question-about-surrounding-parentheses) - Dmitry A. Soshnikov
* [Functions and function scope](https://developer.mozilla.org/en/JavaScript/Reference/Functions_and_function_scope) - Mozilla Developer Network
* [Named function expressions](http://kangax.github.com/nfe/) - Juriy “kangax” Zaytsev
* [JavaScript Module Pattern: In-Depth](http://www.adequatelygood.com/2010/3/JavaScript-Module-Pattern-In-Depth) - Ben Cherry
* [Closures explained with JavaScript](http://skilldrick.co.uk/2011/04/closures-explained-with-javascript/) - Nick Morgan
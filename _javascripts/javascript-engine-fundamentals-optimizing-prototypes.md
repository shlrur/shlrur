---
title: "JavaScript engine fundamentals: Shapes and Inline Caches"
---

_이 포스트는 [JavaScript engine fundamentals: Shapes and Inline Caches](https://mathiasbynens.be/notes/shapes-ics) 의 글을 번역한 것입니다._

이 문서는 V8 엔진뿐만 아니라 모든 JavaScript 엔진에 공통으로 적용되는 몇 가지 핵심 기본 사항을 설명합니다. JavaScript 개발자로서, JavaScript 엔진이 어떻게 작동하는지에 대한 이해를 통해 코드의 성능 특성을 추론할 수 있습니다.

_Note: 문서를 읽는 것보다 발표를 보는 것을 더 좋아한다면 아래의 동영상을 보십시오. 그렇지 않다면 문서를 계속해서 읽어 주십시오._
<div class="embed-responsive embed-responsive-16by9">
  <iframe width="640" height="360" src="https://www.youtube-nocookie.com/embed/5nmpokoRaZI" frameborder="0" allowfullscreen></iframe>
</div>

## The JavaScript engine pipeline

모든 것은 작성된 **JavaScript code**로부터 시작합니다. JavaScript 엔진은 소스 코드를 구문 분석(**parse**)하여 AST(**Abstract Syntax Tree**)로 변환합니다. **Interpreter**는 이 AST를 기반으로 일을 시작하고 바이트 코드를 생산할 수 있습니다. 그 시점에서 엔진은 실제로 JavaScript 코드를 실행하고 있습니다.

![image-center]({{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_shape_and_inline_caches/0_js-engine-pipeline.png){: .align-center}

위의 과정이 더 빨리 실행되도록 하기 위해, **bytecode**는 **profiling data**와 함께 **optimizing compiler**로 보내집니다. **optimizing compiler**는 자신이 보유한 **profiling data**를 기반으로 특정한 가정을 한 다음 고도로 최적화된 시스템 코드(**optimized code**)를 생성합니다.

어느 시점에서 가정 중 하나가 잘못된 것으로 판명될 경우, **optimizing compiler**는 최적화를 수행하지 않고 다시 **interpreter**로 돌아가게 됩니다.

### Interpreter/compiler pipelines in JavaScript engines

이제 JavaScript 코드를 실제로 실행하는 pipeline의 부분(즉, 코드가 해석되고 최적화되는 부분)을 파고들어 주요 JavaScript 엔진 간의 차이점 몇 가지를 살펴보겠습니다.

일반적인 경우를 말해보자면, **interpreter**와 **optimazing compiler**가 들어 있는 pipeline이 있다. 아래의 그림과 함께 살펴보자. **Interpreter**는 최적화되지 않은 **bytecode**를 빠르게 생성하며, **optimazing compiler**는 시간이 좀 더 걸리지만 결국 고도로 최적화된 머신 코드(**optimized code**)를 생성합니다.
<figure class="align-center">
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_shape_and_inline_caches/1_interpreter-optimizing-compiler.png" alt="">
    <figcaption>generally speaking compiler</figcaption>
</figure>

아래 그림의 pipeline이 작동하는 방식은 Chrome 및 Node.js에서 사용되는 JavaScript 엔진이 작동하는 방식과 거의 동일합니다.
<figure class="align-center">
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_shape_and_inline_caches/2_interpreter-optimizing-compiler-v8.png" alt="">
    <figcaption>v8 compiler</figcaption>
</figure>
V8의 interpreter는 **Ignition**이라고 불리며, **bytecode**를 생성하고 실행하는 역할을 합니다. **Bytecode**를 실행하는 동안, 이후에 실행 속도를 높이는데 사용하기 위해서 **profiling data**를 수집합니다. 예를 들어, 종종(often) 실행되는 기능에 부하가 걸리면, 생성된 **bytecode**와 **profiling data**는 **TurboFan**(최적화된 컴파일러)으로 전달되어 **profiling data**를 기반으로 최적화 머신 코드(**optimized code**)를 생성합니다.

<figure class="align-center">
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_shape_and_inline_caches/3_interpreter-optimizing-compiler-spidermonkey.png" alt="">
    <figcaption>SpiderMonkey compiler</figcaption>
</figure>
FireFox와 [SpiderNode](https://github.com/mozilla/spidernode) 에서 사용되는 Mozilla의 JavaScript 엔진인 **SpiderMonkey**는 조금 다른 방식을 사용합니다. **SpiderMonkey**는 하나가 아닌 두 개의 optimizing compiler를 가지고 있습니다. **Interpreter**는 **somewhat optimized code**(다소 최적화된 code)를 생산하는 **Baseline** compiler로 최적화를 수행합니다. **IonMonkey** 컴파일러는 **profiling data**(코드를 실행하는 동안 수집됨)와 결합하여 **optimized code**를 생성합니다. 투기적(speculative) 최적화에 실패하면 **IonMonkey**는 **Baseline** 코드로 되돌아간다.

<figure class="align-center">
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_shape_and_inline_caches/4_interpreter-optimizing-compiler-chakra.png" alt="">
    <figcaption>Chakra compiler</figcaption>
</figure>
Edge 및 Node-ChakraCore에서 사용되는 Microsoft의 JavaScript 엔진인 **Chakra**는 앞서 보여드린 두 최적화 컴파일러와 매우 유사한 설정을 가지고 있습니다. **Interpreter**는 **somewhat optimized code**를 생성하는 SimpleJIT(Just-In-Time)로 **optimized**됩니다. **Profiling data**와 결합된 FullJIT는 보다 효율적으로 **optimized code**를 생성할 수 있습니다.

<figure class="align-center">
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_shape_and_inline_caches/5_interpreter-optimizing-compiler-jsc.png" alt="">
    <figcaption>JSC compiler</figcaption>
</figure>
JavaScriptCore(JSC로 약칭)는 Safari 및 React Native에서 사용되는 Apple의 JavaScript 엔진으로, 세 가지 최적화 컴파일러를 극단적으로 사용합니다. **LLInt**(Low-Level Interpreter)는 **Baseline** compiler로 최적화되며, **Baseline** 컴파일러는 **DFG**(Data Flow Graph) 컴파일러로 최적화고, **DFG** 컴파일러는 **FTL**(Faster Than Light) 컴파일러로 최적화됩니다.

왜 어떤 엔진들은 다른 엔진들보다 더 최적화된 컴파일러를 가지고 있을까요? 모든 것은 타협(trade-offs)이라고 볼 수 있습니다. Interpreter는 빨리 bytecode를 만들 수 있지만, 일반적으로 bytecode는 그리 효율적이지 않습다. 반면에 최적화 컴파일러는 시간이 좀 더 걸리지만 결국 훨씬 더 효율적인 machine code를 만들어냅니다. 빨리 코드를 실행하거나(Interpreter) 더 많은 시간을 들일 수 있지만 결국 최적의 성능(컴파일러 최적화)으로 코드를 실행할 수 있습니다. 일부 엔진은 시간/효율의 특성이 다른 여러 개의 최적화 컴파일러를 추가함으로써 "복잡성이 가중되는 비용(the cost of additional complexity)" 과 "보다 세밀하게 제어(more fine-grained control)"사이의 타협(trade-offs)를 결정할 수 있습니다.

각 JavaScript 엔진에 대해 interpreter와 컴파일러 pipeline 최적화의 주요 차이점을 강조하여 살펴보았습니다. 그러나 이러한 차이점 외에도, high-level에서 보면 **모든 JavaScript 엔진은 동일한 아키텍처(Parser와 일종의 interpreter/compiline pipeline)를 가지고 있습니다.**

## JavaScript's object model
이제 일부 사항들이 구현되는 방법을 파고들어 JavaScript 엔진의 공통점을 살펴보겠습니다.

예를 들어 JavaScript 엔진은 어떻게 JavaScript object model을 구현하고, JavaScript object의 property에 접근하는 속도를 높이기 위해 어떤 기술을 사용할까요? 밝혀진 바와 같이, 모든 주요 엔진들은 이것들을 매우 유사하게 구현합니다.

ECMAScript 규격은 기본적으로 모든 object를 사전([_property attributes_](https://tc39.github.io/ecma262/#sec-property-attributes)에 문자열로 매핑)으로 정의합니다.

<figure class="align-center">
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_shape_and_inline_caches/6_object-model.png" alt="">
    <figcaption>Object model</figcaption>
</figure>

규격(spec)은 object 자신의 <kbd>[[value]]</kbd> 를 제외하더라도, 다른 property들을 가진다는 것을 알려줍니다.

* <kbd>[[Writable]]</kbd> : 재할당할 수 있는지 여부를 결정합니다.
* <kbd>[[Enumerable]]</kbd> : for...in 루프에서 열거될 수 있는지를 나타냅니다.
* <kbd>[[Configurable]]</kbd> : property를 삭제할 수 있는지 여부를 결정합니다.

<kbd>[[double square brakets]]</kbd> 표기법은 우스꽝스러워 보이지만, spec에서 JavaScript이 직접적으로 노출하지 않는 property를 나타내는 방법입니다. <kbd>Object.getOwnPropertyDescriptor</kbd> API를 사용하여 JavaScript에서 지정된 object 및 property에 대한 다음과 같은 property attribute 들을 가져올 수 있습니다.

```js
const object = { foo: 42 };
Object.getOwnPropertyDescriptor(object, 'foo');
// → { value: 42, writable: true, enumerable: true, configurable: true }
```

자, 이것이 JavaScript가 object를 정의하는 방법입니다. Array는 어떨까요?

Array를 object의 특별한 case라고 생각할 수 있습니다. 한 가지 차이점은 array가 array index들을 특별히 처리한다는 것입니다. 여기서 말하는 array index는 ECMAScript 규격의 특수 용어입니다. JavaScript에서 array의 길이는 최대 <var>2<sup>32</sup>-1</var> 개로 제한됩니다. Array index는 이 길이 내에서는 모두 유효합니다. 즉, <var>0 ~ 2<sup>32</sup>-2</var> 사이의 정수를 뜻합니다.

또 다른 차이점은 array에도 마법과 같은 <kbd>length</kbd> 라는 property가 있다는 것입니다.
```js
const array = ['a', 'b'];
array.length; // → 2
array[2] = 'c';
array.length; // → 3
```

위의 코드를 보면, array가 생성될 때는 <kbd>length</kbd>가 <kbd>2</kbd>였습니다. 그 후, index <kbd>2</kbd>에 다른 값을 할당하였더니 <kbd>length</kbd>가 자동으로 변경되었습니다.

JavaScript는 array를 object와 유사하게 정의합니다. 예를 들어 array index를 포함한 모든 키는 문자열로 명시적(explicitly)으로 표시됩니다. Array의 첫 번째 요소는 키 <kbd>'0'</kbd> 아래에 저장됩니다.
<figure class="align-center">
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_shape_and_inline_caches/7_array-1.png" alt="">
    <figcaption>Array - add item 1</figcaption>
</figure>
<kbd>'length'</kbd> property는 열거할 수 없고, 삭제할 수 없는 다른 property입니다.

Array에 항목이 하나 추가되면, JavaScript는 <kbd>'length'</kbd> property의 attributes중 하나인 <kbd>[[value]]</kbd> 값을 자동으로 갱신합니다.
<figure class="align-center">
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_shape_and_inline_caches/8_array-2.png" alt="">
    <figcaption>Array - add item 2</figcaption>
</figure>
일반적으로, array는 object와 비슷하게 작동합니다. :)

## Optimizing property access
우리는 object가 JavaScript에서 어떻게 정의되었는지를 알았습니다. 이제 JavaScript 엔진이 object를 효율적으로 사용하여 작업하는 방법에 대해 알아보겠습니다.

주위의 JavaScript 프로그램들을 살펴보면, property에 접근하는 것이 가장 일반적인 작업입니다. 그러므로 JavaScript 엔진이 property에 빠르게 액세스 하는 것이 중요해집니다.
```js
const object = {
	foo: 'bar',
	baz: 'qux',
};

// Here, we’re accessing the property `foo` on `object`:
doSomething(object.foo);
//          ^^^^^^^^^^
```

### Shapes
JavaScript 프로그램에서, 아래의 코드와 같이, 같은 property key들을 가지는 여러 개의 object를 사용하는 것이 일반적입니다. 이러한 object들은 같은 _shape_ 가 같다고 말합니다.
```js
const object1 = { x: 1, y: 2 };
const object2 = { x: 3, y: 4 };
// `object1` and `object2` have the same shape.
```

그리고 아래의 코드와 같이, 동일한 shape의 object에서 동일한 property에 접근하는 것 또한 매우 일반적입니다.
```js
function logX(object) {
	console.log(object.x);
	//          ^^^^^^^^
}

const object1 = { x: 1, y: 2 };
const object2 = { x: 3, y: 4 };

logX(object1);
logX(object2);
```

이러한 점들을 염두에 두고, JavaScript 엔진은 object shape에 따라 object property에 접근하는 것을 최적화할 수 있습니다. 최적화 하는 방법은 다음과 같습니다.

<kbd>x</kbd>와 <kbd>y</kbd> property를 가진 object가 있다고 가정해 보겠습니다. 앞에서 설명한 사전 데이터 구조(Key는 문자열로서 포함되고, 각각의 property attribute들을 가리킵니다)를 사용합니다.
<figure class="align-center">
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_shape_and_inline_caches/9_object-model.png" alt="">
    <figcaption>object model</figcaption>
</figure>
<kbd>object.y</kbd>에 접근한다고 할 때, JavaScript 엔진은 <kbd>JSObject</kbd>에서 key <kbd>'y'</kbd>를 찾아 해당 property attribute들을 로드한 후 <kbd>[[Value]]</kbd>를 반환합니다.

하지만 이러한 property attribute는 메모리의 어느곳에 저장될까요? 그것들을 JSObject의 일부로 저장해야 할까요? 나중에 이 shape의 object를 더 사용하게 될 것이라고 가정하면, property 이름이 동일한 object가 반복되므로, property 이름과 attribute를 포함하는 전체 사전을 JSObject 자체에 저장하는 것은 낭비가 됩니다. 이러한 것은 중복이 많아지고 불필요하게 메모리를 사용하게 됩니다. 그러므로 최적화로서, JavaScript 엔진은 object의 <kbd>shape</kbd>를 별도로 저장합니다.

<figure class="align-center">
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_shape_and_inline_caches/10_shape-1.png" alt="">
    <figcaption>shape 1</figcaption>
</figure>

아래의 그림에서, 이 <kbd>Shape</kbd>는 <kbd>[[Value]]</kbd>들을 제외한 모든 property 이름과 attribute들을 포함합니다. 대신 <kbd>Shape</kbd>는 <kbd>JSObject</kbd>에 있는 value의 offset 값을 가지고 있어서, JavaScript 엔진이 value의 위치를 알 수 있다. 같은 shape를 가진 모든 <kbd>JSObject</kbd>는 같은 <kbd>Shape</kbd> instance를 정확히 가리킵니다(point). 이제 모든 <kbd>JSObject</kbd>는 이 object에 고유한 value만 저장하면 됩니다.

<figure class="align-center">
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_shape_and_inline_caches/11_shape-2.png" alt="">
    <figcaption>shape 1</figcaption>
</figure>

여러 개의 object를 사용할 때 장점이 더 명확하게 나타납니다. 얼마나 많은 object가 있든 간에, 같은 shape가 있는 한, shape와 property 정보를 한 번만 저장하면 됩니다.

모든 JavaScript 엔진은 shape을 최적화를 위해서 사용하지만, 모두 그것을 shape이라고 하지는 않습니다.
* 학술 논문에서는 이를 _Hidden Classes_ 라고 합니다.(JavaScript class와 혼동)
* V8은 이러한 _Map_ 이라고 부릅니다.(JavaScript의 Map과 혼동) 
* Chakra는 이를 _Types_ 라고 합니다.(JavaScript의 dynamic types, typeof와 혼동)
* JavaScriptCore에서는 _Structures_ 라고 합니다.
* SpiderMonkey는 _Shapes_ 라고 부른다.

이 문서 전반에 걸쳐 'Shapes'라는 용어를 계속 사용하겠습니다.

### Transition chains and trees

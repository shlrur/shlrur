---
title: "JavaScript engine fundamentals: optimizing prototypes"
date: 2018-09-01
categories:
  - JavaScript
tags:
  - JavaScript
  - performance
last_modified_at: 2018-09-01T01:20:00+09:00
---

_이 포스트는 [JavaScript engine fundamentals: optimizing prototypes](https://mathiasbynens.be/notes/prototypes) 의 글을 번역한 것입니다._

이 문서는 V8 엔진뿐만 아니라 모든 JavaScript 엔진에 공통으로 적용되는 몇 가지 핵심 기본 사항을 설명합니다. JavaScript 개발자로서, JavaScript 엔진이 어떻게 작동하는지에 대한 이해를 통해 코드의 성능 특성을 추론할 수 있을것입니다.

[이전 포스트에서](http://localhost:4000/javascripts/javascript-engine-fundamentals-shapes-and-Inline-caches/), 우리는 JavaScript 엔진이 Shapes와 Inline Caches를 사용하여 어떻게 object와 array에 대한 접근을 최적화하는지에 대해서 논의했습니다. 이 글은 optimization pipeline의 trade-off에 관해서 설명하고, JavaScript 엔진이 prototype property에 대한 접근 속도를 높이는 방법을 설명합니다.

_**Note** : 문서를 읽는 것보다 발표를 보는 것을 더 좋아한다면 아래의 동영상을 보십시오. 그렇지 않다면 문서를 계속해서 읽어 주십시오._
<div class="embed-responsive embed-responsive-16by9">
  <iframe width="640" height="360" src="https://www.youtube-nocookie.com/embed/IFWulQnM5E0" frameborder="0" allowfullscreen></iframe>
</div>

## Optimization tiers and execution trade-offs

[이전 포스트에서](http://localhost:4000/javascripts/javascript-engine-fundamentals-shapes-and-Inline-caches/), 현대 JavaScript 엔진들이 전반적으로 동일한 pipeline(아래의 그림과 같이)을 가지는 방법에 대해서 이야기를 나눴습니다.

<figure class="align-center">
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_optimizing_prototypes/0_js-engine-pipeline.svg" alt="0">
    <figcaption>js engine pipeline</figcaption>
</figure>

또한 엔진 간에 high-level의 pipeline은 유사하지만 optimization pipeline에는 종종 차이가 있음을 지적했습니다. 왜 그런 것일까요? **일부 엔진의 최적화 tier가 다른 엔진보다 많은 이유는 무엇일까요?** 그것은 아래의 그림과 같이, 코드를 빨리 만들어서 실행하는 것과 시간을 더 들이지만 결국 최적화된 성능으로 코드를 만들어서 실행하는 것 사이의 trade-off가 있기 때문입니다.

<figure class="align-center">
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_optimizing_prototypes/1_tradeoff-startup-speed.svg" alt="1">
    <figcaption>trade-off startup speed</figcaption>
</figure>

Interpreter는 bytecode를 재빨리 만들 수 있지만, 일반적으로 bytecode는 그리 효율적이지 않습니다. 반면에 optimizing compiler는 시간이 좀 더 걸리지만 결국 훨씬 더 효율적인 machine 코드를 생산합니다.

아래의 그림이 바로 V8이 사용하는 model입니다. V8 interpreter는 Ignition이라고 불리며, 모든 JavaScript 엔진의 interpreter 중에서 가장 빠릅니다(raw bytecode 실행 측면에서) V8의 optimizing compiler이름은 TurboFan이며, 고도로 최적화된 machine 코드를 생성합니다.

<figure class="align-center">
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_optimizing_prototypes/2_tradeoff-startup-speed-v8.svg" alt="2">
    <figcaption>trade-off startup speed: V8</figcaption>
</figure>

이러한 startup latency와 execution speed 간의 trade-off는 일부 JavaScript 엔진이 최적화 tier를 추가하기로 선택하는 이유입니다. 예를 들어 SpiderMonkey는 interpreter와 전체 IonMonkey 사이에 compiler를 최적화하는 Baseline tier를 추가했습니다.

<figure class="align-center">
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_optimizing_prototypes/3_tradeoff-startup-speed-spidermonkey.svg" alt="3">
    <figcaption>trade-off startup speed: SpiderMonkey</figcaption>
</figure>

Interpreter는 bytecode를 빠르게 생성하지만, bytecode의 실행속도는 비교적 느린편입니다. Baseline은 코드를 생성하는데 약간 더 오래 걸리지만 run-time 성능을 향상시킵니다. 마지막으로, optimizing compiler인 IonMonkey는 machine 코드를 생산하는데 가장 오래 걸리지만, machine 코드는 매우 효율적으로 실행됩니다.

구체적인 예시를 통해 각 JavaScript 엔진의 파이프라인이 어떤 방법으로 처리하는지 살펴보겠습니다. 여기 자주 반복되는 코드들이 있습니다.

```js
let result = 0;
for (let i = 0; i < 4242424242; ++i) {
	result += i;
}
console.log(result);
```

V8은 Ignition interpreter에서 bytecode를 실행하기 시작합니다. 어느 시점에 V8은 코드가 hot 하다고 판단하고 TurboFan frontend를 시작합니다. TurboFan frontend는 profiling data를 통합하고 코드의 기본적인 machine representation 체계를 구축하는 TurboFan의 일부입니다. 그런 다음 추가적인 개선을 위해 다른 thread에 있는 TurboFan optimizer로 전송합니다.

<figure class="align-center">
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_optimizing_prototypes/4_pipeline-detail-v8.svg" alt="4">
    <figcaption>pipeline detail: V8</figcaption>
</figure>

Optimizer가 실행되는 동안, V8은 Ingition에서 bytecode를 실행합니다. 어느 시점에서 optimizer가 완료되고 나면 실행 가능한 machine 코드가 생성되며, 이 machine 코드로 계속해서 실행할 수 있습니다.

SpiderMonkey engine 역시 interpreter에서 bytecode를 실행하기 시작합니다. 그러나 Baseline tier가 추가되어 hot 코드는 먼저 Baseline으로 전송됩니다. Baseline compiler는 메인 thread에서 Baseline 코드를 생성하고 준비가되면 실행을 계속합니다.

<figure class="align-center">
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_optimizing_prototypes/5_pipeline-detail-spidermonkey.svg" alt="5">
    <figcaption>pipeline detail: SpiderMonkey</figcaption>
</figure>

한동안 Baseline 코드를 실행하면 SpiderMonkey는 결국 IonMonkey frontend를 실행하고 V8과 매우 유사한 optimizer를 시작합니다. IonMonkey가 최적화하는 동안 Baseline에서 실행됩니다. 마지막으로 optimizer가 완료되면, Baseline 코드 대신 최적화된 코드를 실행합니다.

Chakra의 구조는 SpiderMonkey와 매우 비슷하지만, Chakra는 메인 thread를 막지 않기 위해 더 많은 것들을 동시에 실행하려고 시도합니다. 메인 Thread에서 compiler의 일부를 실행하는 대신, Chakra는 compiler가 필요로 하는 bytecode와 profiling data를 복사하여 전용 compiler 프로세스로 보냅니다.

<figure class="align-center">
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_optimizing_prototypes/6_pipeline-detail-chakra.svg" alt="6">
    <figcaption>pipeline detail: Chakra</figcaption>
</figure>

생성된 코드가 준비되면, Chakra 엔진은 bytecode 대신 이 SimpleJIT 코드를 실행합니다. FullJIT에 대해서도 같은 과정이 실행됩니다. 이러한 접근 방식의 이점은 복사가 발생하는 일시 중지 시간이 보통 전체 compiler가(frontend)를 실행하는 데 비해 훨씬 짧다는 것입니다. 그러나 이러한 접근 방식의 단점은 **copy heuristic** 이 특정 최적화에 필요한 일부 정보를 놓칠 수 있다는 것입니다. 따라서 latency를 위해 코드 품질을 어느 정도 타협한 것입니다.

JavaScriptCore에서 모든 optimizing compiler는 모두 main JavaScript 실행과 **동시에** 일어나기 때문에 copy 단계가 없습니다! 대신에 메인 thread는 다른 thread의 compile 작업만 일으킵니다. 그 후 해당 compiler는 복잡한 locking scheme을 사용해서 메인 thread에서 profiling data에 접근합니다.

<figure class="align-center">
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_optimizing_prototypes/7_pipeline-detail-javascriptcore.svg" alt="7">
    <figcaption>pipeline detail: JavaScriptCore</figcaption>
</figure>

이 접근법의 장점은, 메인 thread에서 일어나는 JavaScript 최적화로 인한 junk를 감소시킨다는 것입니다. 단점은 복잡한 multithreading 문제를 처리해야 하는 것과 다양한 작업에 대해 약간의 locking cost를 감수해야 한다는 것입니다.

우리는 interpreter와 같이 빨리 코드를 생성하는 방법과 optimizing compiler로 빠른 코드를 생성하는 방법 사이의 trade-off에 관해서 이야기했습니다. 하지만 여기에는 또 다른 trade-off가 있습니다. 바로 **memory usage(메모리 사용량)** 입니다. 이를 설명하기 위해, 여기 두 개의 수를 더하는 간단한 JavaScript 프로그램이 있습니다.

```js
function add(x, y) {
	return x + y;
}

add(1, 2);
```

아래의 코드는 위의 <kbd>add</kbd> function을 V8의 interpreter인 Ignition을 사용해서 만든 bytecode입니다.

```js
StackCheck
Ldar a1
Add a0, [0]
Return
```

위의 bytecode에 대해서 걱정하지 마십시오. 실제로 읽을 필요는 없습니다. 요점은 **단지 4개**의 명령문으로 이루어져 있다는 것입니다.

코드가 hot해지면, TurboFan은 아래와 같이 고도로 최적화된 machine 코드를 생성합니다.

```js
leaq rcx,[rip+0x0]
movq rcx,[rcx-0x37]
testb [rcx+0xf],0x1
jnz CompileLazyDeoptimizedCode
push rbp
movq rbp,rsp
push rsi
push rdi
cmpq rsp,[r13+0xe88]
jna StackOverflow
movq rax,[rbp+0x18]
test al,0x1
jnz Deoptimize
movq rbx,[rbp+0x10]
testb rbx,0x1
jnz Deoptimize
movq rdx,rbx
shrq rdx, 32
movq rcx,rax
shrq rcx, 32
addl rdx,rcx
jo Deoptimize
shlq rdx, 32
movq rax,rdx
movq rsp,rbp
pop rbp
ret 0x18
```

코드가 _매우 많습니다._ 특히 4개의 명령문으로 이루어진 bytecode와 비교하면 더 그렇습니다. 보통 bytecode는 machine 코드보다, 특히 최적화된 machine 코드보다 더 간결한 경향이 있습니다. 반면에, bytecode는 실행하는데 interpreter가 필요하지만, 최적화된 machine 코드 processor에 의해 바로 실행될 수 있습니다.

이것이 바로 JavaScript 엔진이 "모든 것을 최적화"하지 않는 주된 이유 중 하나입니다. 우리가 앞서 살펴봤듯이, 최적화된 machine 코드를 생성하는데는 긴 시간이 걸리고, 그 외에도 **최적화된 machine 코드는 더 많은 memory가 필요하다**는 것을 알게 되었습니다.

<figure class="align-center">
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_optimizing_prototypes/8_tradeoff-memory.svg" alt="8">
    <figcaption>trade-off memory</figcaption>
</figure>

_**Note**: JavaScript 엔진들의 최적화 tier가 다른 이유는 interpreter와 같이 **코드를 빠르게 생성**하는 것과 최적화 compiler로 **빠른 코드를 생성**하는 것 사이의 근본적인 trade-off 때문입니다. 최적화 계층을 추가하면 복잡성과 오버헤드를 추가로 희생하여 보다 fine-grained한 의사 결정을 내릴 수 있습니다. 또한, 생성된 코드의 memory-usage와 최적화 level 간의 trade-off도 있습니다.(machine 코드와 bytecode) 그렇기 때문에 JavaScript 엔진은 **hot**한 function만 최적화하려고 합니다._

## Optimizing prototype property access

[우리의 이전 글에서](https://mathiasbynens.be/notes/shapes-ics#optimizing-property-access) JavaScript엔진이 Shape와 Inline cache를 사용해서 어떻게 object property load를 최적화 했는지 설명하였습니다. 다시 상기해보면, 엔진은 object의 값과 별도로 object의 <kbd>Shape</kbd>를 저장합니다.

<figure class="align-center">
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_optimizing_prototypes/9_shape-2.svg" alt="9">
    <figcaption>shape</figcaption>
</figure>

Shape는 Inline Caches 혹은 ICs 라 불리는 최적화를 가능하게 합니다. Shape 및 ICs를 코드의 동일한 위치에서 property에 반복해서 접근할 때 속도를 높일 수 있습니다.

<figure class="align-center">
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_optimizing_prototypes/10_ic-4.svg" alt="10">
    <figcaption>Inline Cache</figcaption>
</figure>

### Classes and prototype-based programming

이제 JavaScript object의 property를 빨리 가져오는 방법에 대해서 알았으니, JavaScript에서 최근에 추가된 **Class**에 대해서 살펴보겠습니다. 아래의 코드는 JavaScript의 class syntax입니다.

```js
class Bar {
	constructor(x) {
		this.x = x;
	}
	getX() {
		return this.x;
	}
}
```

마치 JavaScript에서 새로운 개념이 나온것처럼 보이지만, 단지 JavaScript에서 계속 사용한 prototype-based programming을 위한 syntactic sugar일 뿐입니다.

```js
function Bar(x) {
	this.x = x;
}

Bar.prototype.getX = function getX() {
	return this.x;
};
```

여기 <kbd>Bar.prototype</kbd> object에 <kbd>getX</kbd> property를 할당합니다. 이 작업은 다른 어떤 object에서도 똑같이 일어납니다. **JavaScript에서 prototype은 object일 뿐**이기 때문입니다. JavaScript와 같은 prototype-based programming language에서는, method는 prototype을 통해 공유되고 field는 실제 instance에 저장됩니다.

<kbd>Bar</kbd>의 새로운 instance인 <kbd>foo</kbd>가 생성될 때 뒤에선 어떤 일이 일어나는지 살펴보겠습니다.

```js
const foo = new Bar(true);
```

위의 코드를 통해서 생성된 instance는 <kbd>'x'</kbd> property 가 들어있는 shape를 가진다. <kbd>foo</kbd>의 prototype인 <kbd>Bar.prototype</kbd>은 <kbd>Bar</kbd> class에 속해있다.

<figure class="align-center">
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_optimizing_prototypes/11_class-shape-1.svg" alt="11">
    <figcaption>class shape 1</figcaption>
</figure>

<kbd>Bar.prototype</kbd>은 단일 property <kbd>'getX'</kbd>를 포함하는 shape를 가지고 있으며, <kbd>getX</kbd>의 값은 호출시 <kbd>this.x</kbd>를 반환하는 함수입니다. <kbd>Bar.prototype</kbd>의 prototype은 <kbd>Object.prototype</kbd>이고, 이는 JavaScript 언어의 한 부분입니다. <kbd>Object.prototype</kbd>은 prototype tree의 root이기 때문에 이것의 prototype은 <kbd>null</kbd>입니다.

<figure class="align-center">
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_optimizing_prototypes/12_class-shape-2.svg" alt="12">
    <figcaption>class shape 2</figcaption>
</figure>

만약 같은 class의 다른 instance를 만든다면, 두 instance는 앞서 살펴본것과 같이 object shape를 공유합니다. 그리고 두 instance 모두 같은 <kbd>Bar.prototype</kbd> object를 가리킬것입니다.

### Prototype property access

자, 이제 class를 선언하고 새로운 instance를 생성할 때 어떤 일이 일어나는지 알게 되었습니다. 하지만, 아래 코드와 같이 instance의 method를 호출할 때는 어떻게 될까요?

```js
class Bar {
	constructor(x) { this.x = x; }
	getX() { return this.x; }
}

const foo = new Bar(true);
const x = foo.getX();
//        ^^^^^^^^^^
```

모든 method 호출을 두 가지 개별 단계로 생각할 수 있습니다.

```js
const x = foo.getX();

// is actually two steps:

const $getX = foo.getX;
const x = $getX.call(foo);
```

첫 번째 단계는 prototype의 property일 뿐인 method를 가져오는 것입니다(이러한 값이 function일 수 있습니다).
두 번째 단계는 instance를 <kbd>this</kbd>로 사용해서 function을 call하는 것입니다. Instance <kbd>foo</kbd>에서 method <kbd>getX</kbd>를 가져오는 첫 번째 단계를 살펴보겠습니다.

<figure class="align-center">
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_optimizing_prototypes/13_method-load.svg" alt="13">
    <figcaption>method load</figcaption>
</figure>

Engine은 <kbd>foo</kbd> instance에서 시작하고, <kbd>foo</kbd>의 shape에 <kbd>'getX</kbd>' property가 없다는 것을 알아챕니다. 그래서 engine은 prototype chain을 타고 올라간다. <kbd>Bar.prototype</kbd>에 도달하여, prototype의 shape를 봤을 때 offset <kbd>0</kbd>에 <kbd>'getX'</kbd> property가 있는 것을 알 수 있습니다. <kbd>Bar.prototype</kbd>에서 앞서 찾은 offset의 값을 보면 우리가 찾던 <kbd>JSFunction</kbd> <kbd>getX</kbd>를 찾을 수 있습니다.

JavaScript의 유연함(flexibility)덕분에 prototype chain link를 다음과 같이 변형할 수 있습니다.

```js
const foo = new Bar(true);
foo.getX();
// → true

Object.setPrototypeOf(foo, null);
foo.getX();
// → Uncaught TypeError: foo.getX is not a function
```

위의 예제에서 <kbd>foo.getX</kbd>를 2번 호출했지만, 각각의 호출은 완전히 다른 의미와 결과를 가져옵니다. 그렇기 때문에 prototype은 JavaScript에서 object일 뿐이지만, 일반 object의 자체(own) property에 접근하는 속도를 높이는 것 보다 prototype의 property에 접근하는 속도를 높이는 것이 더 어렵습니다. 

ing...
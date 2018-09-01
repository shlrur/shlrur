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
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_optimizing_prototypes/.png" alt="1">
    <figcaption></figcaption>
</figure>

또한 엔진 간에 high-level의 pipeline은 유사하지만 optimization pipeline에는 종종 차이가 있음을 지적했습니다. 왜 그런 것일까요? **일부 엔진의 최적화 tier가 다른 엔진보다 많은 이유는 무엇일까요?** 그것은 아래의 그림과 같이, 코드를 빨리 만들어서 실행하는 것과 시간을 더 들이지만 결국 최적화된 성능으로 코드를 만들어서 실행하는 것 사이의 trade-off가 있기 때문입니다.

<figure class="align-center">
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_optimizing_prototypes/.png" alt="2">
    <figcaption></figcaption>
</figure>

Interpreter는 bytecode를 재빨리 만들 수 있지만, 일반적으로 bytecode는 그리 효율적이지 않습니다. 반면에 optimizing compiler는 시간이 좀 더 걸리지만 결국 훨씬 더 효율적인 machine 코드를 생산합니다.

아래의 그림이 바로 V8이 사용하는 model입니다. V8 interpreter는 Ignition이라고 불리며, 모든 JavaScript 엔진의 interpreter 중에서 가장 빠릅니다(raw bytecode 실행 측면에서) V8의 optimizing compiler이름은 TurboFan이며, 고도로 최적화된 machine 코드를 생성합니다.

<figure class="align-center">
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_optimizing_prototypes/.png" alt="3">
    <figcaption></figcaption>
</figure>

이러한 startup latency와 execution speed 간의 trade-off는 일부 JavaScript 엔진이 최적화 tier를 추가하기로 선택하는 이유입니다. 예를 들어 SpiderMonkey는 interpreter와 전체 IonMonkey 사이에 compiler를 최적화하는 Baseline tier를 추가했습니다.

<figure class="align-center">
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_optimizing_prototypes/.png" alt="4">
    <figcaption></figcaption>
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
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_optimizing_prototypes/.png" alt="5">
    <figcaption></figcaption>
</figure>

Optimizer가 실행되는 동안, V8은 Ingition에서 bytecode를 실행합니다. 어느 시점에서 optimizer가 완료되고 나면 실행 가능한 machine 코드가 생성되며, 이 machine 코드로 계속해서 실행할 수 있습니다.

SpiderMonkey engine 역시 interpreter에서 bytecode를 실행하기 시작합니다. 그러나 Baseline tier가 추가되어 hot 코드는 먼저 Baseline으로 전송됩니다. Baseline compiler는 메인 thread에서 Baseline 코드를 생성하고 준비가되면 실행을 계속합니다.

<figure class="align-center">
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_optimizing_prototypes/.png" alt="6">
    <figcaption></figcaption>
</figure>

한동안 Baseline 코드를 실행하면 SpiderMonkey는 결국 IonMonkey frontend를 실행하고 V8과 매우 유사한 optimizer를 시작합니다. IonMonkey가 최적화하는 동안 Baseline에서 실행됩니다. 마지막으로 optimizer가 완료되면, Baseline 코드 대신 최적화된 code를 실행합니다.

Chakra의 구조는 SpiderMonkey와 매우 비슷하지만, Chakra는 메인 thread를 막지 않기 위해 더 많은 것들을 동시에 실행하려고 시도합니다. 메인 Thread에서 compiler의 일부를 실행하는 대신, Chakra는 compiler가 필요로 하는 bytecode와 profiling data를 복사하여 전용 compiler 프로세스로 보냅니다.

<figure class="align-center">
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/javascript_engine_fundamentals_optimizing_prototypes/.png" alt="7">
    <figcaption></figcaption>
</figure>

생성된 코드가 준비되면, Chakra 엔진은 bytecode 대신 이 SimpleJIT 코드를 실행합니다. FullJIT에 대해서도 같은 과정이 실행됩니다. 이러한 접근 방식의 이점은 복사가 발생하는 일시 중지 시간이 보통 전체 compiler가(frontend)를 실행하는 데 비해 훨씬 짧다는 것입니다. 그러나 이러한 접근 방식의 단점은 **copy heuristic** 이 특정 최적화에 필요한 일부 정보를 놓칠 수 있다는 것입니다. 따라서 latency를 위해 코드 품질을 어느 정도 타협한 것입니다.


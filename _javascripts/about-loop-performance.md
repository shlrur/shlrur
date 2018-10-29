---
title: "About JavaScript Loop Performance(forEach, for, while, etc...)"
date: 2018-10-18
categories:
  - JavaScript
tags:
  - JavaScript
  - loop
  - performance
  - forEach
  - for
  - while
  - for of
  - lodash
  - underscore
  - Codility
last_modified_at: 2018-10-22T01:35:00+09:00
---

<link href="https://cdnjs.cloudflare.com/ajax/libs/c3/0.6.8/c3.min.css" rel="stylesheet">
<script src="https://cdnjs.cloudflare.com/ajax/libs/d3/5.7.0/d3.min.js" charset="utf-8"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/c3/0.6.8/c3.min.js"></script>
<style type="text/css">
.tg  {border-collapse:collapse;border-spacing:0;}
.tg .tg-s6z2{text-align:center}
</style>

최근 Codility에서 코딩 연습을 하다가 정리해야 할 것이 있어서 포스트를 작성하기로 하였습니다.

연습했던 문제는 [MaxCounters](https://app.codility.com/programmers/lessons/4-counting_elements/max_counters/) 였고, 해당 문제의 난이도는 어렵지 않았습니다. 하지만 계속해서 Performance tests에서 score가 만점이 나오지 않아, 이것저것 시도해본 경험을 기록하고자 합니다.

## Problems raised

아래의 코드는 Performance tests에서 100% score를 기록한 코드입니다.

```js
function solution(N, A) {
    let answer = [];
    let i, j;
    let ALength = A.length;
    let maxValue=0;
    
    let isMaxing = false;
    
    for(i=0 ; i<N ; i++) {
        answer[i] = 0;
    }
    
    for(i=0 ; i<ALength ; i++) {
        if(A[i] === N+1) {
            if(!isMaxing) {
                for(j=0 ; j<N ; j++) {
                    answer[j] = maxValue;
                }
                
                isMaxing = true;
            }
        } else {
            answer[A[i]-1] += 1;
            
            maxValue = maxValue < answer[A[i]-1] ? answer[A[i]-1] : maxValue;
            
            isMaxing = false;
        }
    }
    
    return answer;
}
```

그리고 아래의 코드는 Performance tests에서 100%를 기록하지 못한 코드입니다.

```js
function solution(N, A) {
    let answer = [];
    let i;
    let maxValue=0;
    
    let isMaxing = false;
    
    for(i=0 ; i<N ; i++) {
        answer[i] = 0;
    }
    
    A.forEach((a) => {
        if(a === N+1) {
            if(!isMaxing) {
                for(i=0 ; i<N ; i++) {
                    answer[i] = maxValue;
                }
                
                isMaxing = true;
            }
        } else {
            answer[a-1] += 1;
            
            maxValue = maxValue < answer[a-1] ? answer[a-1] : maxValue;
            
            isMaxing = false;
        }
    });
    
    return answer;
}
```

두 코드의 time complexity는 **O(N+M)**으로 동일합니다. 두 코드의 차이는 loop에서 **for**를 사용했는지 Array.prototype 의 **forEach**를 사용했는지 입니다. Performance tests가 만점이 나오지 않아서 코드의 로직을 이리저리 바꿔보았지만, 단순한 문제에서 로직을 획기적으로 개선할 수 있는 부분이 없었습니다. 혹시나 하는 마음에 forEach를 for로 바꾼 후에 Performance tests를 100%로 통과할 수 있었습니다. 약간의 허무함과 함께 Javascript의 loop performance를 테스트하지 않을 수 없었습니다.

## Tests of Loop Performance
Loop performance를 테스트하기 위한 실험에 아래의 6가지 loop를 사용하였습니다.

* for
* for of(ES6 구문)
* while
* Array의 forEach
* lodash(v4.17.10)의 forEach
* underscore(v1.9.1)의 each

_(Array의 map, filter, reduce등 다른 Array.prototype의 함수들도 loop를 사용하지만 함수의 목적이 다르기때문에 테스트에서 사용하지 않았습니다.)_ 각각의 loop 내에서 숫자 덧셈, array push, string split에 대한 bench mark 결과는 다음과 같습니다. 값의 단위는 **Ops/sec**로 초당 실행한 operation 개수입니다. 큰 숫자일수록 좋습니다.

|              | for       | for of | while     | forEach(native) | forEach(lodash) | each(underscore)     |
| ------------ | --------- | ------ | --------- | --------------- | --------------- | -------------------- |
| Number Add   | 38845     | 23831  | **39871** | 3924            | 3470            | **3289**             | 
| Array Push   | **3335**  | 3147   | 3149      | 2054            | 1795            | **1731**             |
| String Split | 992       | 884    | **1034**  | 659             | 645             | **664**              |

<div id="chart1"></div>

_실험 결과: [jsperf](https://jsperf.com/shlrur)_

실험 결과는 대체적으로
**while >= for > for of > forEach(native) > forEach(loadsh) > each(underscore)**
의 순으로 집계되었습니다.

While과 for가 가장 높은 성능을 기록하였고, 그 다음이 for of, 그리고 나머지 3가지인 forEach(native), forEach(lodash) 그리고 each(underscore)가 비슷한 성능을 기록하였습니다.

## Comparison of Loop functions

While과 for는 가장 단순한 로직을 가지는 loop인 만큼 가장 높은 성능을 기록하였습니다. 단순한 로직이지만, 대상 array에 index를 사용해서 접근해야 하는 단점을 가지고 있습니다.(e.g.: Arr[i])

나머지 for of, forEach(native), forEach(lodash) 그리고 each(underscore)는 loop 내에서 배열에서 처리될 현재 요소의 값을 제공합니다.(e.g.: Array.prototype.forEach의 callback function에서 첫 번째 argument) 그리고 for of를 제외한 나머지 3가지는 break를 할 수 없다는 단점을 가집니다. 하지만 while과 for에 비해서 높은 가독성 및 유지보수에 유리합니다.

...ing

## Async for Loop Performance

Loop의 performance를 향상시키는 방법에 대해서 조사하는 중에, [JavaScript, Node.js: is Array.forEach asynchronous?](https://stackoverflow.com/questions/5050265/javascript-node-js-is-array-foreach-asynchronous)라는 StackOverflow 질문을 봤습니다. 

```js
[many many elements].forEach(function () {lots of work to do});
```

위와같은 코드에서, forEach내의 작업들이 asynchronously하게 수행되는지에 대한 질문이었습니다. 답변에서는 다음과 같이 이야기 합니다.

> 일반적인 JavaScript의 loop는 asynchronously하게 작동하지 않으며 non-blocking이 아닌 blocking이다.

당연히, 알고있던 내용입니다. 하지만, asynchronously하게 작동하는 방법이 있습니다.

### Asynchronous & Non-Parallel

[JavaScript loops — how to handle async/await](https://blog.lavrton.com/javascript-loops-how-to-handle-async-await-6252dd3c795) article에 나오는 방식에 따라서, loop를 asynchronously하게 작동하는 방식을 구현해 보았습니다.

_이하의 모든 코드는 nodejs(v10.9.0)에서 실행하였습니다._

```js
async function asyncTest() {
    const ARRAY_NUM = 100;
    const PROCESS_SIZE = 10000000;

    // ----- prepared code -----
    async function delyedLog(item) {
        // console.log('async start: ', item);
        await bigProcess(item);
        // console.log('async end: ', item);
    }
    
    async function processArray(array) {
        const promises = array.map(delyedLog);
        
        await Promise.all(promises);
    }
    
    function bigProcess(item) {
        let sum = 0;
        for (let i = 0; i < PROCESS_SIZE; i++) { sum += i; }
        return Promise.resolve(sum);
    }
    
    let bigArray = [];
    for (let i = 0; i < ARRAY_NUM; i++) { bigArray.push(i); }
    // -------------------------
    
    // ----- blocking code -----
    console.time('block');
    for (let i = 0, N = bigArray.length; i < N; i++) {
        // console.log('block start: ', i);
        bigProcess(i);
        // console.log('block end: ', i);
    }
    console.timeEnd('block');
    // -------------------------
    
    // ------ async code -------
    console.time('async');
    await processArray(bigArray);
    console.timeEnd('async');
    // -------------------------
}

asyncTest();
```

위의 코드는, array의 크기와(ARRAY_NUM) loop에서 실행되는 process의 크기(PROCESS_SIZE)에 따라서, blobking code와 async code의 실행 시간을 나타냅니다.

아래의 표는 ARRAY_NUM과 PROCESS_SIZE에 대한 bench mark 결과입니다. 값의 단위는 **ms**로 실행에 걸린 시간입니다. 작은 숫자일수록 좋습니다.

| ARRAY_NUM | PROCESS_SIZE | Block TIME | Async Time |
| --------: | -----------: | ---------: | ---------: |
| 10        | 100,000,000  | 1593.491   | 1554.637   |
| 10        | 10,000,000   | 140.981    | 143.005    |
| 10        | 1,000,000    | 20.618     | 15.429     |
| 100       | 10,000,000   | 1470.235   | 1401.06    |
| 100       | 1,000,000    | 145.066    | 143.921    |
| 100       | 100,000      | 20.713     | 18.944     |
| 1,000     | 1,000,000    | 1329.18    | 1340.667   |
| 1,000     | 100,000      | 142.233    | 149.983    |
| 1,000     | 10,000       | 12.371     | 12.249     |
| 10,000    | 100,000      | 1379.174   | 1389.454   |
| 10,000    | 10,000       | 91.379     | 111.287    |
| 10,000    | 1,000        | 14.377     | 33.429     |
| 100,000   | 10,000       | 854.679    | 1083.654   |
| 100,000   | 1,000        | 97.899     | 358.167    |
| 100,000   | 100          | 16.493     | 238.419    |

아래의 차트들은 ARRAY_NUM(AN)과 PROCESS_SIZE(PS)의 곱(총 volumn)에 따라서 benchmark한 결과입니다.

[총 Volumn: 1,000,000,000]
<div id="chart2"></div>

[총 Volumn: 100,000,000]
<div id="chart3"></div>

[총 Volumn: 10,000,000]
<div id="chart4"></div>

그리고 다음은 상위 코드에서 주석 처리된 console.log()들을 살린 후, AN=5로 했을 때 log 결과입니다.

```
block start:  0
block end:  0
block start:  1
block end:  1
block start:  2
block end:  2
block start:  3
block end:  3
block start:  4
block end:  4
block: 44.513ms
async start:  0
async start:  1
async start:  2
async start:  3
async start:  4
async end:  0
async end:  1
async end:  2
async end:  3
async end:  4
async: 27.606ms
```

위의 결과들을 보면, async 코드는 asynchronous하게 작동하고 있습니다. Block 코드와 다르게 각 loop내의 코드가 독립적으로 실행 및 종료되는 것을 볼 수 있습니다. 하지만, 위의 표와 그래프들을 보면 asynchronous하게 작동하는것이 성능에 크게 영향을 주지는 않습니다.(for문 내에서 처리하는 양이 적을때는, 일반적인 for문에 비해서 Array.prototype.map과 Promise.all을 사용하는데 부하가 더 큰 것을 볼 수 있습니다.) 그 이유는 JavaScript가 asynchronous이며 **Single Thread**이기 때문에, block이든 async든 한 thread가 정해진 처리량을 다 맡아야 하기 때문입니다. 처리량을 줄이기 위해서는 asynchronous함과 동시에 parallel한 처리가 필요합니다.

### Asynchronous & Parallel

JavaScript는 기본적으로 asynchronous & single thread 입니다. Web application이라는 말이 나올 정도로, web에서 많은 처리량이 요구되고 있습니다. Google의 V8 엔진을 가지는 nodejs를 사용하여 server를 구축하기도 합니다. 이에 따라서, single thread가 아닌 multi thread를 사용하는 방법이 생겨나고 있습니다.

[Web Worker](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API)는 JavaScript에서 multi-thread를 사용할 수 있게 합니다.(DOM을 다룰 수는 없습니다.) 그리고 Microsoft 사에서 만든 [napa.js](https://github.com/Microsoft/napajs)라는 API가 있습니다.

Napa.js는 **zone**이라는 개념을 사용합니다. Zone은 지정된 code block을 실행할 수 있습니다.(하나의 zone은 여러개의 code block을 실행할 수 없습니다. 여러 code block을 실행하기 위해서는 여러 zone이 필요합니다) Zone은 여러개의 worker들로 구성됩니다.(JavaScript의 worker. 각각의 worker는 각자 thread를 가집니다.) 프로세스는 여러 zone을 가질 수 있습니다. Napa.js에는 각각의 특징을 가지는 두 가지 type의 zone이 있습니다.

* Napa zone
* Node zone

<figure class="align-center">
    <img src="{{ site.url }}{{ site.baseurl }}/assets/images/about-loop-performance/0_napajs_architecture.png" alt="0">
    <figcaption>Napa.js architecture</figcaption>
</figure>

Napa.js에서 사용하는 각각의 worker는 자신만의 heap space를 가집니다. 한 worker에서 다른 worker에게 value들을 serialized/deserialized 로 전송할 수 있습니다. Napa.js에서는 zone의 기능인 'broadcast', 'excute' 그리고 Store API를 사용해서 worker간에 data를 공유할 수 있습니다. 이 작업을 수행할 때 마다 data는 각 thread의 heap space로 복사됩니다. 향후에는, 복사하지 않고 data를 공유하기 위한 memory sharing pattern을 계획하고 있습니다. 이러한 memory sharing pattern은 JavaScript에서 고성능 multi-threading 솔루션의 초석이 될 것입니다.

Napa.js를 사용해서 parallel 코드의 performance를 test해보겠습니다.
아래의 두 코드는 _Matrix(N*N) X Vector(1*N)_ 를 계산합니다. N은 실행 시 입력받습니다.

Serial 코드는 일반적인 for문을 사용하고, Parallel 코드는 하나의 zone에 NUMBER_OF_WORKERS만큼의 worker들이 parallel하게 계산합니다.

```js
/* serial version */

let size;
if (process.argv[2]) {
    size = parseInt(process.argv[2]);
    if (size !== size) {
        console.log('size should be an integer');
        process.exit(1);
    }
} else {
    console.log('size should be provided');
    process.exit(1);
}

const matrix = [];
const vector = [];

for (let i = 0; i < size; i++) {
    matrix.push([]);
    for (let j = 0; j < size; j++) {
        matrix[i].push(getRandomInt(0, 100));
    }
}

for (let i = 0; i < size; i++) {
    vector.push([getRandomInt(0, 100)]);
}

console.log('Matrix:');
console.log(matrix);
console.log('Vector:');
console.log(vector);

const result = [];
const start = Date.now();
for (let i = 0; i < size; i++) {
    let value = 0;
    for (let j = 0; j < size; j++) {
        value += matrix[i][j] * vector[j][0];
    }

    result.push([value]);

    for (const currentTime = new Date().getTime() + 5; new Date().getTime() < currentTime;);  //looping to simulate some work
}

const end = Date.now();
console.log('Result:');
console.log(result);
console.log('Runtime: ' + (end - start) + 'ms');

function getRandomInt(min, max) {
    return Math.floor(Math.random() * (max - min + 1)) + min;
}
```

```js
/* parallel version */

const napa = require('napajs');
const NUMBER_OF_WORKERS = 8;

const zone = napa.zone.create('zone', { workers: NUMBER_OF_WORKERS });

let size;
if (process.argv[2]) {
    size = parseInt(process.argv[2]);
    if (size !== size) {
        console.log('size should be an integer');
        process.exit(1);
    }
} else {
    console.log('size should be provided');
    process.exit(1);
}

const matrix = [];
const vector = [];

for (let i = 0; i < size; i++) {
    matrix.push([]);
    for (let j = 0; j < size; j++) {
        matrix[i].push(getRandomInt(0, 100));
    }
}

for (let i = 0; i < size; i++) {
    vector.push([getRandomInt(0, 100)]);
}

console.log('Matrix:');
console.log(matrix);
console.log('Vector:');
console.log(vector);

function multiply(row, vector) {
    let result = 0;
    for (let i = 0; i < row.length; i++) {
        result += row[i] * vector[i][0];
    }

    for (const currentTime = new Date().getTime() + 5; new Date().getTime() < currentTime;);  //looping to simulate some work

    return result;
}

var promises = [];

const start = Date.now();

for (var i = 0; i < size; i++)
    promises[i] = zone.execute(multiply, [matrix[i], vector]);

Promise.all(promises).then((results) => {
    const end = Date.now();
    console.log('Result:');
    console.log(results.map(result => [parseInt(result._payload)]));
    console.log('Runtime: ' + (end - start) + 'ms');
});

function getRandomInt(min, max) {
    return Math.floor(Math.random() * (max - min + 1)) + min;
}
```

Matrix의 크기와 work의 개수에 따른 performance 결과는 다음과 같습니다.

<table class="tg">
  <tr>
    <th class="tg-s6z2" rowspan="2">Matrix Size</th>
    <th class="tg-s6z2" rowspan="2">serial Runtime(ms)</th>
    <th class="tg-s6z2" colspan="4">parallel Runtime(ms) by worker num</th>
  </tr>
  <tr>
    <td class="tg-s6z2">1 worker</td>
    <td class="tg-s6z2">2 worker</td>
    <td class="tg-s6z2">4 worker</td>
    <td class="tg-s6z2">8 worker</td>
  </tr>
  <tr>
    <td class="tg-s6z2">1</td>
    <td class="tg-s6z2">5</td>
    <td class="tg-s6z2">10</td>
    <td class="tg-s6z2">11</td>
    <td class="tg-s6z2">10</td>
    <td class="tg-s6z2">10</td>
  </tr>
  <tr>
    <td class="tg-s6z2">5</td>
    <td class="tg-s6z2">27</td>
    <td class="tg-s6z2">32</td>
    <td class="tg-s6z2">19</td>
    <td class="tg-s6z2">15</td>
    <td class="tg-s6z2">13</td>
  </tr>
  <tr>
    <td class="tg-s6z2">10</td>
    <td class="tg-s6z2">56</td>
    <td class="tg-s6z2">58</td>
    <td class="tg-s6z2">32</td>
    <td class="tg-s6z2">22</td>
    <td class="tg-s6z2">20</td>
  </tr>
  <tr>
    <td class="tg-s6z2">50</td>
    <td class="tg-s6z2">253</td>
    <td class="tg-s6z2">258</td>
    <td class="tg-s6z2">133</td>
    <td class="tg-s6z2">74</td>
    <td class="tg-s6z2">46</td>
  </tr>
  <tr>
    <td class="tg-s6z2">100</td>
    <td class="tg-s6z2">503</td>
    <td class="tg-s6z2">516</td>
    <td class="tg-s6z2">262</td>
    <td class="tg-s6z2">139</td>
    <td class="tg-s6z2">90</td>
  </tr>
  <tr>
    <td class="tg-s6z2">500</td>
    <td class="tg-s6z2">2503</td>
    <td class="tg-s6z2">3169</td>
    <td class="tg-s6z2">1639</td>
    <td class="tg-s6z2">886</td>
    <td class="tg-s6z2">532</td>
  </tr>
</table>

<div id="chart5"></div>

위의 benchmark 결과를 볼 때, napa.js에 의해서 parallel하게 실행 되었음을 알 수 있습니다. 그리고, parallel 구조가 serial 구조에 비해서 성능이 좋다는 것과, parallel 구조에서 worker가 많을수록 성능이 높게 나왔음을 알 수 있습니다.(위의 결과에 기록하지는 않았지만, worker의 수가 논리 프로세서의 개수보다 많아졌을 때는 성능의 향상이 거의 없었습니다. 당연한 일입니다.)

## References

* [JavaScript, Node.js: is Array.forEach asynchronous?](https://stackoverflow.com/questions/5050265/javascript-node-js-is-array-foreach-asynchronous)
* [JavaScript loops — how to handle async/await](https://blog.lavrton.com/javascript-loops-how-to-handle-async-await-6252dd3c795) - [Anton Lavrenov](https://lavrton.com)
* [High Performance JavaScript: Build Faster Web Application Interfaces 1st Edition](https://www.amazon.com/dp/059680279X/?tag=stackoverflow17-20) - Nicholas C. Zakas
* [Web Workers API](https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API)
* [napa.js](https://github.com/Microsoft/napajs)
* [Starting Parallel Programming In Node.js With Napa.js](https://medium.com/codeinsights/starting-parallel-programming-in-node-js-with-napa-js-ef80e20ec6c2)

<script type="text/javascript">
    var chart = c3.generate({
        bindto: '#chart1',
        data: {
            x: 'x',
            columns: [
                ['x', 'Number Add', 'Array Push', 'String Split'],
                ['for', 38845, 3335, 992],
                ['for of', 23831, 3147, 884],
                ['while', 39871, 3149, 1034],
                ['forEach(native)', 3924, 2054, 659],
                ['forEach(lodash)', 3470, 1795, 645],
                ['each(underscore)', 3289, 1731, 664]
            ],
            type: 'bar'
        },
        axis: {
            x: {
                type: 'category', // this needed to load string x value
                label: 'Test type'
            },
            y: {
                label: 'Ops/sec'
            }
        }
    });

    var chart = c3.generate({
        bindto: '#chart2',
        data: {
            x: 'x',
            rows: [
                ['x', 'Block', 'Async'],
                ['AN_10^1/PS_10^8', 1593.491, 1554.637],
                ['AN_10^2/PS_10^7', 1470.235, 1401.06],
                ['AN_10^3/PS_10^6', 1329.18, 1340.667],
                ['AN_10^4/PS_10^5', 1379.174, 1389.454],
                ['AN_10^5/PS_10^4', 854.679, 1083.654]
            ],
            type: 'bar'
        },
        axis: {
            x: {
                type: 'category', // this needed to load string x value
                label: 'Test type'
            },
            y: {
                label: 'ms'
            }
        }
    });

    var chart = c3.generate({
        bindto: '#chart3',
        data: {
            x: 'x',
            rows: [
                ['x', 'Block', 'Async'],
                ['AN_10^1/PS_10^7', 140.981, 143.005],
                ['AN_10^2/PS_10^6', 145.066, 143.921],
                ['AN_10^3/PS_10^5', 142.233, 149.983],
                ['AN_10^4/PS_10^4', 91.379, 111.287],
                ['AN_10^5/PS_10^3', 97.899, 358.167]
            ],
            type: 'bar'
        },
        axis: {
            x: {
                type: 'category', // this needed to load string x value
                label: 'Test type'
            },
            y: {
                label: 'ms'
            }
        }
    });

    var chart = c3.generate({
        bindto: '#chart4',
        data: {
            x: 'x',
            rows: [
                ['x', 'Block', 'Async'],
                ['AN_10^1/PS_10^6', 20.618, 15.429],
                ['AN_10^2/PS_10^5', 20.713, 18.944],
                ['AN_10^3/PS_10^4', 12.371, 12.249],
                ['AN_10^4/PS_10^3', 14.377, 33.429],
                ['AN_10^5/PS_10^2', 16.493, 238.419]
            ],
            type: 'bar'
        },
        axis: {
            x: {
                type: 'category', // this needed to load string x value
                label: 'Test type'
            },
            y: {
                label: 'ms'
            }
        }
    });

    var chart = c3.generate({
        bindto: '#chart5',
        data: {
            x: 'x',
            rows: [
                ['x', 'Serial', 'Parallel(1 worker)', 'Parallel(2 worker)', 'Parallel(4 worker)', 'Parallel(8 worker)'],
                [1, 5, 10, 11, 10, 10],
                [5, 27, 32, 19, 15, 13],
                [10, 56, 58, 32, 22, 20],
                [50, 253, 258, 133, 74, 46],
                [100, 503, 516, 262, 139, 90],
                [500, 2503, 3169, 1639, 886, 532]
            ],
            type: 'line'
        },
        axis: {
            x: {
                label: 'matrix size'
            },
            y: {
                label: 'runtime(ms)'
            }
        }
    });
</script>
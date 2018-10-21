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

_(Array의 map, filter, reduce등 다른 Array.prototype의 함수들도 loop를 사용하지만 함수의 목적이 다르기때문에 테스트에서 사용하지 않았습니다.)_ 각각의 loop 내에서 숫자 덧셈, array push, string split에 대한 bench mark 결과는 다음과 같습니다. 값의 단위는 Ops/sec로 초당 실행한 operation 개수입니다. 큰 숫자일수록 좋습니다.

|              | for       | for of | while     | forEach(native) | forEach(lodash) | each(underscore)     |
| ------------ | --------- | ------ | --------- | --------------- | --------------- | -------------------- |
| Number Add   | 38845     | 23831  | **39871** | 3924            | 3470            | **3289**             | 
| Array Push   | **3335**  | 3147   | 3149      | 2054            | 1795            | **1731**             |
| String Split | 992       | 884    | **1034**  | 659             | 645             | **664**              |

<div id="chart"></div>

_실험 결과: [jsperf](https://jsperf.com/shlrur)_

실험 결과는 대체적으로
**while >= for > for of > forEach(native) > forEach(loadsh) > each(underscore)**
의 순으로 집계되었습니다.

While과 for가 가장 높은 성능을 기록하였고, 그 다음이 for of, 그리고 나머지 3가지인 forEach(native), forEach(lodash) 그리고 each(underscore)가 비슷한 성능을 기록하였습니다.

## Comparison of Loop functions

While과 for는 가장 단순한 로직을 가지는 loop인 만큼 가장 높은 성능을 기록하였습니다. 단순한 로직이지만, 대상 array에 index를 사용해서 접근해야 하는 단점을 가지고 있습니다.(e.g.: Arr[i])

나머지 for of, forEach(native), forEach(lodash) 그리고 each(underscore)는 loop 내에서 배열에서 처리될 현재 요소의 값을 제공합니다.(e.g.: Array.prototype.forEach의 callback function에서 첫 번째 argument) 그리고 for of를 제외한 나머지 3가지는 break를 할 수 없다는 단점을 가집니다. 하지만 while과 for에 비해서 높은 가독성 및 유지보수에 유리합니다.

...ing

## References

* [ECMA-262-3 in detail. Chapter 5. Functions.](http://dmitrysoshnikov.com/ecmascript/chapter-5-functions/#question-about-surrounding-parentheses) - Dmitry A. Soshnikov


<script type="text/javascript">
    var chart = c3.generate({
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
</script>
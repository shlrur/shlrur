---
layout:     post
title:      "Constructor Design Pattern"
subtitle:   "Constructor Design Pattern"
categories: develog
tags:       javascriptdesignpattern
comments:   true
---

기존의 객체 지향 프로그래밍(OOP) 언어에서, 객체가 새로 생성되고 메모리가 할당될 때 이 객체를 초기화하는데 사용하는 특수한 method가 있습니다. 바로 생성자(Constructor)입니다.

생성자는 method이므로 인수(arguments)를 받을 수 있습니다. 이 인수들을 사용해서 처음 객체가 생성될 때 member property/method 들을 초기화할 수 있습니다.

>ES6(ES2015)부터 class를 사용할 수 있게 되었지만, 문법상의 요소일 뿐 JavaScript는 여전히 prototype 기반 언어입니다. 그러므로 prototype을 기반으로 한 Contructor design pattern에 대해서 알아보겠습니다.

## Basic Constructors

JavaScript는 객체의 생성을 위한 constructor를 특수한 함수(function)로써 제공합니다. Constructor 함수 앞에 <kbd>'new'</kbd> 키워드를 붙임으로써 해당 함수 내에서 정의하는 member들을 가지는 object를 만들고 초기화할 수 있습니다.

Constructor 내에서 <kbd>'this'</kbd>가 붙은 변수 혹은 함수들은 새로 생성된 instance가 가지는 member들이 됩니다.

```js
function SmartPhone( name, company, cost ) {
  this.name = name;
  this.company = company;
  this.cost = cost;
 
  this.toString = function () {
    return `${this.company} of ${this.name} is ${this.cost}won.`;
  };
}

var galaxyZ121 = new SmartPhone( 'GalaxyZ121', 'SamSong', 1000000 );
var iphong99 = new SmartPhone( 'iPhong99', 'Aple', 1200000 );

console.log( galaxyZ121.toString() );
console.log( iphong99.toString() );
```

위의 코드는 간단한 Constructor pattern을 보여주고 있습니다.
SmartPhone 함수가 Constructor이며, 이를 통해 생성된 instance는 name, company, cost, (member field) 그리고 toString(member method)를 가집니다. <kbd>'new'</kbd> 키워드를 통해서 instance를 생성할 수 있으며 사용할 수도 있습니다.
하지만 위의 방식은 좋은 방식이 아닙니다. <kbd>toString</kbd> 함수는 모든 instance에서 같은 역할, 같은 코드를 가지지만 모든 instance의 member method가 되기 때문입니다. SmartPhone constructor를 통해서 만들어진 모든 instance가 하나의 toString 함수를 공유한다면 좋을 것 같습니다. 이때 사용되는 것이 **Prototype** 입니다.

## Constructors With Prototypes

JavaScript의 거의 모든 object와 function은 <kbd>"prototype"</kbd> object를 가집니다. Constructor통해 instance를 만들면, constructor의 prototype이 가지는 property까지 instance에서 사용할 수 있게 됩니다.

```js
function SmartPhone( name, company, cost ) {
  this.name = name;
  this.company = company;
  this.cost = cost;
}

SmartPhone.prototype.toString = function () {
  return `${this.company} of ${this.name} is ${this.cost}won.`;
};

var galaxyZ121 = new SmartPhone( 'GalaxyZ121', 'SamSong', 1000000 );
var iphong99 = new SmartPhone( 'iPhong99', 'Aple', 1200000 );

console.log( galaxyZ121.toString() );
console.log( iphong99.toString() );
```

위의 코드는 더 위의 코드와 실행 결과는 같지만 내부적으로는 다르게 동작합니다. toString 함수를 SmartPhone의 prototype에 할당함으로써 모든 instance가 같은 toString을 공유할 수 있게 되었습니다.

## references
* [Learning JavaScript Design Patterns](https://addyosmani.com/resources/essentialjsdesignpatterns/book/#modulepatternjavascript)
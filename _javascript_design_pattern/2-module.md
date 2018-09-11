---
title: "Module Design Pattern"
date: 2018-09-06
categories:
  - JavaScript Design Pattern
tags:
  - JavaScript
  - designpattern
  - module
last_modified_at: 2018-09-10T17:22:00+09:00
---

소프트웨어 개발에서 소스코드는 특정 기능을 수행하거나(function), 특정 작업을 수행하는데 필요한 모든 것을 포함하는 구성요소(object)로 구성할 수 있습니다. Modular programming은 이러한 개념을 사용한 프로그래밍 방법입니다.

소프트웨어 공학에서 말하는 **module design pattern**은 modular programming에서 의미하는 module 개념이 지원되지 않는 언어에서 module을 사용하기 위해서 사용하는 pattern입니다. Module 이라는 개념은 모든 언어에서 지원되지 않기 때문입니다.

여러 종류의 언어에서 module design pattern을 구현하는 방법들이 있지만, 이 글에서는 Prototype-based language인 JavaScript에서의 module design pattern에 대해서 알아보겠습니다.

## Module

>Modules are an integral piece of any robust application's architecture and typically help in keeping the units of code for a project both cleanly separated and organized.

[Learning JavaScript Design Patterns](https://addyosmani.com/resources/essentialjsdesignpatterns/book/#modulepatternjavascript) 에서는 module을 위와 같이 정의합니다.

이를 번역하면 다음과 같습니다.

>Module은 강력한 애플리케이션 아키텍처의 필수적인 요소이며, 일반적으로 프로젝트의 코드 단위를 깨끗하게 분리하여 구성하는 데 도움이 됩니다.

위의 정의에서 가장 중요한 부분은 _프로젝트의 코드 단위를 깨끗하게 **분리**하여 구성_ 한다는 것입니다. JavaScript에서의 module은 **Class**라고 볼 수 있습니다. OOP에서 말하는 class의 특징중 하나는 encapsulation인데, 이는 특정 class의 member들을 다른 class에서 접근하지 못하도록 보호합니다. Module pattern은 ES6이전 class가 지원되지 않던 JavaScript에서도 class의 member에 대한 private과 public 설정을 가능하게 하였습니다.



## references
* [Learning JavaScript Design Patterns](https://addyosmani.com/resources/essentialjsdesignpatterns/book/#modulepatternjavascript)
* [Wiki - Module Pattern](https://en.wikipedia.org/wiki/Module_pattern)
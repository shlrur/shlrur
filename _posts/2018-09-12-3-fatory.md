---
title: "Fatory Design Pattern"
date: 2018-09-12
categories:
  - JavaScript Design Pattern
tags:
  - JavaScript
  - designpattern
  - factory
last_modified_at: 2018-09-12T19:01:00+09:00
---

Factory pattern은 object를 만들 때 사용하는 pattern입니다. Object를 만드는 다른 pattern들과는 다르게 constructor가 꼭 필요하진 않습니다. 대신 object를 생성할 때 사용하는 interface를 제공해야 하며, 이 interface에서 object의 type을 지정할 수 있어야 합니다.

Component type에 따라 UI를 만들어주는 UI 생성기가 있다고 해보겠습니다. <kbd>new</kbd> 키워드나 constructor를 사용해서 바로 component를 만들지 말고, Factory pattern을 사용해서 만들어보려 합니다. 이때 일어나는 작업을 간단히 말하자면, 우리가 필요한 component의 type을 Factory pattern에게 알려주고, 그 type에 따라서 instance를 만든 후 return 하게 됩니다.

Factory pattern은 object의 생성이 복잡할 때 유용하게 사용됩니다. JavaScript의 Object()를 사용해서 object를 생성할 때, 주어지는 값이 Number, String, 그리고 Boolean 등등 runtime에 type이 주어지는 경우가 많습니다. 이런 경우에 특히 유용하게 사용할 수 있습니다.

아래의 예제는 Factory pattern을 사용한 것입니다.

```js
// Dog constructor
function Dog(options) {
    this.size = options.size || 'medium';
    this.species = options.species || 'Jindo';
    this.color = options.color || 'white';
}

// Cat constructor
function Cat(options) {
    this.species = options.species || 'Korean Shorthair';
    this.hair = options.hair || 'short';
    this.color = options.color || 'brown';
}

// Define a skeleton pet factory
function PetFactory() { }

// default petType is Dog
PetFactory.prototype.PetType = Dog;

// IMPORTANT Function!!
// Factory method for creating new Pet instance
PetFactory.prototype.createPet = function (options) {
    switch (options.PetType) {
        case 'dog':
            this.PetType = Dog;
            break;
        case 'cat':
            this.PetType = Cat;
            break;
    }

    return new this.PetType(options);
};

// Create an instance of factory that makes dog
var dogFactory = new PetFactory();
var dodo = dogFactory.createPet({
    PetType: 'dog',
    color: 'black',
    size: 'small',
    species: 'Toy Poodle'
});

// true
console.log(dodo instanceof Dog);

// Dog object with the species is 'Toy Poodle', color is 'black' and size is 'small'
console.log(dodo);
```

### Factory pattern을 사용해야 할 때

Object를 생성하는 logic이 복잡할 때 해당 pattern을 사용할 수 있습니다. 그리고 당연히, 서로 다르지만 공통적으로 많은 특성(same properties)을 가지는 object들을 관리, 유지 또는 사용할 때 많은 장점을 가질 수 있습니다.

### Factory pattern을 사용하지 말아야 할 때

해당 pattern을 사용 할 상황이 아닐 때는 application에 더욱 큰 복잡성을 줄 수 있습니다.

...ing

## references
* [Learning JavaScript Design Patterns](https://addyosmani.com/resources/essentialjsdesignpatterns/book/#factorypatternjavascript)
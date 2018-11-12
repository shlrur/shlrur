---
layout:     post
title:      "[React] Immutable.js: Using Immutable Data Structures"
subtitle:   "DOM operation"
categories: develog
tags:       api
comments:   true
---

React에서 performance를 높이는 방법은 여러가지가 있습니다.

### Use the Production Build

React를 통한 사이트를 제작한 후 배포할 때 production 모드로 build를 하게 되면, 여러 js 파일을 하나로 합친다든지 js파일 내의 name들을 줄인다든지 등의 작업들을 통해서 해당 사이트의 performance를 향상시킬 수 있습니다.

### Profiling Components with the Chrome Performance Tab

브라우저에서 제공하는 performance tool(현재 Chrome, Edge, 그리고 IE에서 이 기능을 제공합니다)을 사용하면 component들이 어떻게 mount, update 그리고 unmount되는지 볼 수 있습니다.
이 기능을 사용하면 원하지 않는 UI가 실수에 의해서 update되는 현상을 알아낼 수 있고, UI가 어떻게 update되는지 확인할 수 있습니다.
Chrome에서의 사용법은 [Ben Schwarz의 article](https://building.calibreapp.com/debugging-react-performance-with-react-16-and-chrome-devtools-c90698a522ad)에서 볼 수 있습니다.

### Profiling Components with the DevTools Profiler

**react-dom** 16.5+ 과 **react-native** 0.57+ 에서는 **React DevToold Profiler**라는 profiling 기능을 제공합니다. 해당 기능은 [“Introducing the React Profiler”](https://reactjs.org/blog/2018/09/10/introducing-the-react-profiler.html)에서 확인 가능합니다.

### Virtualize Long Lists

UI에서 어떤 list를 보여주기 위해서 스크롤을 사용할 때, 해당 list의 크기가 정도 이상이라면 performance의 저하를 불러올 수 있습니다. 화면에 보이지 않는 list의 요소들이 dom에서 그려져있기 때문입니다. 이때, **windowing**이라는 기술을 사용할 수 있습니다. 이 기술을 통해서, list의 요소중에서 화면에 보여지는 요소만 그리도록 할 수 있습니다.

[react-window](https://react-window.now.sh/)와 [react-virtualized](https://bvaughn.github.io/react-virtualized/)가 유명한 windowing library입니다.

### Avoid Reconciliation

의도하지 않은 불필요한 re-rendering은 대부분 크게 문제되지 않지만, 만약 눈에 띌 정도로 성능저하가 일어난다면 lifecycle 함수중의 하나인 **shouldComponentUpdate**를 overriding함으로써 성능을 높일 수 있습니다.

### shouldComponentUpdate In Action

React에서 화면이 rendering되는데 영향을 주는 요소는 두가지가 있습니다. __**shouldComponentUpdate** 함수가 true를 return__ 하거나 __새로 그려질 elements가 이전의 elements와 같을때__(virtual dom을 통해서 비교합니다) 입니다. 두 요소를 각각 **SCU**와 **vDOMEq**(elements were equivalent의 줄임말)라고 하겠습니다.

* SCU===true && vDOMEq===true
  * SCU가 true를 return했지만, 바뀐것이 없기 때문에 re-rendering되지 않습니다.
* SCU===true && vDOMEq===false
  * SCU도 true를 return하고, 바뀐것이 있기 때문에 re-rendering하게 됩니다.
* SCU===false && vDOMEq===true
  * 바뀐것도 없고, shouldComponentUpdate 함수도 false를 return하기 때문에 자신과 child component들이 re-rendering 되지 않습니다.
* SCU===false && vDOMEq===false
  * 바뀌었지만, shouldComponentUpdate 함수가 false를 return하기 때문에 자신과 child component들이 re-rendering 되지 않습니다.

위와 같은 경우들을 고려해서 React의 performance를 향상시킬 수 있습니다. 여기서, shouldComponentUpdate 함수를 좀 더 살펴보겠습니다.

# Mutation in React

## Problems raised

```js
class CounterButton extends React.Component {
  constructor(props) {
    super(props);
    this.state = {count: 1};
  }

  shouldComponentUpdate(nextProps, nextState) {
    if (this.props.color !== nextProps.color) {
      return true;
    }
    if (this.state.count !== nextState.count) {
      return true;
    }
    return false;
  }

  render() {
    return (
      <button
        color={this.props.color}
        onClick={() => this.setState(state => ({count: state.count + 1}))}>
        Count: {this.state.count}
      </button>
    );
  }
}
```

**CounterButton** component는 **props.color**나 **state.count**가 변경될 때만 re-rendering되도록 shouldComponentUpdate 함수에서 설정되어 있습니다. 이 component가 좀 더 복잡해지더라도 props와 state의 모든 field에 대해서 __shallow comparsion__ 을 통해서 update할지 말지를 결정할 수 있습니다. 이런 패턴은 매우 흔하기 때문에 React에서 제공하는것이 있습니다. React.Component가 아닌 **React.PureComponent**를 상속받으면 됩니다. 아래의 코드는 위의 코드와 같은 로직을 가집니다.

```js
class CounterButton extends React.PureComponent {
  constructor(props) {
    super(props);
    this.state = {count: 1};
  }

  render() {
    return (
      <button
        color={this.props.color}
        onClick={() => this.setState(state => ({count: state.count + 1}))}>
        Count: {this.state.count}
      </button>
    );
  }
}
```

하지만, state나 props의 field가 primitive variable이 아닌 array나 object처럼 복잡한 data structure를 가지게 된다면 **shallow comparision**으로 비교할 수 없으므로 문제가 됩니다. 아래의 코드를 보겠습니다.

```js
class ListOfWords extends React.PureComponent {
  render() {
    return <div>{this.props.words.join(',')}</div>;
  }
}

class WordAdder extends React.Component {
  constructor(props) {
    super(props);
    this.state = {
      words: ['marklar']
    };
    this.handleClick = this.handleClick.bind(this);
  }

  handleClick() {
    // This section is bad style and causes a bug
    const words = this.state.words;
    words.push('marklar');
    this.setState({words: words});
  }

  render() {
    return (
      <div>
        <button onClick={this.handleClick} />
        <ListOfWords words={this.state.words} />
      </div>
    );
  }
}
```

**ListOfWords** component는 **React.PureComponent**를 상속받아서, props.words에 대한 **shallow comparision**을 통해 re-rendering을 하려 합니다. 하지만 단순한 비교(shallow comparision)로는 handleClick 함수에 의한 **words** 배열의 변화를 알수없고 계속 같다고 판단합니다. 그리고 **ListOfWords** component는 update되지 않습니다...

## The Power Of Not Mutating Data

위와같은 문제를 해결하기위한 가장 간단한 방법은 **mutating values**를 사용하지 않는 것입니다. 위에서 본 코드의 **handleClick** 함수를 아래 코드와 같이 바꿀 수 있습니다.

```js
  handleClick() {
    this.setState(state => ({
      words: [...state.words, 'marklar']
    }));
  }
```

위와 비슷한 방법으로, object의 mutation을 방지하기 위해 object를 mutate할 수 있습니다. 막상 들으면 이상하게 들리지만, 아래의 코드를 보겠습니다. :)

```js
function updateColorMap(colormap) {
  colormap.right = 'blue';
}
```

위의 코드는 colormap.right를 'blue'로 바꾸려 합니다. Original object를 mutating하지 않기위해 아래의 코드를 사용할 수 있습니다.

```js
function updateColorMap(colormap) {
  return {...colormap, right: 'blue'};
}
```

# Immutable.js: Using Immutable Data Structures

위의 문제를 해결하기 위한 방법으로 [Immutable.js](https://github.com/facebook/immutable-js)를 들수있습니다. Immutable.js는 구조 공유를 통해 작동하는 **불변하고(Immutable)** **지속적인(Persistent)** collection을 제공합니다.

* __Immutable__: 일단 생성되면, collection은 이후에 변경할 수 없습니다.
* __Persistent__: 새로운 collection은 이전 collection과 여러 mutation으로 인해서 만들어집니다. 그리고 새로운 collection이 만들어지더라도 이전 collection은 유효(valid)합니다.
* __Structural Sharing__: 새로운 collection은 원래의 collection과 최대한 동일한 구조로 생성되어서, 복사를 최소한으로 줄여서 performance를 향상시킵니다.

**Immutability**는 변화를 추적할때 유용할 수 있습니다. 변화가 있을때 마다 새로운 object를 만들기 때문에, object의 reference가 바뀌었는지만 확인하면 됩니다.

```js
const x = { foo: 'bar' };
const y = x;
y.foo = 'baz';
x === y; // true
```

위의 코드는 일반적인 JavaScript 코드입니다. **y.foo**를 바꾸더라도, x===y는 true입니다. object의 copy는 "__copy by reference__"이기 때문입니다. 위의 코드를 **Immutable.js**를 사용하면 아래와 같은 코드가 됩니다.

```js
const SomeRecord = Immutable.Record({ foo: null });
const x = new SomeRecord({ foo: 'bar' });
const y = x.set('foo', 'baz');
const z = x.set('foo', 'bar');
x === y; // false
x === z; // true
```

x에 변화가 있을 때 새로운 object와 reference가 생성됩니다. 그리고 __(x===y)__ reference equality check를 통해서 y에 새로 저장된 값과 x에 원래 있던 값이 같은지 확인할 수 있습니다.

**Immutable data structures**는 **shouldComponentUpdate**함수에서 구현해야하는 **object의 변화 추적**에 큰 도움이 되고, 이는 결국 **performance 향상**으로 귀결될 수 있습니다.

## Why do I need it?

지금까지 React의 performance와 관련해서 Immutability의 중요성을 살펴봤습니다. 그런데, 위의 Immutable.js를 사용한 코드만 보면 딱히 Immutable.js를 사용할 필요를 못느낄수도 있습니다. 그냥 [ES6의 Spread syntax](https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Operators/Spread_syntax)를 사용하면 될거같기도 합니다. 하지만, object의 구조가 저렇게 단순하지 않고, 매우 복잡한 경우엔 골치아파집니다.

```js
const config = {
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
      type: 'category',
      label: 'Test type'  // I want to change this string!!
    },
    y: {
      label: 'ms'
    }
  }
};
```

위의 **config**라는 object의 config.axis.x.label을 바꾸는데 순수 JavaScript만을 사용한다면 어떻게 될까요?

```js
// using assign
const assignConfig = Object.assign({}, config, {axis: Object.assign({}, config.axis, {x: Object.assign({}, config.axis.x, {label: 'New String'})})});

// using spread syntax
const spreadConfig = {
  ...config,
  axis: {
    ...config.axis,
    x: {
      ...config.axis.x,
      label: 'New String'
    }
  }
};
```

더러운 코드가 됩니다... 코드의 가독성도 나쁘고 유지보수에도 좋지 않습니다. 실수 할 수도 있습니다.
(__JSON__ 객체의 __parse__와 __stringify__를 사용해서 deep copy하는 방법도 있지만, 이 방법은 object 내에 function이 value로 있을 경우 function은 복사가 되지 않습니다.)
이런 경우에 Immutable.js를 사용하면 편하게 작업할 수 있습니다.

## Usages of Immutable.js

**Immutable.js**에서 자주 사용되는 API들의 사용법에 대해서 예제와 함께 알아보겠습니다.


# References

* [Using Immutable Data Structures](https://reactjs.org/docs/optimizing-performance.html#using-immutable-data-structures)
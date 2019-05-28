---
layout: post
title: 정말 간단한 react-redux
comments: true
categories: react
---
정말 간단한 react-redux
=====================

최근들어서 React에 관심이 많아졌다.
React를 공부하다보니 redux에 관한 이야기가 나와서 적어보려고 한다.

Redux에 대한 소개를 보면 페이지가 매우매우 깔끔하고 번역도 잘되어 있어 이해하기 쉽다.

Redux는 자바스크립트 앱을 위한 예측 가능한 상태 컨테이너라고 명시되어 있다.
React을 써본지 얼마되지 않았지만 State를 관리하는 것이 꾀나 귀찮은 일이다.
직접 Component에서 setState를 하고 어떤 Component들이 영향을 받는 지를 생각해야되고 디버깅 시에는 어디서 어떻게 State가 바뀌었는 지를 찾아내야 하는 일이 어려운 일이다.

Redux는 이런 것들을 해결하기 위해 세가지 원칙을 제시하고 있다.
1. 어플리케이션의 모든 State는 하나의 Store 안에 객체트리구조로 저장된다
2. 하나의 State는 읽기전용이다
3. 변화는 순수함수로 구성되어야 한다

Redux에서는 store의 개념이 등장하는 데 1번에서 이야기한 것과 같이 앱 내의 모든 State들의 집합이라고 볼 수 있다. 여러 Component가 가진 State들을 모아두는 개념이다.

이 Store를 변경하는 행동의 정의가 바로 Action이다. Redux에서 우리는 Action을 정의할 수 있고 보낼 수 있다.

실제로 이 Action을 처리하는 것이 바로 Reducer이다. reduce라는 뜻이 무엇을 줄인다라기 보다는 다룬다라는 것이 좀 더 정확한 해석이다.

일단 간단한 예제를 살펴보도록 하자!

![My helpful screenshot]({{ "/assets/toggle.gif" | absolute_url }})

그냥 클릭을 하면 글이 toggle 되면서 TRUE <-> FALSE를 반복하는 정말 쉬운 앱이다.

일단 가장 처음 index.html이다.

```html
<!DOCTYPE html>
<html lang="en">
  <body>
    <h1> this is html </h1>
    <div id="root">
    </div>
  </body>
</html>
```

일단 root라는 아이디를 가진 element를 선언하고 여기에 React App이 위치한다.

그리고 root를 정의하고 있는 index.js이다.

다음은 리듀서이다.

```javascript
import React from 'react';
import ReactDOM from 'react-dom';
import { createStore } from 'redux';
import { Provider } from 'react-redux';
import App from './components/App';
import toggleApp from './reducers';

const store = createStore(toggleApp);
const rootElement = document.getElementById('root');

ReactDOM.render(
	<Provider store={store}>
		<App />
    </Provider>, rootElement);
```
일반적으로 React를 쓰는 것과 크게 다르지 않지만 여기서 중요한 것은 toggleApp 객체를 통해서 redux에서 제공하는 `createStore()`를 통해서 Store를 만든다는 것이다. 이 Store는 Provider의 Prop으로 들어간다.

```javascript

import { combineReducers } from 'redux';
import { TOGGLE } from './actions';

function value(state = false, action) {
    switch (action.type) {
        case TOGGLE:
            return !state;
        default:
            return state;
    }
}

const toggleApp = combineReducers({
    value
});

export default toggleApp;
```

여기서는 현재 State는 매우 간단하게 bool 값 하나만을 가진다. 이것을 value라는 function으로 정의하고 어떠한 액션이 들어왔는 지에 따라서 액션을 처리하는 방법을 결정한다.

간단히 이야기하면 액션을 적용하는 것이다.

그리고 여러가지 리듀서를 하나로 묶어주는 `combineReducers()` 를 이용해서 toggleApp으로 만들고 index.js에서 보여줬던 것처럼 이를 통해서 Store를 만든다.

다음은 Action 이다.

``` javascript
export const TOGGLE = 'TOGGLE';

export function toggle_action() {
    return { type: TOGGLE};
}
```

너무 간단하다! 말그대로 행동을 정의하고 이것을 Reducer에서 사용한다.

다음부터는 Component를 소개할 것인데 Component는 크게 Presentational과 Container로 나뉜다.
Persentational Component는 원래 사용하던 React Component와 크게 다르지 않다. 이 예제에서는 단순하게 `render()`만 사용한다.

Container는 리듀서에서 State를 조회할 수 있고 이 조회된 데이터를 Persentational의 Prop으로 내려주는 역할을 한다.
Persentational은 이것을 받아서 `render()`를 하는 것이다.

결국 State를 바꾸는 최초의 트리거는 Persentational에서 `onClick()` 에서 시작되며 Persentational은 State를 직접 수정하지 않으므로 Container에서 내려주는 function을 호출시키는 형태가 되어야 한다.

말로는 어려운 것 같으니 직접 코드로 보자

일단 Container인 ToggleContainer이다.

```javascript
import { connect } from 'react-redux';
import { toggle_action } from '../actions';
import Toggle from '../components/Toggle';


function getText(val) {
    if (val == true)
        return 'TRUE';
    else
        return 'FALSE';
}

const mapStateToProps = state => {
    return {
        text: getText(state.value)
    }
};

const mapDispatchToProps = dispatch => {
    return {
        onToggleClick:
            () => {
                dispatch(toggle_action())

            }
    }
}

const ToggleContainer = connect(
    mapStateToProps,
    mapDispatchToProps
)(Toggle);

export default ToggleContainer;
```

`mapStateToProps`와 `mapDispatchToProps` 두 함수를 정의해서 connect에 넣어주고 나오는 결과 함수에 또 Toggle을 넣어주면 Container가 만들어진다!

`mapStateToProps`에서는 Reducer에서 넘어온 value라는 bool 값을 Persentational로 넘겨주는데 string으로 간단한 변환 작업을 거친다.
`mapDispatchToProps`에서는 Persentational에서 클릭이벤트시에 사용할 함수 `onToggleClick`를 Prop형태로 넘겨준다. Persentational의 onClick에 연결하면 작동한다.
신기한 것은 단순한 `toggle_action()`호출이 아니라 dispatch를 통해서 처리하는 것인데 이는 어떠한 Action을 리듀서로 전달해서 처리한다는 뜻이다.

redux에서는 엄격한 일방향 데이터의 흐름이 매우매우 중요하다.

이제 Presentational Component이다.

```javascript
import React, { Component } from 'react';
import PropTypes from 'prop-types';
import { connect } from 'react-redux';
import { TOGGLE, toggle } from '../actions';
import Toggle from './Toggle';
import ToggleContainer from '../containers/ToggleContainer';


class App extends Component {
	render() {
		return (
			<div>
				<ToggleContainer />
			</div>
		);
	}
}


export default App;
```
앞에서 만든 ToggleContainer를 `render()`하도록 설정한다.

그리고 Toggle.js이다.
``` javascript
import React, { Component } from 'react'
import PropTypes from 'prop-types';



const Toggle = ({ text, onToggleClick }) => (
	<div>
		<h1 onClick={() => onToggleClick()}>
			{
				text
			}
		</h1>
	</div>
);


Toggle.propTypes = {
	text: PropTypes.string.isRequired,
	onToggleClick: PropTypes.func.isRequired
};

export default Toggle;
```
Container에서 넘겨주는 Prop을 통해서 출력과 onClick을 구현한 모습이다.
여기서 신기한것은 Toggle Container를 만들때 render()을 정의하지 않아도 `connect()`만 하면 알아서 ToggleContainer아래에 Toggle이 붙는 다는 것이다.

이것으로 redux를 정말 쉬운 예제로 설정해보았다.
공부한지 얼마안되서 그런지 아직도 헷갈리는 것이 너무 많고 더 노력해야 겠다는 생각이 든다.

참조

[redux 공식 튜토리얼](https://lunit.gitbook.io/redux-in-korean/)
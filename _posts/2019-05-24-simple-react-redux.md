정말 간단한 react-redux 
(간단한 Toggle 예제 작성하기)
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

... 개념 설명 계속

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
일반적으로 React를 쓰는 것과 크게 다르지 않지만 createStore()를 사용하는 것이 눈에 띈다. 그리고 앞에서 index.html에서 정의한 root element도 여기서 React를 통해서 render하고 있다.


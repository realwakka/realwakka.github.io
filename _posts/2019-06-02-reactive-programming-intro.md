---
title: "Reactive Programming 소개"
date: 2019-06-02 08:26:28 -0400
categories: reactive-programming
---

리액티브 프로그래밍에 대해서 배운 것을 정리해보자
처음에 너무나도 좋은 참고 사이트를 찾아서 거기에서 배운 것을 정리하려고 한다.

일단 리액티브 프로그래밍은 비동기 데이터 스트림을 프로그래밍하는 것이다!

사실 이것은 전혀 새로울 게 없다. 이벤트 버스나 클릭 이벤트는 비동기 스트림이며 이것들을 observe할수도 있고 다른 side effects도 가진다.
리액티브 프로그래밍은 그에대한 강화이다. 클릭이나 hover이벤트 말고도 무엇이든지 스트림으로 만들 수 있다. 스트림은 간단하고 어디든지 존재한다.
일반 변수, 유저 input, 속성, 캐시, 자료구조 같이 말이다. 예를 들면 트위터 feed가 데이터 스트림 이고 같은 방식으로 클릭이벤트도 마찬가지이다.

예를 들면 위의 클릭이벤트 같은 방식으로 트위터 피드가 데이터 스트림이 된다고 상상해보자.
그리고 당신은 스트림을 listen하고 그것에 따라서 반응할 수 있다.

그리고 이 스트림들을 combine, create, filter 할 수 있는 놀라운 도구를 받을 것입니다. 여기서 바로 "함수형" 이라는 마법이 벌어지는 것이죠.
스트림은 다른 스트림의 입력이 될 수 있습니다. 여러 스트림이 하나의 스트림에 입력이 될수도 있습니다.
두개의 스트림을 하나로 합칠 수도 있고, 관심있는 이벤트들만 필터링 할수도 있습니다.
데이터의 값을 map을 통해서 하나의 또다른 스트림을 만들어 낼 수 있습니다.

하나의 스트림은 이벤트들이 시간순으로 정렬된 시퀀스이다. 세가지 형태의 이벤트를 발생한다. value, error, completed 세가지 신호를 발생시킨다. 예를 들어 completed가 발생하면 현재 보고 있는 view 버튼이 닫혔다는 뜻이다.

우리는 비동기적으로 발생된 이벤트들을 함수를 통해 capture할 수 있으며 value, error, completed가 발생 할때 실행되는 함수가 각각 다르다. 보통 error와 completed는 생략하고 value 함수에 신경을 많이 쓰는 편이다. 이 함수가 스트림을 listening한다는 것을 우리는 subscribing이라고 한다.
여기서 우리가 정의한 함수는 observer이며 스트림은 observable이다. 정확히 옵저버 패턴이다.

```
--a---b-c---d---X---|->

a, b, c, d are emitted values
X is an error
| is the 'completed' signal
---> is the timeline
```

옵저버패턴은 너무 익숙해서 다른 이야기를 해보겠다. 우리는 원래의 클릭 그대로를 통해서 새로운 클릭 이벤트 스트림을 만들어보겠다.

처음으로 버튼이 몇번 클릭되었는 지를 나타내는 카운터 스트림을 만들자. 보통의 Reactive 라이브러리에서는 map, filter, scan 등의 다양한 스트림에 적용되는 함수를 제공하고 있다. clickStream.map(f) 같이 함수를 사용하면 clickStream으로 만든 새로운 스트림을 반환한다. 이 새로운 스트림은 기존의 스트림을 변경하지 않는다. 이것을 바로 immutability라고 부르며 이것은 Reactive 내에서 팬케익의 시럽같은 좋은 역할을 한다. 그리고 이것을 연속된 clickStream.map(f).scan(g)같이 연속된 형태로 사용할 수 있다.

```
  clickStream: ---c----c--c----c------c-->
               vvvvv map(c becomes 1) vvvv
               ---1----1--1----1------1-->
               vvvvvvvvv scan(+) vvvvvvvvv
counterStream: ---1----2--3----4------5-->
```

`map(f)`에서는 각각의 이벤트 값이 f 함수로 전달된다. 이 경우에는 간단하게 1로 매핑했다. `scan(g)`에서는 스트림 이전의 값들을 모두 취합하여 x = g(accumulated, current)를 만들어 낸다.
여기서 g는 간단하게 더하기 함수를 사용하였다. 그런 다음 counterStream은 총 클릭이 몇번이 일어났는가를 나타내게 된다.

리액티브의 진정한 힘을 보여주기 위해서는 더블 클릭이벤트를 이야기 해보아야 한다. 트리플 클릭까지 더블클릭을 보는 보통, 다중 클릭을 생각하면 더 재밌는 이야기가 될것이다.
만약에 당신이 이것을 기존의 imperative와 stateful한 방법으로 구현한다면 어떨지 생각해보아라. 짜증나고 몇몇의 변수들을 유지하면서 시간 간격과 씨름하게 될것이다.

리액티브는 정말 간단하다. 로직은 단 4줄이다. 코드는 일단 나중에 보도록하자. 다이어그램으로 스트림 설계를 생각하는 것이 초보든 전문가든 최고의 방법이다.

![Multiple clicks stream](http://i.imgur.com/HMGWNO5.png)

회색 상자가 스트림을 변형시켜서 다른 스트림을 얻어내는 함수를 나타낸다. 처음으로 250ms 간격으로 발생한 이벤트를 리스트로 만들어 준다. 이것이 바로 `stream.throttle(250ms)`에서 하는일이다.
일단은 작은 것까지 이해하기는 어려울 지도 모른다. 앞의 함수의 결과는 리스트의 스트림이다. 우리는 이것에 `map()`을 적용시켜서 리스트의 길이를 나타내는 스트림을 만단다. 마지막으로 `filter(x >= 2)`를 통해서 1 값을 필터링한다.
3개의 연산을 통해서 우리는 목표한 스트림을 만들어냈다. 그리고 이 스트림을 subscribe 또는 listen할 수 있다.


I hope you enjoy the beauty of this approach. This example is just the tip of the iceberg:
you can apply the same operations on different kinds of streams, for instance, on a stream of API responses;
on the other hand, there are many other functions available.

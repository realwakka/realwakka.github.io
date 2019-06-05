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

The map(f) function replaces (into the new stream) each emitted value according to a function f you provide.
In our case, we mapped to the number 1 on each click.
The scan(g) function aggregates all previous values on the stream, producing value x = g(accumulated, current),
where g was simply the add function in this example. Then, counterStream emits the total number of clicks whenever a click happens.

`map(f)` 함수는 발생한 각각의 value를 함수 f를 통해서 교체한다.
이 경우에는 클릭당 숫자 1을 매핑 시켰다.
`scan(g)` 함수는 이전의 스트림에 있는 value를 모아서 x value = g(accumulated, current) 의 형태로 값을 만들어낸다.
예제에서는 간단히 더하기 함수를 적용시켰다. 이렇게 하면 countStream은 클릭된 숫자를 나타내게 된다.


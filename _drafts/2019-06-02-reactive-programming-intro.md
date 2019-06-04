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

imagine your Twitter feed would be a data stream in the same fashion that click events are.
You can listen to that stream and react accordingly.
예를 들면 위의 클릭이벤트 같은 방식으로 트위터 피드가 데이터 스트림이 된다고 상상해보자.
그리고 당신은 스트림을 listen하고 그것에 따라서 반응할 수 있다.

On top of that, you are given an amazing toolbox of functions to combine, create and filter any of those streams.
That's where the "functional" magic kicks in.
A stream can be used as an input to another one. Even multiple streams can be used as inputs to another stream.
You can merge two streams. You can filter a stream to get another one that has only those events you are interested in.
You can map data values from one stream to another new one.


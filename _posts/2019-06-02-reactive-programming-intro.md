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

그리고 이 스트림들을 combine, create, filter 할 수 있는 놀라운 도구를 받을 것입니다. 여기서 바로 "함수형" 이라는 마법이 벌어지는 것이죠.
스트림은 다른 스트림의 입력이 될 수 있습니다. 여러 스트림이 하나의 스트림에 입력이 될수도 있습니다.
두개의 스트림을 하나로 합칠 수도 있고, 관심있는 이벤트들만 필터링 할수도 있습니다.
데이터의 값을 map을 통해서 하나의 또다른 스트림을 만들어 낼 수 있습니다.

A stream is a sequence of ongoing events ordered in time. It can emit three different things: a value (of some type), an error, or a "completed" signal. Consider that the "completed" takes place, for instance, when the current window or view containing that button is closed.

하나의 스트림은 이벤트들이 시간순으로 정렬된 시퀀스이다. 세가지 형태의 이벤트를 발생한다. value, error, completed 세가지 신호를 발생시킨다. 예를 들어 completed가 발생하면 현재 보고 있는 view 버튼이 닫혔다는 뜻이다.

We capture these emitted events only asynchronously, by defining a function that will execute when a value is emitted, another function when an error is emitted, and another function when 'completed' is emitted. Sometimes these last two can be omitted and you can just focus on defining the function for values. The "listening" to the stream is called subscribing. The functions we are defining are observers. The stream is the subject (or "observable") being observed. This is precisely the Observer Design Pattern.

우리는 비동기적으로 발생된 이벤트들을 함수를 통해 capture할 수 있으며 value, error, completed가 발생 할때 실행되는 함수가 각각 다르다. 보통 error와 completed는 생략하고 value 함수에 신경을 많이 쓰는 편이다. 이 함수가 스트림을 listening한다는 것을 우리는 subscribing이라고 한다. 여기서 우리가 정의한 함수는 observer이며 스트림은 observable이다. 정확히 옵저버 패턴이다.

Since this feels so familiar already, and I don't want you to get bored, let's do something new: we are going to create new click event streams transformed out of the original click event stream.
옵저버패턴은 너무 익숙해서 다른 이야기를 해보겠다. 우리는 원래의 클릭 그대로를 통해서 새로운 클릭 이벤트 스트림을 만들어보겠다.

First, let's make a counter stream that indicates how many times a button was clicked. In common Reactive libraries, each stream has many functions attached to it, such as map, filter, scan, etc. When you call one of these functions, such as clickStream.map(f), it returns a new stream based on the click stream. It does not modify the original click stream in any way. This is a property called immutability, and it goes together with Reactive streams just like pancakes are good with syrup. That allows us to chain functions like clickStream.map(f).scan(g):

처음으로 버튼이 몇번 클릭되었는 지를 나타내는 카운터 스트림을 만들자. 보통의 Reactive 라이브러리에서는 map, filter, scan 등의 다양한 스트림에 적용되는 함수를 제공하고 있다. clickStream.map(f) 같이 함수를 사용하면 clickStream으로 만든 새로운 스트림을 반환한다. 이 새로운 스트림은 기존의 스트림을 변경하지 않는다. 이것을 바로 immutability라고 부르며 이것은 Reactive 내에서 팬케익의 시럽같은 좋은 역할을 한다. 그리고 이것을 연속된 clickStream.map(f).scan(g)같이 연속된 형태로 사용할 수 있다.
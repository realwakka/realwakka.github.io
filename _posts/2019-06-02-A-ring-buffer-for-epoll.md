---
title: "epoll을 위한 ring buffer"
comments: "true"
categories: "linux"
---

lwn 기사에 대한 번역 글입니다.
<!--
The set of system calls known collectively as epoll was designed to make polling for I/O events more scalable.
-->
우리가 epoll로 알고있는 시스템콜은 I/O 이벤트들을 확장적으로 polling 하기위해 설계되었다고 알려져있습니다.
<!--
To that end, it minimizes the amount of setup that must be done for each system call and returns multiple events so that the number of calls can also be minimized.
-->
이것을 위해 의무적으로 시스템콜을 최소화하고 여러가지 이벤트들을 반환해서 콜 횟수를 줄이고 있습니다.
<!-- But that turns out to still not be scalable enough for some users. -->
그러나 이것은 몇몇 사용자들에게는 충분하지 않습니다.
<!-- The response to this problem, in the form of this patch series from Roman Penyaev, takes a familiar form: add yet another ring-buffer interface to the kernel. -->
이 문제에 대한 답변으로 Roman Penyaev의 패치는 다음과 같이 제안하고 있습니다. : 커널에 또다른 링버퍼 인터페이스를 추가합니다.
<!-- The poll() and select() system calls can be used to wait until at least one of a set of file descriptors is ready for I/O.
Each call, though, requires the kernel to set up an internal data structure so that it can be notified when any given descriptor changes state.
Epoll gets around this by separating the setup and waiting phases, and keeping the internal data structure around for as long as it is needed. -->
poll()과 select()는 적어도 하나의 fd가 ready 상태 일때까지 기다리는 데 사용되었습니다.
그러나 이 시스템콜은 fd가 ready 상태를 알려줄 수 있도록 커널 내부 자료구조를 필요로 하였습니다.
epoll은 이것을 설정과 기다리는 상태를 나누면서 해결하였고 내부구조를 필요한 할때 까지만 유지하였다.

<!--
An application starts by calling epoll_create1() to create a file descriptor to use with the subsequent steps. That call, incidentally, supersedes epoll_create(); it replaces an unused argument with a flags parameter.
Then epoll_ctl() is used to add individual file descriptors to the set monitored by epoll. Finally, a call to epoll_wait() will block until at least one of the file descriptors of interest has something to report.
This interface is a bit more work to use than poll(), but it makes a big difference for applications that are monitoring huge numbers of file descriptors.
-->

어플리케이션은 뒤의 과정에 쓰이는 fd를 생성하기 위해 epoll_create1()을 호출합니다. 덧붙여 말하자면 이 시스템콜은 epoll_create()를 대체하며 사용되지 않는 인자를 플래그로 대체하는 구조입니다.
그리고 epoll_ctl()은 모니터링할 각 fd를 추가하기 위해 쓰입니다. 마지막으로 epoll_wait()은 적어도 하나의 fd가 깨어날 때까지 block 상태가 됩니다.
이 인터페이스는 poll()보다 조금 더 쓰기 어려운 면이 있습니다. 그러나 이 차이가 많은 fd를 사용하는 어플리케이션에는 큰 차이를 만들어냅니다.

<!--
That said, it would seem that there is still room for doing things better.
Even though epoll is more efficient than its predecessors,
an application still has to make a system call to get the next set of file descriptors that are ready for I/O.
On a busy system, where there is almost always something that is needing attention,
it would be more efficient if there were a way to get new events without calling into the kernel.
That is where Penyaev's patch set comes in;
it creates a ring buffer shared between the application and the kernel that can be used to transmit events as they happen.
-->

epoll()을 좀 더 개선 시킬 여지가 보입니다.
epoll이 이전 것들 보다 좀더 효율적일지라도, 어플리케이션은 여전히 시스템콜을 사용해서 다음 fd set을 얻어야합니다.
계속해서 돌아가는 바쁜 시스템에서는, 만약에 새로운 이벤트를 시스템콜 없이 얻을 수 있다면 더 효율적일 것입니다.
패치 내용은 아래와 같습니다.
어플리케이션과 커널간에 공유하는 ring buffer를 만들어서 발생한 이벤트를 전송합니다.

epoll_create() — the third time is the charm
The first step for an application that wishes to use this mechanism is to tell the kernel that polling will be used and how big ring buffer should be.
이 메커니즘을 사용할 때 어플리케이션이 가장 처음에 해야할 일은 커널에게 polling을 사용할 것과 ring buffer의 크기를 설정하는 것입니다.
Of course, epoll_create1() does not have a parameter that can be used for the size information, so it is necessary to add epoll_create2():
epoll_create1()은 크기에 관한 인자를 가지고 있지 않아 epoll_create2()를 추가합니다.
```c
    int epoll_create2(int flags, size_t size);
```
<!--
There is a new flag, EPOLL_USERPOLL, that tells the kernel to use a ring buffer to communicate events;
the size parameter says how many entries the ring buffer should hold.
This size will be rounded up to the next power of two;
the result sets an upper bound on the number of file descriptors that this epoll instance will be able to monitor.
A maximum of 65,536 entries is enforced by the current patch set.
-->

새로운 EPOLL_USERPOLL flag입니다. 이것은 이벤트를 교환하기 위한 ring buffer를 추가하는 옵션입니다.
size는 ring buffer가 가지는 엔트리의 크기입니다.
size는 2의 제곱의 형태로 올림이 되며 이것에 의해 모니터링 할 수 있는 fd의 크기가 결정됩니다.
이 패치에서는 65536개가 최대값으로 강제 설정됩니다.

<!--
File descriptors are then added to the polling set in the usual way with epoll_ctl().
There are some restrictions that apply here, though, since some modes of operation are not compatible with user-space polling.
In particular, every file descriptor must request edge-triggered behavior with the EPOLLET flag.
Only one event will be added to the ring buffer when a file descriptor signals readiness;
continually adding events for level-triggered behavior clearly would not work well.
The EPOLLWAKEUP flag (which can be used to prevent system suspend while specific events are being processed) does not work in this mode;
EPOLLEXCLUSIVE is also not supported.
-->

fd는 polling set에 추가할 때 보통하던데로 `epoll_ctl()`을 사용합니다.
약간의 제약사항이 생기는데 몇몇 연산이 이는 user-space polling과 호환이 되지 않아서 그렇습니다.
이때 특이하게 모든 fd는 edge-triggered behavior를 사용해야하며 EPOLLET flag을 사용합니다.
fd가 읽기가 가능할 때 오직하나의 이벤트만 ring buffer에 추가됩니다.
이벤트를 level-triggered behavior로 추가하면 제대로 동작하지 않습니다.
이벤트가 처리 될때 시스템 suspend를 막는 EPOLLWAKEUP 또한 제대로 동작되지 않으며 EPOLLEXCLUSIVE 또한 마찬가지이다.

<!--
Two or three separate mmap() calls are required to map the ring buffer into user space.
The first one should have an offset of zero and a length of one page; it will yield a page containing this structure:
-->
ring buffer을 userspace에서 매핑하기 위해서는 두 세가지의 `mmap()`이 필요합니다.
처음에는 한페이지의 길이와 오프셋이 0으로 되고 이것은 아래의 구조체를 포함하는 페이지를 생성합니다.
'''c

    struct epoll_uheader {
	u32 magic;          /* epoll user header magic */
	u32 header_length;  /* length of the header + items */
	u32 index_length;   /* length of the index ring, always pow2 */
	u32 max_items_nr;   /* max number of items */
	u32 head;           /* updated by userland */
	u32 tail;           /* updated by kernel */

	struct epoll_uitem items[];
    };

'''
<!--
The header_length field, somewhat confusingly, contains the length of both the epoll_uheader structure and the items array.
As seen in this example program, the intended use pattern appears to be that the application will map the header structure, get the real length, unmap the just-mapped page, then remap it using header_length to get the full items array.
-->
약간 헷갈리지만 헤더 길이 필드는 `epoll_uheader`와 배열 길이까지 포함한 값입니다.
예제프로그램에서 봤듯이 이 패턴을 사용하는 것은 어플리케이션이 이 구조체를 매핑할 때 실제크기가 필요하기 때문입니다.

<!--
One might expect that items is the ring buffer, but there is a layer of indirection used here.
Getting at the actual ring buffer requires calling mmap() another time with header_length as the offset and the index_length header field as the length.
The result will be an array of integer indexes into the items array that functions as the real ring buffer.
-->

`item`이 ring buffer 일거라고 생각하겠지만 여기서는 간접 레이어가 있습니다.
실제 ring buffer를 얻는 것은 `mmap()`을 한번 더 호출하는 것을 필요로 합니다.
이것은 인덱스 배열을 반환하여 진짜 ring buffer처럼 동작합니다.

<!--
The actual items used to indicate events are represented by this structure:
-->
이벤트를 가리키는 데 사용되는 실제 아이템은 아래의 구조체로 표현 됩니다.

```c

    struct epoll_uitem {
	__poll_t ready_events;
	__poll_t events;
	__u64 data;
    };

```
<!--
Here, events appears to be the set of events that was requested when epoll_ctl() was called,
-->
events는 epoll_ctl()을 호출했을 때 요청된 이벤트들의 set이다.
<!--
and ready_events is the set of events that has actually happened.
-->
ready_events는 실제로 이벤트가 일어난 이벤트 들이다.
<!--
The data field comes through directly from the epoll_ctl() call that added this file descriptor.
-->
data는 epoll_ctl()을 통해 직접 전달 된다.

<!--
Whenever the head and tail fields differ, there is at least one event to be consumed from the ring buffer.
To consume an event, the application should read the entry from the index array at head;
this read should be performed in a loop until a non-zero value is found there.
The loop, evidently, is required to wait, if necessary, until the kernel's write to that entry is visible.
The value read is an index into the items array — almost. It is actually the index plus one.
The data should be copied from the entry and ready_events set to zero; then the head index should be incremented.
-->
head과 tail이 다를 때 적어도 하나의 이벤트가 ringbuffer로 부터 처리 되었다는 것이다.
이벤트를 처리하기 위해서는 어플리케이션이 인덱스 배열을 읽어야한다. 이것은 0이 아닌 값이 나올때 까지 loop을 도는 것이다.
loop이 커널이 엔트리가 visible을 write 할때 까지 기다려야한다. 읽은 값은 아이템 배열의 인덱스 값이다. 사실은 +1한 값
엔트리로부터 데이터가 복사되어야하고 ready_events가 0이 된다. 그리고 head 인덱스가 증가한다.

<!--
So, in a cleaned up form, code that reads from the ring buffer will look something like this:
-->
그래서 ring buffer를 읽는 코드는 다음과 같이 됩니다.
```c
    while (header->tail == header->head)
        ;  /* Wait for an event to appear */
    while (index[header->head] == 0)
        ;  /* Wait for event to really appear */
    item = header->items + index[header->tail] - 1;
    data = item->data;
    item->ready_events = 0;  /* Mark event consumed */
    header->head++;
```
<!--
In practice, this code is likely to be using C atomic operations rather than direct reads and writes, and head must be incremented in a circular fashion. But hopefully the idea is clear.
-->
이 코드는 직접 읽기/쓰기 동작 보다는 C의 atomic operations에 가깝습니다. 순환 방식에서 head가 증가되어야 합니다. 그러나 이 아이디어는 명확합니다.
<!--
Busy-waiting on an empty ring buffer is obviously not ideal. Should the application find itself with nothing to do, it can still call epoll_wait() to block until something happens.
This call will only succeed, though, if the events array is passed as NULL, and maxevents is set to zero;
in other words, epoll_wait() will block, but it will not, itself, return any events to the caller.
It will, though, helpfully return ESTALE to indicate that there are events available in the ring buffer.
-->
빈 ring-buffer를 Busy-waiting하는 것은 분명히 이상적이지 않습니다.
어플리케이션이 할일이 없어지면 epoll_wait()을 불러서 무언가 발생할 때까지 기다릴 수 있습니다.
이 시스템콜이 성공이라도 이벤트 배열이 NULL로 전달되면 maxevents는 0으로 설정됩니다.
다시말해서 epoll_wait()는 block되며 자체적으로 어떤 이벤트를 반환하지 않습니다.
비록 ESTALE을 반환해서 가능한 이벤트가 ring buffer에 있다는 것을 나타내주긴 합니다.

<!---
This patch set is in its third revision, and there appears to be little opposition to its inclusion at this point.
The work has not yet found its way into linux-next, but it still seems plausible that it could be deemed ready for the 5.3 merge window.
--->
이 패치는 세번째 수정본입니다. 이것이 지금 포함되는 것에 약간의 반대 의견이 보입니다.
아직 linux-next에 넣어지지 않았지만, 5.3 merge window에 준비가 된것 같아보입니다.

<!--
Some closing grumbles
Figuring out the above interface required a substantial amount of reverse engineering of the code.
This is a rather complex new API, but it is almost entirely undocumented;
that will make it hard to use, but the lack of documentation also makes it hard to review the API in the first place.
It is doubtful that anybody beyond the author has written any code to use this API at this point.
Whether the development community will fully understand this API before committing to it is far from clear.
-->

약간의 투정을 부리자면, 위의 인터페이스들을 이해하려면 상당히 많은 리버스 엔지니어링이 필요합니다.
이것은 복잡한 API 때문이라기 보다는 모두 문서화 되지 않아서 그렇습니다.
이것 때문에 사용하기 어렵긴 하지만 처음에는 문서화의 부재 또한 API를 보기 어렵게 만드는 요인입니다.
작성자 이외의 사람들이 API를 잘 사용할 수 있을 지는 매우 의심스럽습니다.
개발 커뮤니티가 API를 커밋하기 전에 완전히 이해할지도 모르겠습니다.

<!--
Perhaps the saddest thing, though, is that this will be yet another of many ring-buffer interfaces in the kernel.
Others include perf events, ftrace, io_uring, AF_XDP and, doubtless, others that don't come immediately to mind.
Each of these interfaces has been created from scratch and must be understood (and consumers implemented) separately by user-space developers.
Wouldn't it have been nice if the kernel had defined a set of standards for ring buffers shared with user space rather than creating something new every time?
One cannot blame the current patch set for this failing; that ship sailed some time ago.
But it does illustrate a shortcoming in how Linux kernel APIs are designed;
they seem doomed to never fit into a coherent and consistent whole.
-->
하지만 또 다른 많은 링버퍼 인터페이스들이 있는 것은 슬픈 일입니다.
그 다른 링버퍼들이란 perf events, ftrace, io_uring, AF_XDP 등등을 말합니다. 이것들은 별로 마음에 들지 않습니다.
이 인터페이스들은 스크래치에서 만들어졌고 각각 user-space개발자들이 이해해야 합니다.
계속해서 새로운 것을 만들기 보다는 만약 커널이 user-space와 공유 링버퍼에 대한 표준을 마련하면 얼마나 좋을까요?
이미 늦은 것 같습니다만 이 패치를 비난할 생각은 없습니다.
하지만 리눅스 커널 API가 설계된 방식의 단점을 보여주고 있습니다.
전체적인 구조에 절대 어울리지 않는 것으로 결정될 것으로 보입니다.


원문 : [LWN: A ring buffer for epoll](https://lwn.net/Articles/789603/)
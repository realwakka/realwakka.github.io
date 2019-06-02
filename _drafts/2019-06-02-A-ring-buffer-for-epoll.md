---
title: "epoll을 위한 ring buffer"
comments: "true"
categories: "linux"
---

The set of system calls known collectively as epoll was designed to make polling for I/O events more scalable.
우리가 epoll로 알고있는 시스템콜은 I/O 이벤트들을 확장적으로 polling 하기위해 설계되었다.
To that end, it minimizes the amount of setup that must be done for each system call and returns multiple events so that the number of calls can also be minimized.
마지막에, 이것은 꼭 해야하는 시스템콜을 최소화하고 여러가지 이벤트들을 반환해서 콜 횟수를 줄일 수 있다.
But that turns out to still not be scalable enough for some users.
그러나 이것은 몇몇 사용자들에게는 확장성있지 않다.
The response to this problem, in the form of this patch series from Roman Penyaev, takes a familiar form: add yet another ring-buffer interface to the kernel.
이 문제에 대한 답변으로 Roman Penyaev의 패치는 익숙한 폼을 가지고 있다 : 커널에 링버퍼 인터페이스를 추가한다.
The poll() and select() system calls can be used to wait until at least one of a set of file descriptors is ready for I/O.
Each call, though, requires the kernel to set up an internal data structure so that it can be notified when any given descriptor changes state.
Epoll gets around this by separating the setup and waiting phases, and keeping the internal data structure around for as long as it is needed.
poll()과 select()는 적어도 하나의 fd가 ready 상태 일때까지 기다리는 데 사용되었다. 비록 이 시스템콜은 fd가 ready 상태를 알려줄 수 있도록 커널 내부 자료구조를 필요로 하였다.
epoll은 이것을 설정과 기다리는 상태를 나누면서 해결하였고 내부구조를 필요한 할때 까지만 유지하였다.

An application starts by calling epoll_create1() to create a file descriptor to use with the subsequent steps. That call, incidentally, supersedes epoll_create(); it replaces an unused argument with a flags parameter.
Then epoll_ctl() is used to add individual file descriptors to the set monitored by epoll. Finally, a call to epoll_wait() will block until at least one of the file descriptors of interest has something to report.
This interface is a bit more work to use than poll(), but it makes a big difference for applications that are monitoring huge numbers of file descriptors.
어플리케이션은 뒤의 과정에 쓰이는 fd를 생성하기 위해 epoll_create1()을 호출한다. 이 시스템콜은 epoll_create()를 대체한다.
쓰이지않는 사용되지 않는 flag 인자를 대체한다.
그리고 epoll_ctl()은 모니터링할 각 fd를 추가하기위해 쓰인다. 마지막으로 epoll_wait()은 적어도 하나의 fd가 깨어날 때까지 block 상태가 된다.
이 인터페이스는 poll()보다 조금더 노력이 필요하다. 그러나 이 차이가 많은 fd를 사용하는 어플리케이션에는 큰 차이를 만들어낸다.

That said, it would seem that there is still room for doing things better.
Even though epoll is more efficient than its predecessors,
an application still has to make a system call to get the next set of file descriptors that are ready for I/O.
On a busy system, where there is almost always something that is needing attention,
it would be more efficient if there were a way to get new events without calling into the kernel.
That is where Penyaev's patch set comes in;
it creates a ring buffer shared between the application and the kernel that can be used to transmit events as they happen.
좀 더 개선 시킬 여지가 보인다.
epoll이 이전 것들 보다 좀더 효율적일지라도, 어플리케이션은 여전히 시스템콜을 사용해서 다음 fd set을 얻어야한다.
계속해서 돌아가는 바쁜 시스템에서는, 만약에 새로운 이벤트를 시스템콜 없이 얻을 수 있다면 더 효율적일 것이다.
패치 내용은 이렇다.
어플리케이션과 커널간에 공유하는 ring buffer를 만들어서 발생한 이벤트를 전송한다.

epoll_create() — the third time is the charm
The first step for an application that wishes to use this mechanism is to tell the kernel that polling will be used and how big ring buffer should be.
이 메커니즘을 사용할 때 어플리케이션이 가장 처음에 해야할 일은 커널에게 polling을 사용할 것과 ring buffer의 크기를 설정하는 것이다.
Of course, epoll_create1() does not have a parameter that can be used for the size information, so it is necessary to add epoll_create2():
epoll_create1()은 크기에 관한 인자를 가지고 있지 않아 epoll_create2()를 추가한다.

    int epoll_create2(int flags, size_t size);
There is a new flag, EPOLL_USERPOLL, that tells the kernel to use a ring buffer to communicate events;
the size parameter says how many entries the ring buffer should hold.
This size will be rounded up to the next power of two;
the result sets an upper bound on the number of file descriptors that this epoll instance will be able to monitor.
A maximum of 65,536 entries is enforced by the current patch set.
새로운 EPOLL_USERPOLL flag이다. 이것은 이벤트를 교환하기 위한 ring buffer를 추가하는 옵션이다.
size는 ring buffer가 가지는 엔트리의 크기이다.
size는 2의 제곱의 형태로 올림이 되며 이것에 의해 모니터링 할 수 있는 fd의 크기가 결정 된다.
이 패치에서는 65536개가 최대값으로 강제 설정된다.

File descriptors are then added to the polling set in the usual way with epoll_ctl().
There are some restrictions that apply here, though, since some modes of operation are not compatible with user-space polling.
In particular, every file descriptor must request edge-triggered behavior with the EPOLLET flag.
Only one event will be added to the ring buffer when a file descriptor signals readiness;
continually adding events for level-triggered behavior clearly would not work well.
The EPOLLWAKEUP flag (which can be used to prevent system suspend while specific events are being processed) does not work in this mode;
EPOLLEXCLUSIVE is also not supported.
fd는 polling set에 추가할 때 보통하던데로 epoll_ctl()을 사용한다. 약간의 제약사항이 생기는데 몇몇 연산이 이는 user-space polling과 호환이 되지 않아서 이다.
이때 특이하게 모든 fd는 edge-triggered behavior를 사용해야하며 EPOLLET flag을 사용한다.
fd가 읽기가 가능할 때 오직하나의 이벤트만 ring buffer에 추가된다.
이벤트를 level-triggered behavior로 추가하면 제대로 동작하지 않는다.
이벤트가 처리 될때 시스템 suspend를 막는 EPOLLWAKEUP 또한 제대로 동작되지 않으며 EPOLLEXCLUSIVE 또한 마찬가지이다.

Two or three separate mmap() calls are required to map the ring buffer into user space.
The first one should have an offset of zero and a length of one page; it will yield a page containing this structure:
ring buffer을 userspace에서 매핑하기 위해서는 두세가지의 mmap()이 필요하다.
처음에는 한페이지의 길이와 오프셋이 0으로 되고 이것은 아래의 구조체를 포함하는 페이지를 생성한다.

    struct epoll_uheader {
	u32 magic;          /* epoll user header magic */
	u32 header_length;  /* length of the header + items */
	u32 index_length;   /* length of the index ring, always pow2 */
	u32 max_items_nr;   /* max number of items */
	u32 head;           /* updated by userland */
	u32 tail;           /* updated by kernel */

	struct epoll_uitem items[];
    };

The header_length field, somewhat confusingly, contains the length of both the epoll_uheader structure and the items array.
As seen in this example program, the intended use pattern appears to be that the application will map the header structure, get the real length, unmap the just-mapped page, then remap it using header_length to get the full items array.

약간 헷갈리지만 헤더 길이 필드는 epoll_uheader와 배열 길이까지 포함한 값이다.
예제프로그램에서 봤듯이 이 패턴을 사용하는 것은 어플리케이션이 이 구조체를 매핑할 때 실제크기가 필요하기 때문이다.

One might expect that items is the ring buffer, but there is a layer of indirection used here.
Getting at the actual ring buffer requires calling mmap() another time with header_length as the offset and the index_length header field as the length.
The result will be an array of integer indexes into the items array that functions as the real ring buffer.

아이템이 ring buffer 일거라고 생각하겠지만 여기서는 간접 레이어가 있따.
실제 ring buffer를 얻는 것은 mmap()을 한번 더 호출하는 것을 필요로 한다.
이것은 인덱스 배열을 반환하여 진짜 ring buffer처럼 동작한다.

The actual items used to indicate events are represented by this structure:
이벤트를 가리키는 데 사용되는 실제 아이템은 아래의 구조체로 표현된다.

    struct epoll_uitem {
	__poll_t ready_events;
	__poll_t events;
	__u64 data;
    };

Here, events appears to be the set of events that was requested when epoll_ctl() was called,
events는 epoll_ctl()을 호출했을 때 요청된 이벤트들의 set이다.
and ready_events is the set of events that has actually happened.
ready_events는 실제로 이벤트가 일어난 이벤트 들이다.
The data field comes through directly from the epoll_ctl() call that added this file descriptor.
data는 epoll_ctl()을 통해 직접 전달 된다.

Whenever the head and tail fields differ, there is at least one event to be consumed from the ring buffer.
To consume an event, the application should read the entry from the index array at head;
this read should be performed in a loop until a non-zero value is found there.
The loop, evidently, is required to wait, if necessary, until the kernel's write to that entry is visible.
The value read is an index into the items array — almost. It is actually the index plus one.
The data should be copied from the entry and ready_events set to zero; then the head index should be incremented.
head과 tail이 다를 때 적어도 하나의 이벤트가 ringbuffer로 부터 처리 되었다는 것이다.
이벤트를 처리하기 위해서는 어플리케이션이 인덱스 배열을 읽어야한다. 이것은 0이 아닌 값이 나올때 까지 loop을 도는 것이다.
loop이 커널이 엔트리가 visible을 write 할때 까지 기다려야한다. 읽은 값은 아이템 배열의 인덱스 값이다. 사실은 +1한 값
엔트리로부터 데이터가 복사되어야하고 ready_events가 0이 된다. 그리고 head 인덱스가 증가한다.


So, in a cleaned up form, code that reads from the ring buffer will look something like this:

    while (header->tail == header->head)
        ;  /* Wait for an event to appear */
    while (index[header->head] == 0)
        ;  /* Wait for event to really appear */
    item = header->items + index[header->tail] - 1;
    data = item->data;
    item->ready_events = 0;  /* Mark event consumed */
    header->head++;
In practice, this code is likely to be using C atomic operations rather than direct reads and writes, and head must be incremented in a circular fashion. But hopefully the idea is clear.

Busy-waiting on an empty ring buffer is obviously not ideal. Should the application find itself with nothing to do, it can still call epoll_wait() to block until something happens. This call will only succeed, though, if the events array is passed as NULL, and maxevents is set to zero; in other words, epoll_wait() will block, but it will not, itself, return any events to the caller. It will, though, helpfully return ESTALE to indicate that there are events available in the ring buffer.

This patch set is in its third revision, and there appears to be little opposition to its inclusion at this point. The work has not yet found its way into linux-next, but it still seems plausible that it could be deemed ready for the 5.3 merge window.

Some closing grumbles
Figuring out the above interface required a substantial amount of reverse engineering of the code. This is a rather complex new API, but it is almost entirely undocumented; that will make it hard to use, but the lack of documentation also makes it hard to review the API in the first place. It is doubtful that anybody beyond the author has written any code to use this API at this point. Whether the development community will fully understand this API before committing to it is far from clear.

Perhaps the saddest thing, though, is that this will be yet another of many ring-buffer interfaces in the kernel. Others include perf events, ftrace, io_uring, AF_XDP and, doubtless, others that don't come immediately to mind. Each of these interfaces has been created from scratch and must be understood (and consumers implemented) separately by user-space developers. Wouldn't it have been nice if the kernel had defined a set of standards for ring buffers shared with user space rather than creating something new every time? One cannot blame the current patch set for this failing; that ship sailed some time ago. But it does illustrate a shortcoming in how Linux kernel APIs are designed; they seem doomed to never fit into a coherent and consistent whole.


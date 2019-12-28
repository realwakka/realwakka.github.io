---
title: "새로운 비동기 I/O API 안의 Ring"
comments: "true"
categories: "linux"
tags:
  - "linux"
  - "kernel"
  - "io_uring"
---

lwn 기사의 번역 글입니다.

<!-- While the kernel has had support for asynchronous I/O (AIO) since the 2.5 development cycle, it has also had people complaining about AIO for about that long. The current interface is seen as difficult to use and inefficient; additionally, some types of I/O are better supported than others. That situation may be about to change with the introduction of a proposed new interface from Jens Axboe called "io_uring". As might be expected from the name, io_uring introduces just what the kernel needed more than anything else: yet another ring buffer. -->

2.5 개발 사이클 이후로 리눅스 커널은 비동기 IO를 제공했지만 계속되는 사용자들의 불만이 있습니다. 현재 인터페이스는 사용하기 어렵고 비효율적입니다. 몇몇 IO 타입이 다른 것들보다 더 좋게 지원되었습니다. 이 때 Jens Axboe가 새로운 인터페이스 "io_uring"을 소개하였습니다. 이름에서 알 수 있듯이, io_uring은 커널에서 필요한 또 다른 ring buffer를 소개합니다.

Setting up

<!-- Any AIO implementation must provide for the submission of operations and the collection of completion data at some future point in time. In io_uring, that is handled through two ring buffers used to implement a submission queue and a completion queue. The first step for an application is to set up this structure a using new system call:
모든 AIO 구현은 연산을 제출(submission)하는 것과 미래 시점에 완료(completion)된 데이터의 묶음을 제공합니다. io_uring은 submission queue와 completion queue, 총 두개의 ring buffer로 사용합니다. 어플리케이션에서의 첫번째 과정은 새로운 시스템 콜을 통해서 이 구조체를 초기화하는 것입니다. -->
```c
    int io_uring_setup(int entries, struct io_uring_params *params);
```

<!-- The entries parameter is used to size both the submission and completion queues. The params structure looks like this: -->
entires 인자는 submission과 completion 모두에 사용됩니다. 구조체는 아래와 같이 생겼습니다.
```c
    struct io_uring_params {
	__u32 sq_entries;
	__u32 cq_entries;
	__u32 flags;
	__u16 resv[10];
	struct io_sqring_offsets sq_off;
	struct io_cqring_offsets cq_off;
    };
```
<!-- On entry, this structure (with the possible exception of flags as described later) should simply be initialized to zero. On return from a successful call, the sq_entries and cq_entries fields will be set to the actual sizes of the submission and completion queues; the code is set up to allocate entries submission entries, and twice that many completion entries. -->

초기에 이 구조체는 0으로 초기화되어야 합니다. 호출에 성공하면, sq_entries와 cq_entries 필드가 실제 submission과 completion queue의 사이즈로 세팅됩니다. 이 코드는 submission 엔트리를 할당 하고 그 두배를 completion 엔트리로 할당합니다.

<!-- The return value from io_uring_setup() is a file descriptor that can then be passed to mmap() to map the buffer into the process's address space. More specifically, three calls are needed to map the two ring buffers and an array of submission-queue entries; the information needed to do this mapping will be found in the sq_off and cq_off fields of the io_uring_params structure. In particular, the submission queue, which is a ring of integer array indices, is mapped with a call like: -->

io_uring_setup()의 리턴 값은 file descriptor(fd)이며 버퍼를 프로세스의 address space로 매핑하기 위해 mmap()에 넣을 수 있습니다. 좀 더 자세하게 이야기하자면, 두개의 ring buffer와 submission queue 엔트리 배열을 매핑하기 위해서는 세번의 함수 호출이 필요합니다. 이 매핑을 하기 위한 값은 io_uring_params 구조체의 sq_off와 cq_off 필드에서 구할 수 있습니다. 특히 정수 배열의 형태를 가진 queue submission은 다음과 같은 호출을 통해서 매핑됩니다.
```c
    subqueue = mmap(0, params.sq_off.array + params.sq_entries*sizeof(__u32),
    		    PROT_READ|PROT_WRITE|MAP_SHARED|MAP_POPULATE,
		    ring_fd, IORING_OFF_SQ_RING);
```
<!-- Where params is the io_uring_params structure, and ring_fd is the file descriptor returned from io_uring_setup(). The addition of params.sq_off.array to the length of the region accounts for the fact that the ring is not located right at the beginning. The actual array of submission-queue entries, instead, is mapped with: -->

params의 io_uring_params 구조체에서, ring_fd는 io_uring_setup()에서 리턴된 fd입니다. params.sq_off.array에 영역의 길이를 더하는 것(params.sq_off.array + params.sq_entries*sizeof(__u32))은 ring이 정확히 처음에 위치하지 않는 사실을 나타냅니다. 대신에 실제 submission queue 엔트리의 길이는 아래와 같은 방법으로 매핑합니다.
```c
    sqentries = mmap(0, params.sq_entries*sizeof(struct io_uring_sqe),
    		    PROT_READ|PROT_WRITE|MAP_SHARED|MAP_POPULATE,
		    ring_fd, IORING_OFF_SQES);
```
<!-- This separation of the queue entries from the ring buffer is needed because I/O operations may well complete in an order different from the submission order. The completion queue is simpler, since the entries are not separated from the queue itself; the incantation required is similar: -->

submission의 순서와는 다르게 IO 동작이 완료되기 때문에 ring buffer로부터 queue entries를 분리되어 있습니다. completion queue의 경우는 더 쉽습니다. queue와 엔트리가 분리되어 있지 않기 때문입니다. 사용하는 방법은 비슷합니다.
```c
    cqentries = mmap(0, params.cq_off.cqes + params.cq_entries*sizeof(struct io_uring_cqe),
    		    PROT_READ|PROT_WRITE|MAP_SHARED|MAP_POPULATE,
		    ring_fd, IORING_OFF_CQ_RING);
```
<!-- It's perhaps worth noting at this point that Axboe is working on a user-space library that hide will much of the complexity of this interface from most users. -->
Axboe가 userspace 라이브러리에서 작업을 하며 대부분의 사용자에게 인터페이스의 복잡도를 숨길 수 있다는 점에서 주목할 필요가 있습니다.

I/O submission
<!-- Once the io_uring structure has been set up, it can be used to perform asynchronous I/O. Submitting an I/O request involves filling in an io_uring_sqe structure, which looks like this (simplified a bit): -->
처음에 io_uring 구조체가 세팅되면, 비동기 IO를 수행하는 데 사용됩니다. io_uring_sqe 구조체를 채우고 IO 동작을 요청하며, io_uring_sqe는 약간 간소화되어 아래와 같습니다.
```c
    struct io_uring_sqe {
	__u8	opcode;		/* type of operation for this sqe */
	__u8	flags;		/* IOSQE_ flags */
	__u16	ioprio;		/* ioprio for the request */
	__s32	fd;		/* file descriptor to do IO on */
	__u64	off;		/* offset into file */
	void	*addr;		/* buffer or iovecs */
	__u32	len;		/* buffer size or number of iovecs */
	union {
	    __kernel_rwf_t	rw_flags;
	    __u32		fsync_flags;
	};
	__u64	user_data;	/* data to be passed back at completion time */
	__u16	buf_index;	/* index into fixed buffers, if used */
    };
```
<!-- The opcode describes the operation to be performed; options include IORING_OP_READV, IORING_OP_WRITEV, IORING_OP_FSYNC, and a couple of others that we will return to. There are clearly a number of parameters that affect how the I/O is performed, but most of them are relatively straightforward: fd describes the file on which the I/O will be performed, for example, while addr and len describe a set of iovec structures pointing to the memory where the I/O is to take place. -->

opcode는 수행될 동작을 나타냅니다. 옵션에는 IORING_OP_READV, IORING_OP_WRITEV, IORING_OP_FSYNC같은 것들이 있습니다. IO가 어떻게 동작할 지에 영향을 끼치는 여러 인자가 있습니다. 하지만 대부분은 상대적으로 간단합니다. 예를 들면 fd는 IO가 일어날 파일을 이야기하며, addr과 len은 IO가 일어나는 메모리를 가리키는 iovec 구조체를 나타냅니다.

<!-- As mentioned above, the io_uring_sqe structures are kept in an array that is mapped into both user and kernel space. Actually submitting one of those structures requires placing its index into the submission queue, which is defined this way: -->

위에서 이야기 했듯이, io_uring_sqe 구조체는 user와 kernel space에 매핑되는 배열에 저장됩니다. 사실은 이 구조체들 중 하나를 제출하려면 submission queue에 인덱스를 할당해야 하며, 아래와 같다.
```c
    struct io_uring {
	u32 head;
	u32 tail;
    };

    struct io_sq_ring {
	struct io_uring		r;
	u32			ring_mask;
	u32			ring_entries;
	u32			dropped;
	u32			flags;
	u32			array[];
    };
```
<!-- The head and tail values are used to manage entries in the ring; if the two values are equal, the ring is empty. User-space code adds an entry by putting its index into array[r.tail] and incrementing the tail pointer; only the kernel side should change r.head. Once one or more entries have been placed in the ring, they can be submitted with a call to: -->

head와 tail 값은 ring의 엔트리를 관리합니다. 두 값이 같다면 ring은 비어있는 상태입니다. userspace 코드는 인덱스를 배열에 넣고 tail을 증가시킴으로서 엔트리를 추가합니다. 오직 커널만이 r.head를 바꿀 수 있습니다. 한개 이상의 엔트리가 ring에 들어가면, 한번의 호출로 제출할 수 있습니다.

    int io_uring_enter(unsigned int fd, u32 to_submit, u32 min_complete, u32 flags);

<!-- Here, fd is the file descriptor associated with the ring, and to_submit is the number of entries in the ring that the kernel should submit at this time. The return value should be zero if all goes well. -->
여기 fd는 ring과 관련되어 있으며 to_submit은 커널이 제출해야하는 링 안의 엔트리의 숫자입니다. 리턴값은 성공하면 0입니다.

<!-- Completion events will find their way into the completion queue as operations are executed. If flags contains IORING_ENTER_GETEVENTS and min_complete is nonzero, io_uring_enter() will block until at least that many operations have completed. The actual results can be found in the completion structure: -->
completion 이벤트는 동작이 실행되면 completion queue로 갑니다. flags가 IORING_ENTER_GETEVENTS를 포함하고 min_complete가 0이 아니면, io_uring_enter()는 동작이 완료 될때까지 block됩니다. 실제 결과는 completion 구조체를 통해서 얻을 수 있습니다.
```c
    struct io_uring_cqe {
	__u64	user_data;	/* sqe->user_data submission passed back */
	__s32	res;		/* result code for this event */
	__u32	flags;
    };
```
<!-- Where user_data is a value passed from user space when the operation was submitted and res is the return code for the operation. The flags field will contain IOCQE_FLAG_CACHEHIT if the request could be satisfied without needing to perform I/O ? an option that may yet have to be reconsidered given the current concern about using the page cache as a side channel. -->
user_data는 userspace에서 제출되었을 때 값이 넘겨지고 res는 동작의 리턴 값입니다. 만약 리퀘스트가 IO를 수행할 필요없이 만족된다면 flags 필드는 IOCQE_FLAG_CACHEHIT을 가진다. 이것은 사이드 채널로서 페이지 캐시를 사용하는 것에 대한 우려를 생각해보면 다시 생각해봐야하는 옵션입니다.
<!--   
These structures live in the completion queue, which looks similar to the submission queue: -->
이 구조체들은 completion queue 에 있고 submission queue 와 비슷합니다:
```c
    struct io_cq_ring {
	struct io_uring		r;
	u32			ring_mask;
	u32			ring_entries;
	u32			overflow;
	struct io_uring_cqe	cqes[];
    };
```
<!-- In this ring, the r.head index points to the first available completion event, while r.tail points to the last; user space should only change r.head. -->
이 ring에서는 r.head 인덱스가 첫 completion event를 가리키며, r.tail은 마지막을 가리킵니다; userspace에서는 r.head만을 수정할 수 있습니다.

<!-- The interface as described so far is enough to enable a user-space program to enqueue multiple I/O operations and to collect the results as those operations complete. The functionality is similar to what the current AIO interface provides, though the interface is quite different. Axboe claims that it is far more efficient, but no benchmark results have been included yet to back up that claim. Among other things, this interface can do asynchronous buffered I/O without a context switch in cases where the desired data is in the page cache; buffered I/O has always been a bit of a sore spot for Linux AIO. -->

위에서 설명한 인터페이스들은 userspace 프로그램에서 다중 IO 동작을 넣고 그에 대한 결과를 받기에는 충분합니다. 기능적인 측면에서 현재 AIO 인터페이스가 제공하는 것과 비슷하지만 인터페이스는 많이 다릅니다. Axboe는 훨씬 다르다고 주장하나 이것을 뒷바침할 벤치마크 결과가 없습니다. 또 다른 것은 이 인터페이스는 페이지 캐시의 몇몇 경우에는 context switch없이 비동기 buffered IO를 할 수 있다는 것입니다. buffered IO는 Linux AIO의 아픈 손가락입니다.

<!-- Advanced features

There are, however, some more features worthy of note in this interface. One of those is the ability to map a program's I/O buffers into the kernel. This mapping normally happens with each I/O operation so that data can be copied into or out of the buffers; the buffers are unmapped when the operation completes. If the buffers will be used many times over the course of the program's execution, it is far more efficient to map them once and leave them in place. This mapping is done by filling in yet another structure describing the buffers to be mapped: -->

고급 기능:
이 인터페이스는 주목할 만한 기능이 몇 가지 더 있습니다. 프로그램의 IO 버퍼를 커널에 매핑하는 것입니다. 보통 각 IO 연산 데이터가 버퍼 안밖으로 복사 될 때 일어납니다. 버퍼들은 연산이 끝나면 매핑이 해제됩니다. 버퍼들이 프로그램 실행 중에 여러번 사용된다면 한번 매핑해두는 것보다 훨씬 효율적일 것입니다.

```c
    struct io_uring_register_buffers {
	struct iovec *iovecs;
	__u32 nr_iovecs;
    };
```

<!-- That structure is then passed to another new system call: -->
이 구조체는 새로운 시스템 콜로 전달된다.

```c
    int io_uring_register(unsigned int fd, unsigned int opcode, void *arg);
```

<!-- In this case, the opcode should be IORING_REGISTER_BUFFERS. The buffers will remain mapped for as long as the initial file descriptor remains open, unless the program explicitly unmaps them with IORING_UNREGISTER_BUFFERS. Mapping buffers in this way is essentially locking memory into RAM, so the usual resource limit that applies to mlock() applies here as well. When performing I/O to premapped buffers, the IORING_OP_READ_FIXED and IORING_OP_WRITE_FIXED operations should be used. -->

이 경우, opcode는 IORING_REGISTER_BUFFERS가 됩니다. 프로그램에서 IORING_UNREGISTER_BUFFERS를 통해서 명시적으로 unmap을 하지 않는 이상, 초기 fd가 open된 상태와 함께 버퍼의 매핑이 유지됩니다. 버퍼를 mapping하는 것은 메모리를 RAM에 locking 하는 것입니다. mlock()에 사용되는 리소스 한도가 여기에도 적용된다. 매핑된 버퍼에 IO동작을 수행하는 것은 IORING_OP_READ_FIXED와 IORING__OP_WRITE_FIXED가 사용됩니다. 

<!-- There is also an IORING_REGISTER_FILES operation that can be used to optimize situations where many operations will be performed on the same file(s). -->
IORING_REGISTER_FILES라는 옵션도 있는 데 이것은 같은 파일에 여러 연산이 동작할 때 최적화를 위해서 사용합니다.

<!-- In many high-bandwidth settings, it can be more efficient for the application to poll for completion events rather than having the kernel collect them and wake the application up; that is the motivation behind the existing block-layer polling interface, for example. Polling is most efficient in situations where, by the time the application gets around to doing a poll, there is almost certainly at least one completion ready for it to consume. This polling mode can be enabled for io_uring by setting the IORING_SETUP_IOPOLL flag when calling io_uring_setup(). In such rings, an occasional call to io_uring_enter() (with the IORING_ENTER_GETEVENTS flag set) is mandatory to ensure that completion events actually make it into the completion queue. -->

많은 high-bandwidth 경우에서, completion event를 polling하는 것이 커널이 이벤트를 통해서 어플리케이션을 깨워주는 방식보다 효율적입니다. 예를 들면, 이것은 기존에 존재하는 block-layer polling interface의 동기가 됩니다. 적어도 하나의 사용될 completion 이벤트가 있고 이것을 어플리케이션이 polling 을 하고 있을 때 polling은 가장 효율적입니다. 이 polling mode는 io_uring의 io_uring_setup()을 호출할 때 IORING_SETUP_IOPOLL플래그를 세팅하면 동작합니다. completion 이벤트를 completion 큐에 작성되도록 하려면 io_uring_enter()를 호출해야 합니다.

<!-- Finally, there is also a fully polled mode that (almost) eliminates the need to make any system calls at all. This mode is enabled by setting the IORING_SETUP_SQPOLL flag at ring setup time. A call to io_uring_enter() will kick off a kernel thread that will occasionally poll the submission queue and automatically submit any requests found there; receive-queue polling is also performed if it has been requested. As long as the application continues to submit I/O and consume the results, I/O will happen with no further system calls. -->

마지막으로, 시스템 콜이 거의 필요없는 완전한 polled mode도 있습니다. 이 모드는 IORING_SETUP_SQPOLL 플래그를 세팅하면 가능합니다. io_uring_enter()를 호출하면 커널스레드가 실행되고 submission queue를 polling하며 거기에 있는 요청들을 제출합니다; 요청받으면 receive-queue polling 또한 수행합니다. 어플리케이션이 IO를 요청하고 결과를 받을 동안 시스템 콜 없이 IO가 발생합니다. 

<!-- Eventually, though (after one second currently), the kernel will get bored if no new requests are submitted and the polling will stop. When that happens, the flags field in the submission queue structure will have the IORING_SQ_NEED_WAKEUP bit set. The application should check for this bit and, if it is set, make a new call to io_uring_enter() to start the mechanism up again. -->

결론적으로, 새로운 request가 없고 polling 이 멈추면 커널이 지루할 것입니다. 그 때에는 submission queue에 IORING_SQ_NEED_WAKEUP가 세팅될 것입니다. 어플리케이션에서는 비트를 체크해서 세팅되어 있다면 io_uring_enter()를 호출하여 다시 메커니즘을 실행해야 합니다.

<!-- This patch set is in its third version as of this writing, though that is a bit deceptive since there were (at least) ten revisions of the polled AIO patch set that preceded it. While it is possible that the interface is beginning to stabilize, it would not be surprising to see some significant changes yet. One review comment that has not yet been addressed is Matthew Wilcox's request that the name be changed to "something that looks a little less like io_urine". That could yet become the biggest remaining issue ? as we all know, naming is always the hardest part in the end. But, once those details are worked out, the kernel may yet have an asynchronous I/O implementation that is not a constant source of complaints. -->

이 패치 셋은 이번에 작성한 것이 세번째 버전입니다. 이전에 최소 10개의 개정된 polled AIO 패치가 있었기 때문에 약간 기만적입니다. 반면에 이제 안정화를 시작할 수 있어서 아직 특별히 큰 변경사항이 없다는 것이 그다지 놀랍지 않습니다. 이름을 "io_urine과 덜 비슷한 것"을 바꿔야 한다는 Matthew Wilcox의 요청은 아직 검토하지 않았습니다. 이것이 남은 가장 큰 이슈가 될 수 있습니다. 모두들 알듯이, 이름짓는 것은 마지막의 가장 어려운 부분입니다. 그러나 이러한 세세한 부분이 해결되면 리눅스 커널은 불만 없는 비동기 IO 구현체를 가질 것입니다.

<!-- For the curious, Axboe has posted a complete example of a program that uses the io_uring interface.  -->
궁금증을 위해, Axobe는 io_uring 인터페이스의 완전한 예제 프로그램을 게시했습니다.

원문 : [LWN: Ringing in a new asynchronous I/O API](https://lwn.net/Articles/776703/)
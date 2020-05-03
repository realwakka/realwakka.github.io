---
title: "쉬운 liburing example"
comments: "true"
categories: "linux"
tags:
  - "linux"
  - "kernel"
  - "io_uring"
  - "example"

---

test.txt에 간단하게 hello world를 작성하는 예제입니다.

```c

#include <liburing.h>
#include <unistd.h>
#include <fcntl.h>
#include <stdio.h>

int main()
{
  struct io_uring ring;
  io_uring_queue_init(32, &ring, 0);

  struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
  int fd = open("test.txt", O_WRONLY | O_CREAT);
  
  struct iovec iov = {
    .iov_base = "Hello world",
    .iov_len = strlen("Hello world"),
  };
  io_uring_prep_writev(sqe, fd, &iov, 1, 0);
  io_uring_submit(&ring);

  struct io_uring_cqe *cqe;

  for (;;) {
    io_uring_peek_cqe(&ring, &cqe);
    if (!cqe) {
      puts("Waiting...");
    } else {
      puts("Finished.");
      break;
    }
  }
  io_uring_cqe_seen(&ring, cqe);
  io_uring_queue_exit(&ring);
  close(fd);
}
```

간단한 설명!

```c
 struct io_uring ring;
 ```
 간단한 io_uring을 만듭니다. 안에는 fd와 sq와, cq가 들어있습니다.
 
 sq는 submission queue로 하고자하는 작업 (여기서는 write)를 submit하는 queue입니다.
 cq는 completion queue로 submit된 작업이 완료되면 커널에서 이 queue에 작업 결과를 넣어둡니다.

```c
io_uring_queue_init(32, &ring, 0);
```

io_uring을 초기화하는 함수입니다. 32는 엔트리 개수입니다. sq와 cq의 크기를 의미합니다. 마지막 파라미터인 flags는 특별한 옵션이 없으면 0으로 설정합니다.

```c
struct io_uring_sqe *sqe = io_uring_get_sqe(&ring);
```
io_uring에서 sqe를 얻습니다. 이 sqe에 원하는 작업에 대한 정보를 넣어서 submit할 예정입니다.

```c
struct iovec iov = {
    .iov_base = "Hello world",
    .iov_len = strlen("Hello world"),
  };
```

작업을 submit할 때 쓰이는 iovec 구조체입니다. .iov_base에는 사용되는 메모리 버퍼의 주소를 쓰면되고 .iov_len은 길이를 나타냅니다.

```c
io_uring_prep_writev(sqe, fd, &iov, 1, 0);
```

위에서 얻은 sqe에 작업에 대한 내용을 적어둡니다. 쓰려고 하는 파일의 fd와 위에서 만든 iov를 넣습니다. 그 다음에는 iov의 개수(여기서는 하나)를 넣어주고 마지막으로 파일에서 쓰려고하는 오프셋(0)을 넣어줍니다.

이외에도 `io_uring_prep_readv()`가 있습니다.


```c
io_uring_peek_cqe(&ring, &cqe);
```

ring의 cq에서 cqe를 하나 peek합니다. 만약 cq에 아무것도 없으면 cqe가 NULL로 나옵니다.

여기서는 peek을 통해서 busy waiting을 사용하였지만 필요한 경우 `io_uring_wait_cqe()`를 사용할 수도 있습니다.

```c
io_uring_cqe_seen(&ring, cqe);
```

cqe를 다 사용하였다 라는 의미로 이 함수를 호출해서 cq에서 제거합니다.

```c
io_uring_queue_exit(&ring);
```

io_uring를 제거하고 리소스를 반환합니다.




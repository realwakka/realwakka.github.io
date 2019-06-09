---
title: "The Linux 'copy problem'"
comments: "true"
categories: "linux"
---

<!--
In a filesystem session on the third day of the 2019 Linux Storage, Filesystem, and Memory-Management Summit (LSFMM),
Steve French wanted to talk about copy operations.
Much of the development work that has gone on in the Linux filesystem world over the last few years has been
related to the performance of copying files, at least indirectly, he said. There are still pain points around copy
operations, however, so he would like to see those get addressed.
-->

2019 LSFMM의 세 번째 날의 파일시스템 세션에 Steve French의 복사 연산을 주제로 발표가 있었습니다.
많은 리눅스의 파일 복사에 대한 개발이 최근 몇년 사이에 많이 있었습니다. 하지만 아직도 뼈아픈 단점들이 있고 그것에 대한 이야기를 하려고 합니다.

<!--
The "copy problem" is something that has been discussed at LSFMM before, French said,
but things have gotten better over the last year due to efforts by Oracle, Amazon, Microsoft, and others.
Things are also changing for copy operations; many of them are done to and from the cloud, which has to deal with
a wide variation in network latency. At the other end, NVMe is making larger storage faster at a relatively
affordable price. Meanwhile virtualization is taking more CPU, at times, because operations that might have been
offloaded to network hardware are being handled by the CPU.
-->

복사 문제는 LSFMM에서 이전에 다루었던 문제이긴합니다. 하지만 오라클, 아마존, 마이크로소프트 등등의 노력으로 좀 더 발전하고 있습니다.
복사 연산 또한 많은 변화 생기고 있습니다. 많은 연산이 클라우드에서 이루어지며 그것은 네트워크 지연과 많은 연관이 있습니다.
또한 NVMe는 큰 용량과 빠른 속도를 적절한 가격에 제공하고 있습니다. 가상화는 많은 CPU연산을 필요로 하지만 네트워크 장치로 념긴 연산은 CPU에서 처리됩니다.

<!--
[Steve French]
But copying files is one of the simplest, most intuitive operations for users; people do it all the time.
 He made multiple copies of his presentation slides in various locations, for example.
 Some of the most common utilities used are rsync, which is part of the Samba tools, scp from OpenSSH, and cp from the coreutils.
-->

그러나 파일을 복사하는 것은 정말 간단하고 사용자에게 직관적입니다. 사람들이 계속 사용합니다.
여러가지 슬라이드를 여러 위치에 복사하는 것도 하나의 예시입니다.
가장 일반적인 유틸은 rsync가 있습니다. Samba, scp, cp의 일부입니다.

<!--
The source code for cp is "almost embarrassingly small" at around 4K lines of code; scp is about the same and r
sync is somewhat larger. They each have to deal with some corner cases as well. He showed some examples of the
amount of time it takes to copy files on Btrfs and ext4 on two different drives attached to his laptop, one faster
and one slower. On the slow drive with Btrfs, scp took almost five times as long as cp for a 1GB copy.
On the fast drive, for a 2GB copy on ext4, cp took 1.2s (1.7s on the slow), scp 1.7s (8.4s), and rsync took 4.3s (not run on the slow drive, apparently).
These represent "a dramatic difference in performance" for a "really stupid" copy operation
-->

`cp`의 소스코드는 당황스러울리만큼 간단합니다. 4000줄 정도 됩니다. scp는 rsync와 비슷하거나 좀 더 깁니다. scp는 좀 더 특이 케이스들이 많습니다.
그는 노트북에서 Btrfs와 ext4의 2개의 드라이버에서 복사에 걸린 시간을 보여주었습니다. Btrfs에서는 느립니다. scp가 거의 1기가를 복사하는 데 cp보다 거의 5배가 들었습니다.
ext4에서는 2기가를 복사하는데 cp는 1.2초 (btrfs에서는 1.7초), scp는 1.7초 (btrfs 8.4초)가 걸렸고 rsync는 4.3초 (btrfs에서는 실행하지 않음)가 걸렸습니다.
이것은 복사연산에 대한 엄청난 성능차이를 보여줍니다.

<!--
The I/O size for cp is 128K and the others use 16K, which explains some of the difference, he said.
These copies are all going through the page cache, which is kind of odd because you don't normally need the data
you just copied again. None of the utilities uses O_DIRECT, if they did there would an improvement in
the performance of a few percent, he said. Larger I/O sizes would also improve things.
-->

보통 `cp`에 대한 I/O 크기는 128K이며 다른 것들은 16K입니다. 이것이 차이를 가져오는 것이죠.
복사하는 것은 페이지 캐시와 연관이 깊습니다. 이건 좀 이상한 것 같습니다! 한번 복사한 다음에는 캐시가 필요하진 않으니 말이죠.
유틸리티들은 `O_DIRECT`를 사용하지 않습니다. 만약에 사용했으면 약간의 성능이 올라갔겠지만 I/O 크기를 키우는 것이 더 좋은 성능 향상을 이끌어냅니다.

<!--
There are alternative tools that make various changes to improve performance.
For example, parcp and parallel parallelize copy operations. In addition, fpart and fpsync can parallelize
the operations that copy directories. Beyond that, Mutil is a parallel copy that is based on the cp and md5sum code
from coreutils; it comes out of a ten-year old paper [PDF] covering some work that NASA did on analyzing copy
performance because the agency found Linux cp to be lacking. The code never went upstream, however, so it can't even
be built at this point, French said.
-->

이것을 대체할 수 있는 다른 여러가지 툴이 있습니다. `parcp`와 복사연산을 병렬화 하는것입니다. 추가적으로 `fpart`와 `fpsync` 또한 디렉토리 복사를 병렬화 할 수 있습니다.
이것 시아으로 'Multi'는 'cp'와 'md5sum'을 사용해서 만든 병렬 복사입니다.

Cluster environments and network filesystems would rather have the server handle the copies directly using copy offload. Cloud providers would presumably prefer to have their backends handle copy operations rather than have them done directly from clients, he said. Also parallelization is a need because the common tools overuse one processor rather than spreading the load, especially if you are doing any encryption. In addition, local cross-mount copies are not being done efficiently; he believes that Linux filesystems could do a much better job in the kernel than cp does in user space even if they were copying between two different mounted filesystems of the same type.

Luis Chamberlain asked if French had spoken about these issues at conferences that had more of a user-space focus. The problems are real, but it is not up to kernel developers to fix them, he said. In addition, any change to parallelize, say, cp would need to continue to operate serially in the default case for backward compatibility. In the end, these are user-space problems, Chamberlain said.

In the vast majority of cases for the open-source developers of these tools, it is the I/O device that is the bottleneck, Ted Ts'o said. If you have a 500-disk RAID array, parallel cp makes a lot of sense, but the coreutils developers are not using those kinds of machines. Similarly, "not everyone is bottlenecked on the network"; those who are will want copy offload. More progress will be made by targeting specific use cases, rather than some generic "copy problem", since there are many different kinds of bottlenecks at play here.

French said that he strongly agrees with that. The problem is that when people run into problems with copy performance on SMB or NFS, they contact him. Other types of problems lead users to contact developers of other kernel filesystems. For example, he said he was tempted to track down a Btrfs developer when he was running some of his tests that took an inordinate amount of time on that filesystem.

Chris Mason said that if there are dramatically different results from local copies on the same device using the standard utilities, it probably points to some kind of bug in buffered I/O. The I/O size should not make a huge difference as the readahead in the kernel should keep the performance roughly the same. French agreed but said that the copy problem in Linux is something that is discussed in multiple places. For example, Amazon has a day-long tutorial on copying for Linux, he said; "it's crazy". This is a big deal for many, "and not just local filesystems and not just clusters".

There are different use cases, some are trying to minimize network bandwidth, others are trying to reduce CPU use, still others have clusters that have problems with metadata updates. The good news is that all of these problems have been solved, he said, but the bad news is that the developers of cp, parcp, and others do not have the knowledge that the filesystems developers have, so they need advice.

Though there are some places "where our APIs are badly broken", he said. For example, when opening a file and setting a bunch of attributes, such as access control lists (ACLs), there are races because those operations cannot be done atomically. That opens security holes.

There are some things he thinks the filesystems could do. For example, Btrfs could support copy_file_range(); there are cases where Btrfs knows how to copy faster and, if it doesn't, user space can fall back to what it does today. There are five or so filesystems in the kernel that support copy_file_range() and Btrfs could do a better job with copies if this copy API is invoked; Btrfs knows more about the placement of the data and what I/O sizes to use.

Metadata is another problem area, French said. The race on setting ACLs is one aspect of that. Another is that filesystem-specific metadata may not get copied as part of a copy operation, such as file attributes and specialized ACLs. There is no API for user space to call that knows how to copy everything about a file; Windows and macOS have that, though it is not the default.

Ts'o said that shows a need for a library that provides ways to copy ACLs, extended attributes (xattrs), and the like. Application developers have told him that they truncate and rewrite files because "they are too lazy to copy the ACLs and xattrs", but then complain when they lose their data if the machine crashes. The solution is not a kernel API, he said.

But French is concerned that some of the xattrs have security implications (e.g. for SELinux), so he thinks the filesystems should be involved in copying them. Ts'o said that doing so in the kernel actually complicates the problem; SELinux is set up to make decisions about what the attributes should be from user space, doing it in the kernel is the wrong place. Mason agreed, saying there is a long history with the API for security attributes; he is "not thrilled" with the idea of redoing that work. He does think that there should be a way to create files with all of their attributes atomically, however.

There was more discussion of ways to possibly change the user-space tools, but several asked for specific ideas of what interfaces the kernel should be providing to help. French said that one example would be to provide a way to get the recommended I/O size for a file. Right now, the utilities base their I/O size on the inode block size reported for the file; SMB and NFS lie and say it is 1MB to improve performance.

But Mason said that the right I/O size depends on the device. Ts'o said that the st_blksize returned from stat() is the preferred I/O size according to POSIX; "whatever the hell that means". Right now, the filesystem block size is returned in that field and there are probably applications that use it for that purpose so a new interface is likely needed to get the optimal I/O size; that could perhaps be added to statx(). But if a filesystem is on a RAID device, for example, it would need to interact with the RAID controller to try to figure out the best I/O size; often the devices will not provide enough information so the filesystem has to guess and will get it wrong sometimes. That means there will need to be a way to override that value via sysfs.

Another idea would be to give user space a way to figure out if it makes sense to turn off the page cache, French said. But that depends on what is going to be done with the file after the copy, Mason said; if you are going to build a kernel with the copied files, then you want that data in the page cache. It is not a decision that the kernel can help with.

The large list of copy tools with different strategies is actually a good motivation not to change what the kernel does, Mason said. User space is the right place to have different policies for how to do a copy operation. French said that 90% of the complaints he hears are about cp performance. Several in the discussion suggested that volunteers or interns be found to go fix cp and make it smarter, but French would like to see the filesystem developers participate in developing the tools or at least advising those developers. Mason pointed out that kernel developers are not ambassadors to go fix applications across the open-source world, however; "our job is to build the interfaces", so that is where the focus of the discussion should be.

As the session closed, French said that Linux copy performance was a bit embarrassing; OS/2 was probably better for copy performance in some ways. But he did note that the way sparse files were handled using FIEMAP in cp was great. Ts'o pointed out that FIEMAP is a great example of how the process should work. Someone identified a problem, so kernel developers added a new feature to help fix it, and now that code is in cp; that is what should be happening with any other kernel features needed for copy operations.
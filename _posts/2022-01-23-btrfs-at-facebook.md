---
title: "쉬운 liburing example"
comments: "true"
categories: "linux"
tags:
  - "linux"
  - "kernel"
  - "btrfs"
  - "filesystem"
---

lwn 기사의 번역 글입니다.

<!-- The Btrfs filesystem has had a long and sometimes turbulent history; LWN first wrote about it in 2007. It offers features not found in any other mainline Linux filesystem, but reliability and performance problems have prevented its widespread adoption. There is at least one company that is using Btrfs on a massive scale, though: Facebook. At the 2020 Open Source Summit North America virtual event, Btrfs developer Josef Bacik described why and how Facebook has invested deeply in Btrfs and where the remaining challenges are. -->

Btrfs 파일시스템은 길고 가끔은 격동의 역사를 가지고 있다. LWN에서는 2007년에 첫 기사가 있었다. 그 기사는 이전까지 리눅스 파일시스템에는 없던 기능들을 제공했다. 그러나 안정성과 성능 문제가 광범위한 적용을 막고 있었다. 이럼에도 불구하고 큰 스케일에 btrfs을 사용하는 회사가 하나 있는 데 바로 페이스북이다. 2020 Open Source Summit North America 에서는 Btrfs 개발자 Josef Bacik 은 왜, 그리고 어떻게 페이스북이 Btrfs을 개발하고 있는 지, 남은 도전과제가 무엇인지를 이야기했다.

<!-- Every Facebook service, Bacik began, runs within a container; among other things, that makes it easy to migrate services between machines (or even between data centers). Facebook has a huge number of machines, so it is impossible to manage them in any sort of unique way; the company wants all of these machines to be as consistent as possible. It should be possible to move any service to any machine at any time. The company will, on occasion, bring down entire data centers to test how well its disaster-recovery mechanisms work. -->

 Bacik은 모든 페이스북 서비스는 컨테이너에서 실행된다고 시작했다. 이렇게되면 서비스들을 다른 장비 (데이터 센터까지도)로 이식하기가 쉬워진다. 페이스북은 굉장히 많은 장비를 가지고 있고, 보통 회사가 장비들을 일정하게 관리하고 싶어하는 데 이러한 방법으로는 이 정도의 장비들을 관리하는 것이 불가능하다. 모든 서비스가 모든 장비에 모든 시간에 적용될 수 있어야 한다. 때때로 회사에서는 모든 데이터를 내리고 재해복구 시스템을 테스트하기도 한다.

<!-- Faster testing and more
All of these containerized services are using Btrfs for their root filesystem. The initial use case within Facebook, though, was for the build servers. The company has a lot of code, implementing the web pages, mobile apps, test suites, and the infrastructure to support all of that. -->

빠른 테스팅과 그 이상
 컨테이너화 된 모든 서비스는 루트 파일시스템으로 Btrfs를 사용한다. 비록 초기에는 빌드 서버에서만 사용되었지다. 회사에서는 웹페이지, 모바일앱, 테스트, 이것을 지원하는 모든 환경에 대한 코드가 있다.

<!-- The Facebook workflow dictates that nobody commits code directly to a repository. Instead, there is a whole testing routine that is run on each change first. The build system will clone the company repository, apply the patch, build the system, and run the tests; once that is done, the whole thing is cleaned up in preparation for the next patch to test. That cleanup phase, as it turns out, is relatively slow, averaging two or three minutes to delete large directory trees full of code. Some tests can take ten minutes to clean up, during which that machine is unavailable to run the next test. -->

 페이스북 워크플로우는 누구도 저장소에 직접 커밋하지 않는다. 대신에 많은 테스팅 루틴이 각 변경 마다 진행된다. 빌드 시스템은 회사 저장소를 복사하고 패치를 적용하고 빌드하고 테스트를 진행한다. 이것이 완료되면 클린업하고 다음 테스트를 위한 준비를 진행한다. 이 중에 클린업 단계가 상대적으로 느린데 평균적으로 큰 디렉토리를 지우는 데 2-3분이 걸린다. 일부 테스트는 클린업 단계가 10분까지 걸려 다음 테스트가 그때가지 진행이 안된다.

<!-- The end result was that developers were finding that it would take hours to get individual changes through the process. So the infrastructure team decided to try using Btrfs. Rather than creating a clone of the repository, the test system just makes a snapshot, which is a nearly instantaneous operation. After the tests are run, the snapshot is deleted, which also appears to be instantaneous from a user-space point of view. There is, of course, a worker thread actually cleaning up the snapshot in the background, but cleaning up a snapshot is a lot faster than removing directories from an ordinary filesystem. This change saves a lot of time in the build system and reduced the capacity requirement — the number of machines needed to do builds and testing — by one-third. -->

 그로 인해 개발자들은 각각의 변경이 몇시간이 걸리기도 한다. 그래서 인프라팀은 Btrfs를 사용하기로 하였다. 저장소를 clone하는 것이 아닌 테스트 시스템이 명령에 즉각 반응하는 스냅샷을 만드는 것이다. 테스트가 실행된 후에는 스냅샷은 삭제되며 사용자의 관점에서 즉각 반영된다. 실제로는 워커스레드가 스냅샷을 백그라운드에서 제거하며 스냅샷을 제거하는 것은 실제 파일시스템을 제거하는 것보다 훨씬 빠르다. 이러한 변화를 통해서 빌드 시스템에서 많은 시간을 절약하고 용량 요구사항을 줄였다. 빌드와 테스팅에 필요한 장비의 수가 1/3 수준으로 줄었다.

<!-- After that experiment worked out so well, the infrastructure team switched fully to Btrfs; the entire build system now uses it. It turns out that there was another strong reason to switch to Btrfs: its support for on-disk compression. The point here is not just saving storage space, but also extending its lifetime. Facebook spends a lot of money on flash storage — evidently inexpensive, low-quality flash storage at that. The company would like this storage to last as long as possible, which implies minimizing the number of write cycles performed. Source code tends to compress well, so compression reduces the number of blocks written considerably, slowing the process of wearing out the storage devices. -->

 이러한 실험이 성공하고 난 후, 인프라팀은 완전히 Btrfs로 전환하였다. 모든 빌드 시스템이 사용하고 있다. 또 하나의 이유가 있었는 데, 바로 온디스크 압축 지원이다. 요점은 단순히 스토리지 공간을 절약하는 것 뿐만 아니라 저장소의 라이프타임을 증가시키는 데 있다. 페이스북은 플래시 스토리지에 많은 돈을 쓰고 있고 이는 엄청 비싸고 저품질의 플래시 스토리지이다. 회사에서는 이 스토리지를 될수 있으면 길게 사용하고 싶고 이는 쓰기 사이클을 최소화하는 것을 의미한다. 소스 코드는 압축이 보통 압축이 잘되고 압축은 쓰여지는 블록을 줄이고 스토리지의 마모를 지연 시키는 효과가 있다.

<!-- This work, Bacik said, was done by the infrastructure team without any sort of encouragement by Facebook's Btrfs developers; indeed, they didn't even know that it was happening. He was surprised by how well it worked in the end. -->

 Bacik은 이것은 Btrfs 개발자의 어떠한 권유 없이 인프라팀에서 이루어졌다고 밝혔다. 사실 그들은 이 사실을 알지도 못했다. 그는 이 일이 잘되었다는 것에 놀랐다고 한다.

<!-- Then, there is the case of virtual machines for developers. Facebook has an "unreasonable amount" of engineering staff, Bacik said; each developer has a virtual machine for their work. These machines contain the entire source code for the web site; it is a struggle to get the whole thing to fit into the 800GB allotted for each and still leave room for some actual work to be done. Once again, compression helps to save the day, and works well, though he did admit that there have been some "ENOSPC issues" (problems that result when the filesystem runs out of available space). -->

 가상 머신의 케이스도 있다. 페이스북은 말도 안되는 양의 엔지니어링 스탭들이 있고 각 개발자들은 업무상 가상머신을 하나씩 가지고 있다. 이 머신들은 웹사이트의 전체코드를 지니고 있다. 모든 것을 각각의 800GB의 할당된 공간에  맞춰 넣고 실제 작업을 수행할 여지를 두는 것은 쉽지 않은 일이다. 다시 말하지만 압축은 시간을 절약하고 잘동작하지만 ENOSPC 이슈가 있다는 것을 인정했습니다.

<!-- Another big part of the Facebook code base is the container system itself, known internally as "tupperware". Containers in this system use Btrfs for the root filesystem, a choice that enables a number of interesting things. The send and receive mechanism can be used both to build the base image and to enable fast, reliable upgrades. When a task is deployed to a container, a snapshot is made of the base image (running a version of CentOS) and the specific container is loaded on top. When that service is done, cleanup is just a matter of deleting the working subvolume and returning to the base snapshot. -->

 또 다른 페이스북 코드 베이스의 큰 부분은 컨테이너 시스템 그자체이다. 내부적으로 tupperware라고 부른다. Btrfs를 루트 파일시스템으로 사용하는 컨테이너는 몇가지 재미있는 기능을 사용할 수 있다. send와 receive의 메커니즘이 베이스 이미지를 만드는 것과 빠르고 유연한 업그레이드를 수행하는 데에 사용될 수 있습니다. 테스크가 컨테이너에 배포되면, 베이스 이미지로 만들어진 스냅샷 (CentOS 버전을 실행) 특정 컨테이너가 탑에 로딩됩니다. 서비스가 끝나면 클린업은 서브볼륨의 삭제와 베이스 스냅샷으로 돌아갑니다.

<!-- Additionally, Btrfs compression once again reduces write I/O, helping Facebook to make the best use of cheap flash drives. Btrfs is also the only Linux filesystem that works with the io.latency and io.cost (formerly io.weight) block I/O controllers. These controllers don't work with ext4 at all, he said, and there are some problems still with XFS; he has not been able to invest the effort to make things work better on those filesystems. -->
추가적으로, Btrfs 압축은 쓰기 IO를 줄여 페이스북의 값싼 플래시 드라이브를 최적으로 사용할 수 있게합니다. Btrfs는  io.latency와 io.cost(이전에는 io.weight) 블록 IO 컨트롤러를 통해서 동작하는 유일한 리눅스 파일시스템입니다. 이 컨트롤러는 ext4에서는 전혀 작동하지 않고 XFS와도 여전히 문제가 있습니다. 그는 이 파일시스템들을 더 잘 작동하기 위한 노력을 투자하기가 어려웠습니다. 

<!-- An in-progress project concerns the WhatsApp service. WhatsApp messages are normally stored on the users' devices, but they must be kept centrally when the user is offline. Given the number of users, that's a fair amount of storage. Facebook is using XFS for this task, but has run into unspecified "weird scalability issues". Btrfs compression can help here as well, and snapshots will be useful for cleaning things up. -->
프로젝트 진행 도중에 WhatsApp 서비스에 관여하게 되었습니다. WhatsApp 메세지는 유저 장치에 저장되는 데, 그러나 오프라인 일때에는 서버에 저장되어야 했습니다. 사용자 수에 따라 정말 큰 양의 스토리지입니다. 페이스북은 이작업에 XFS를 사용하고 있습니다. 그러나 "이상한 확장성 이슈"에 빠지게 됩니다. Btrfs 압축은 이를 해결할 수 있고 스냅샷 기능도 클린업 시에 유용합니다.

<!-- But Btrfs, too, has run into scalability problems with this workload. Messages are tiny, compressed files; they are small enough that the message text is usually stored with the file's metadata rather than in a separate data extent. That leads to filesystems with hundreds of gigabytes of metadata and high levels of fragmentation. These problems have been addressed, Bacik said, and it's "relatively smooth sailing" now. That said, there are still some issues left to be dealt with, and WhatsApp may not make the switch to Btrfs in the end. -->
그러나 Btrfs 역시 확장성 문제가 있었습니다. 메세지는 작고 압축된 형태의 파일입니다. 메세지 텍스트는 메타데이터와 따로가 아닌 같이 저장될 만큼 크기가 작습니다. 그래서 이것은 수백 기가의 메타데이터를 만들어 내며 이것은 파일시스템의 높은 파편화를 가져옵니다. Bacik이 말하길 이러한 문제가 해결되면서 "비교적 순항중이다" 라고 말했습니다. 이것은 논의할 약간의 이슈가 남아 있어, WhatsApp은 결국 Btrfs로 전환하지 못할 수도 있습니다. 

<!-- The good, the bad, and the unresolved
Bacik concluded with a summary of what has worked well and what has not. He told the story of tracking down a bug where Btrfs kept reporting checksum errors when working with a specific RAID controller. Experience has led him to assume that such things are Btrfs bugs, but this time it turned out that the RAID controller was writing some random data to the middle of the disk on every reboot. This problem had been happening for years, silently corrupting filesystems; Btrfs flagged it almost immediately. That is when he started to think that, perhaps, it's time to start trusting Btrfs a bit more. -->

좋은 점, 나쁜 점, 아직 남은 문제들
Bacik 은 무엇지 잘되었고 아닌지를 요약하면서 결론을 내렸다. 그는 특정 RAID 컨트롤러에서 동작하는 Btrfs의 checksum 에러들을 트래킹하는 이야기를 하였다. 그 경험이 Btrfs 버그들을 예측하게 만들었고 지금 RAID 컨트롤러가 재부팅할 때마다 무작위 데이터를 쓴다는 것을 발견하였다. 이 문제는 수 년간 일어났고 조용히 파일시스템을 망가뜨리고 있었다. Btrfs는 이것을 거의 즉시 인지했다. 그는 Btrfs를 믿기 시작하는 시간이라고 생각하기 시작했다.

<!-- Another unexpected benefit was the help Btrfs has provided in tracking down microarchitectural processor bugs. Btrfs tends to stress the system's CPU more than other filesystems; features like checksumming, compression, and work offloaded to threads tend to keep things busy. Facebook, which builds its own hardware, has run into a few CPU problems that have been exposed by Btrfs; that made it easy to create reproducers to send to CPU vendors in order to get things fixed. -->

다른 예상치 못한 이득은 Btrfs는 마이크로 아키텍처 프로세서의 버그의 추적을 제공한다는 것이다. Btrfs는 다른 파일시스템보다 CPU자원을 더 쓰는 경향이 있다. 체크섬, 압축, 스레드에서의 처리 작업은 CPU를 busy한 상태로 유지시킨다. 자기의 하드웨어를 직접 만드는 페이스북은 몇몇 CPU문제들이 Btrfs에 의해서 발견했다. 이것은 CPU 벤더에게 전달하는 reproducer를 만드는 것을 쉽게 하였다.

<!-- In general, he said, he has spent a lot of time trying to track down systemic problems in the filesystem. Being a filesystem developer, he is naturally conservative; he worries that "the world will burn down" and it will all be his fault. In almost every case, these problems have turned out to have their origin in the hardware or other parts of the system. Hardware, he said, is worse than Btrfs when it comes to quality. -->
보통 그는 파일시스템의 시스템적인 문제를 추적하려고 많은 시간을 할애한다. 파일시스템 개발자로 그는 자연스럽게 보수적이다. 그는 세상이 불탈까봐 걱정이고 그것이 자신의 탓일거라고 생각한다. 대부분의 케이스에서 이런 문제들은 하드웨어나 시스템의 다른 부분의 문제로 밝혀진다. 그는 품질적인 측면에서 하드웨어가 btrfs보다 나쁘다고 이야기했습니다.

<!-- What he was most happy with, though, was perhaps the fact that most Btrfs use cases in the company have been developed naturally by other groups. He has never gone out of his way to tell other teams that they need to use Btrfs, but they have chosen it for its merits anyway. -->
그가 가장 행복했던 것은 아마도 btrfs를 회사내에서 사용하는 사례가 자연스럽게 다른 그룹에서 개발되었다는 것이다. 그는 절대 다른팀에게  btrfs를 사용하라고 권하지 않았다. 그러나 그들은 나름대로의 이득을 위해 사용했다.

<!-- "It's not Btrfs if there isn't an ENOSPC issue", he said.All is not perfect, however. At the top of his list was bugs that manifest when Btrfs runs out of space, a problem that has plagued Btrfs since the beginning; "it's not Btrfs if there isn't an ENOSPC issue", he said, adding that he has spent most of his career chasing these problems. There are still a few bad cases in need of fixing, but these are rare occurrences at this point. He is relatively happy, finally, with the state of ENOSPC handling. -->
하지만 모든 것이 완벽하진 않듯이 Btrfs가 아니면 ENOSPC 이슈가 없을 것이라고 이야기했다. 그의 리스트 상단은 btrfs가 공간이 부족할때 생기는 문제였고 이 문제는 btrfs를 처음부터 괴롭혔다. "btrfs가 아니라면 ENOSPC 이슈가 없을 것" 이라고 이야기했다. 그리고 그는 대부분의 그의 커리어를 이 문제를 쫓는 데 보냈다. 여전히 약간의 수정이 필요한 나쁜 케이스가 있으나, 이것은 매우 드물게 나타난다. 결론 적으로 그는 상대적으로 ENOSPC 상태를 다루는데 있어 행복하다.

<!-- There have been some scalability problems that have come up, primarily with the WhatsApp workload as described above. These bugs have highlighted some interesting corner cases, he said, but haven't been really difficult to work out. There were also a few "weird, one-off issues" that mostly affected block subsystem maintainer Jens Axboe; "we like Jens, so we fixed it". -->
약간의 확장성 이슈가 나타났는데, 위에서 이야기한 WhatsApp 의 작업에서 발견되었다. 이 버그들은 약간 흥미 있는 코너 케이스를 강조했지만, 해결하기에는 그리 어렵지 않다고 이야기했다. 블록 서브시스템 관리자 Jens Axboe에 영향을 받은 약간의 "이상한 일회성 문제"가 있었다. 그리고 우리는 고쳤다.

<!-- At the top of the list of problems still needing resolution is quota groups; their overhead is too high, he said, and things just break at times. He plans to solve these problems within the next year. There are users who would like to see encryption at the subvolume level; that would allow the safe stacking of services with differing privacy requirements. Omar Sandoval is working on that feature now. -->

여전히 해결할 문제들 중에는 quota groups이 있습니다. 기능의 오버헤드가 너무 높고 때때로 부서집니다. 그는 내년에 이러한 문제들을 해결하려고 합니다. subvolume 레벨에서 암호화를 사용하려는 사용자가 있습니다; 이것은 각각 다른 프라이버시 요구사항의 서비스들을 안전하게 쌓을 수 있게합니다. Omar Sandoval이 이 기능에 작업 중입니다.

<!-- Then there is the issue of RAID support, another longstanding problem area for Btrfs. Basic striping and mirroring work well, and have done so for years, he said. RAID 5 and 6 have "lots of edge cases", though, and have been famously unreliable. These problems, too, are on his list, but solving them will require "lots of work" over the next year or two. -->

RAID 지원에도 또 하나의 오래된 이슈가 있다. Basic striping과 mirroring은 몇년 동안 잘 작동했다. RAID 5, 6은 많은 엣지 케이스가 있고 신뢰할 수 없는 것으로 유명하다. 이 문제들 역시 그의 리스트에 있으며 이 문제를 풀기 위해서는 내년 또는 내후년까지 많은 작업이 필요할 것이다.

원문 : [LWN: Btrfs at Facebook](https://lwn.net/Articles/824855/)

# SREの観点から見たGoogleの本番環境

脚本 : JC van Winkel
編集 : Betsy Beyer

Googleデータセンターは、従来のほとんどのデータセンターや小規模サーバーファームとは大きく異なる。これらの違いは、余分な問題と機会の両方をもたらす。この章では、Googleデータセンターの特徴である問題と機会について説明し、本全体で使用される用語を紹介する。

## ハードウェア

Googleのコンピューティングリソースのほとんどは、Googleが設計したデータセンターにより、独自の配電、冷却、ネットワーキング、およびコンピューティングハードウェアを備えている。「標準」のコロケーションデータセンターとは異なり、Googleが設計したデータセンターのコンピューティングハードウェアは、全体で同じ。サーバーハードウェアとサーバーソフトウェアの混乱を避けるために、本では次の用語を使用している。

- **マシン**
  - ハードウェア (またはおそらくVM)
- **サーバー**
  - サービスを実装するソフトウェア

マシンはどのサーバーでも実行できるため、特定のマシンを特定のサーバープログラム専用にすることはない。たとえば、メールサーバーを実行する特定のマシンはない。代わりに、リソースの割り当てはクラスタオペレーティングシステム**Borg**によって処理される。

**サーバー**という単語のこのような使用は珍しいことに気づいた。「ネットワーク接続を受け入れるバイナリ」は**マシン**で一般的に使用されているが、Googleでのコンピューティングについて語るときは、両者を区別することが重要。**サーバー**の使用法に慣れると、Google内だけでなく、この本の他の部分でも、この専門用語を使用することが理にかなっていることがさらに明らかになる。

[図2-1](https://landing.google.com/sre/sre-book/chapters/production-environment/#fig_production-environment_topology)は、Googleデータセンターのトポロジーを示している。

- 数十台のマシンが**ラック**に配置されている。
- ラックは**1列**に並んでいる。
- 1つ以上の行が**クラスタ**を形成する。
- 通常、**データセンター**の建物には複数のクラスタが収容されている。
- 近接して配置された複数のデータセンターの建物が**キャンパス**を形成する。

![Example Google datacenter campus topology.](https://lh3.googleusercontent.com/reety2gbKIv2DsOTZ_xlGO4NwoGNVMRJUchhm6Ov1PiOODJc5svag80YGi1W_-iIXpKjtR_5dU9PsvSoUnRCRMXeJzvbslEjwr6FUQ=s0)
図2-1 : Googleデータセンターキャンパストポロジーの例

特定のデータセンター内のマシンは相互に通信できる必要があるため、数万のポートを持つ非常に高速な仮想スイッチを作成した。これを実現するには、**Jupiter**という名前のClosネットワークファブリックに数百のGoogle製スイッチを接続する。最大の構成では、Jupiterはサーバー間で1.3Pbpsの2分割帯域幅をサポートする。

データセンターは、世界中に広がるバックボーンネットワーク**B4**で相互に接続されている。B4は、ソフトウェア定義のネットワークアーキテクチャ (OpenFlowオープン標準通信プロトコルを使用する)。適度な数のサイトに大量の帯域幅を提供し、柔軟な帯域幅割り当てを使用して平均帯域幅を最大化する。

# System Software That "Organizes" the Hardware

Our hardware must be controlled and administered by software that can handle massive scale. Hardware failures are one notable problem that we manage with software. Given the large number of hardware components in a cluster, hardware failures occur quite frequently. In a single cluster in a typical year, thousands of machines fail and thousands of hard disks break; when multiplied by the number of clusters we operate globally, these numbers become somewhat breathtaking. Therefore, we want to abstract such problems away from users, and the teams running our services similarly don’t want to be bothered by hardware failures. Each datacenter campus has teams dedicated to maintaining the hardware and datacenter infrastructure.

## Managing Machines

*Borg*, illustrated in [Figure 2-2](https://landing.google.com/sre/sre-book/chapters/production-environment/#fig_production-environment_borg), is a distributed cluster operating system [[Ver15\]](https://landing.google.com/sre/sre-book/chapters/bibliography#Ver15), similar to Apache Mesos.[10](https://landing.google.com/sre/sre-book/chapters/production-environment/#id-BWDuecjF7IPTj) Borg manages its jobs at the cluster level.

![High-level Borg cluster architecture.](https://lh3.googleusercontent.com/u-b_doOMDhXmXkA52nLOu6OEn9sf_hXxku7Hp8GQe6xj_hO5RZ6OKuSbTa4MUE-71jTxGSJ3N2gjt9juDJ5hXsQCeIxFu37BmjblZw=s0)Figure 2-2. High-level Borg cluster architecture

Borg is responsible for running users’ *jobs*, which can either be indefinitely running servers or batch processes like a MapReduce [[Dea04\]](https://landing.google.com/sre/sre-book/chapters/bibliography#Dea04). Jobs can consist of more than one (and sometimes thousands) of identical *tasks*, both for reasons of reliability and because a single process can’t usually handle all cluster traffic. When Borg starts a job, it finds machines for the tasks and tells the machines to start the server program. Borg then continually monitors these tasks. If a task malfunctions, it is killed and restarted, possibly on a different machine.

Because tasks are fluidly allocated over machines, we can’t simply rely on IP addresses and port numbers to refer to the tasks. We solve this problem with an extra level of indirection: when starting a job, Borg allocates a name and index number to each task using the *Borg Naming Service* (BNS). Rather than using the IP address and port number, other processes connect to Borg tasks via the BNS name, which is translated to an IP address and port number by BNS. For example, the BNS path might be a string such as `/bns/<*cluster*>/<*user*>/<*job name*>/<*task number*>`, which would resolve to `<*IP address*>:<*port*>`.

Borg is also responsible for the allocation of resources to jobs. Every job needs to specify its required resources (e.g., 3 CPU cores, 2 GiB of RAM). Using the list of requirements for all jobs, Borg can binpack the tasks over the machines in an optimal way that also accounts for failure domains (for example: Borg won’t run all of a job’s tasks on the same rack, as doing so means that the top of rack switch is a single point of failure for that job).

If a task tries to use more resources than it requested, Borg kills the task and restarts it (as a slowly crashlooping task is usually preferable to a task that hasn’t been restarted at all).

## Storage

Tasks can use the local disk on machines as a scratch pad, but we have several cluster storage options for permanent storage (and even scratch space will eventually move to the cluster storage model). These are comparable to Lustre and the Hadoop Distributed File System (HDFS), which are both open source cluster filesystems.

The storage layer is responsible for offering users easy and reliable access to the storage available for a cluster. As shown in [Figure 2-3](https://landing.google.com/sre/sre-book/chapters/production-environment/#fig_production-environment_storage-stack), storage has many layers:

1. The lowest layer is called *D* (for *disk*, although D uses both spinning disks and flash storage). D is a fileserver running on almost all machines in a cluster. However, users who want to access their data don’t want to have to remember which machine is storing their data, which is where the next layer comes into play.
2. A layer on top of D called *Colossus* creates a cluster-wide filesystem that offers usual filesystem semantics, as well as replication and encryption. Colossus is the successor to GFS, the Google File System [[Ghe03\]](https://landing.google.com/sre/sre-book/chapters/bibliography#Ghe03).
3. There are several database-like services built on top of Colossus:
   1. Bigtable [[Cha06\]](https://landing.google.com/sre/sre-book/chapters/bibliography#Cha06) is a NoSQL database system that can handle databases that are petabytes in size. A Bigtable is a sparse, distributed, persistent multidimensional sorted map that is indexed by row key, column key, and timestamp; each value in the map is an uninterpreted array of bytes. Bigtable supports eventually consistent, cross-datacenter replication.
   2. Spanner [[Cor12\]](https://landing.google.com/sre/sre-book/chapters/bibliography#Cor12) offers an SQL-like interface for users that require real consistency across the world.
   3. Several other database systems, such as *Blobstore*, are available. Each of these options comes with its own set of trade-offs (see [Data Integrity: What You Read Is What You Wrote](https://landing.google.com/sre/sre-book/chapters/data-integrity)).

![Portions of the Google storage stack.](https://lh3.googleusercontent.com/vRFHzV6MGcmYL3af2KVAHkj4ODl5EcNAAlKV28S0cpf1dBvQii8SV_dguh9-8SqJFbFFhvI_wqlld1NsK2U5N3CYR6s8FRhRE8c=s0)Figure 2-3. Portions of the Google storage stack

## Networking

Google’s network hardware is controlled in several ways. As discussed earlier, we use an OpenFlow-based software-defined network. Instead of using "smart" routing hardware, we rely on less expensive "dumb" switching components in combination with a central (duplicated) controller that precomputes best paths across the network. Therefore, we’re able to move compute-expensive routing decisions away from the routers and use simple switching hardware.

Network bandwidth needs to be allocated wisely. Just as Borg limits the compute resources that a task can use, the Bandwidth Enforcer (BwE) manages the available bandwidth to maximize the average available bandwidth. Optimizing bandwidth isn’t just about cost: centralized traffic engineering has been shown to solve a number of problems that are traditionally extremely difficult to solve through a combination of distributed routing and traffic engineering [[Kum15\]](https://landing.google.com/sre/sre-book/chapters/bibliography#Kum15).

Some services have jobs running in multiple clusters, which are distributed across the world. In order to minimize latency for globally distributed services, we want to direct users to the closest datacenter with available capacity. Our *Global Service Load Balancer* (GSLB) performs load balancing on three levels:

- Geographic load balancing for DNS requests (for example, to *www.google.com*), described in [Load Balancing at the Frontend](https://landing.google.com/sre/sre-book/chapters/load-balancing-frontend)
- Load balancing at a user service level (for example, YouTube or Google Maps)
- Load balancing at the Remote Procedure Call (RPC) level, described in [Load Balancing in the Datacenter](https://landing.google.com/sre/sre-book/chapters/load-balancing-datacenter)

Service owners specify a symbolic name for a service, a list of BNS addresses of servers, and the capacity available at each of the locations (typically measured in queries per second). GSLB then directs traffic to the BNS addresses.

# Other System Software

Several other components in a datacenter are also important.

## Lock Service

The *Chubby* [[Bur06\]](https://landing.google.com/sre/sre-book/chapters/bibliography#Bur06) lock service provides a filesystem-like API for maintaining locks. Chubby handles these locks across datacenter locations. It uses the Paxos protocol for asynchronous Consensus (see [Managing Critical State: Distributed Consensus for Reliability](https://landing.google.com/sre/sre-book/chapters/managing-critical-state)).

Chubby also plays an important role in master election. When a service has five replicas of a job running for reliability purposes but only one replica may perform actual work, Chubby is used to select *which* replica may proceed.

Data that must be consistent is well suited to storage in Chubby. For this reason, BNS uses Chubby to store mapping between BNS paths and `IP address:port` pairs.

## Monitoring and Alerting

We want to make sure that all services are running as required. Therefore, we run many instances of our *Borgmon* monitoring program (see [Practical Alerting from Time-Series Data](https://landing.google.com/sre/sre-book/chapters/practical-alerting)). Borgmon regularly "scrapes" metrics from monitored servers. These metrics can be used instantaneously for alerting and also stored for use in historic overviews (e.g., graphs). We can use monitoring in several ways:

- Set up alerting for acute problems.
- Compare behavior: did a software update make the server faster?
- Examine how resource consumption behavior evolves over time, which is essential for capacity planning.

# Our Software Infrastructure

Our software architecture is designed to make the most efficient use of our hardware infrastructure. Our code is heavily multithreaded, so one task can easily use many cores. To facilitate dashboards, monitoring, and debugging, every server has an HTTP server that provides diagnostics and statistics for a given task.

All of Google’s services communicate using a Remote Procedure Call (RPC) infrastructure named *Stubby*; an open source version, gRPC, is available.[11](https://landing.google.com/sre/sre-book/chapters/production-environment/#id-mX2u9tnIwix) Often, an RPC call is made even when a call to a subroutine in the local program needs to be performed. This makes it easier to refactor the call into a different server if more modularity is needed, or when a server’s codebase grows. GSLB can load balance RPCs in the same way it load balances externally visible services.

A server receives RPC requests from its *frontend* and sends RPCs to its *backend*. In traditional terms, the frontend is called the client and the backend is called the server.

Data is transferred to and from an RPC using *protocol buffers*,[12](https://landing.google.com/sre/sre-book/chapters/production-environment/#id-BWDu0tehOiB) often abbreviated to "protobufs," which are similar to Apache’s Thrift. Protocol buffers have many advantages over XML for serializing structured data: they are simpler to use, 3 to 10 times smaller, 20 to 100 times faster, and less ambiguous.

# Our Development Environment

Development velocity is very important to Google, so we’ve built a complete development environment to make use of our infrastructure [[Mor12b\]](https://landing.google.com/sre/sre-book/chapters/bibliography#Mor12b).

Apart from a few groups that have their own open source repositories (e.g., Android and Chrome), Google Software Engineers work from a single shared repository [[Pot16\]](https://landing.google.com/sre/sre-book/chapters/bibliography#Pot16). This has a few important practical implications for our workflows:

- If engineers encounter a problem in a component outside of their project, they can fix the problem, send the proposed changes ("changelist," or *CL*) to the owner for review, and submit the CL to the mainline.
- Changes to source code in an engineer’s own project require a review. All software is reviewed before being submitted.

When software is built, the build request is sent to build servers in a datacenter. Even large builds are executed quickly, as many build servers can compile in parallel. This infrastructure is also used for continuous testing. Each time a CL is submitted, tests run on all software that may depend on that CL, either directly or indirectly. If the framework determines that the change likely broke other parts in the system, it notifies the owner of the submitted change. Some projects use a push-on-green system, where a new version is automatically pushed to production after passing tests.

# Shakespeare: A Sample Service

To provide a model of how a service would hypothetically be deployed in the Google production environment, let’s look at an example service that interacts with multiple Google technologies. Suppose we want to offer a service that lets you determine where a given word is used throughout all of Shakespeare’s works.

We can divide this system into two parts:

- A batch component that reads all of Shakespeare’s texts, creates an index, and writes the index into a Bigtable. This job need only run once, or perhaps very infrequently (as you never know if a new text might be discovered!).
- An application frontend that handles end-user requests. This job is always up, as users in all time zones will want to search in Shakespeare’s books.

The batch component is a MapReduce comprising three phases.

The mapping phase reads Shakespeare’s texts and splits them into individual words. This is faster if performed in parallel by multiple workers.

The shuffle phase sorts the tuples by word.

In the reduce phase, a tuple of (*word*, *list of locations*) is created.

Each tuple is written to a row in a Bigtable, using the word as the key.

## Life of a Request

[Figure 2-4](https://landing.google.com/sre/sre-book/chapters/production-environment/#fig_production-environment_life-of-a-request) shows how a user’s request is serviced: first, the user points their browser to *shakespeare.google.com*. To obtain the corresponding IP address, the user’s device resolves the address with its DNS server (1). This request ultimately ends up at Google’s DNS server, which talks to GSLB. As GSLB keeps track of traffic load among frontend servers across regions, it picks which server IP address to send to this user.

![Life of a request.](https://lh3.googleusercontent.com/oABYup26V8DtDwmugmNQebUmUSOzM9UFY-jXD-C31MIuIA7MODQe8fCOT5pHERILEcptT5ymIa12DLZACCJmlQWzE3AT2KV2cop96jA=s0)Figure 2-4. The life of a request

The browser connects to the HTTP server on this IP. This server (named the Google Frontend, or GFE) is a reverse proxy that terminates the TCP connection (2). The GFE looks up which service is required (web search, maps, or—in this case—Shakespeare). Again using GSLB, the server finds an available Shakespeare frontend server, and sends that server an RPC containing the HTTP request (3).

The Shakespeare server analyzes the HTTP request and constructs a protobuf containing the word to look up. The Shakespeare frontend server now needs to contact the Shakespeare backend server: the frontend server contacts GSLB to obtain the BNS address of a suitable and unloaded backend server (4). That Shakespeare backend server now contacts a Bigtable server to obtain the requested data (5).

The answer is written to the reply protobuf and returned to the Shakespeare backend server. The backend hands a protobuf containing the results to the Shakespeare frontend server, which assembles the HTML and returns the answer to the user.

This entire chain of events is executed in the blink of an eye—just a few hundred milliseconds! Because many moving parts are involved, there are many potential points of failure; in particular, a failing GSLB would wreak havoc. However, Google’s policies of rigorous testing and careful rollout, in addition to our proactive error recovery methods such as graceful degradation, allow us to deliver the reliable service that our users have come to expect. After all, people regularly use *www.google.com* to check if their Internet connection is set up correctly.

## Job and Data Organization

Load testing determined that our backend server can handle about 100 queries per second (QPS). Trials performed with a limited set of users lead us to expect a peak load of about 3,470 QPS, so we need at least 35 tasks. However, the following considerations mean that we need at least 37 tasks in the job, or N+2:

- During updates, one task at a time will be unavailable, leaving 36 tasks.
- A machine failure might occur during a task update, leaving only 35 tasks, just enough to serve peak load.[13](https://landing.google.com/sre/sre-book/chapters/production-environment/#id-0vYuXSpSqF0IzCmUg)

A closer examination of user traffic shows our peak usage is distributed globally: 1,430 QPS from North America, 290 from South America, 1,400 from Europe and Africa, and 350 from Asia and Australia. Instead of locating all backends at one site, we distribute them across the USA, South America, Europe, and Asia. Allowing for N+2 redundancy per region means that we end up with 17 tasks in the USA, 16 in Europe, and 6 in Asia. However, we decide to use 4 tasks (instead of 5) in South America, to lower the overhead of N+2 to N+1. In this case, we’re willing to tolerate a small risk of higher latency in exchange for lower hardware costs: if GSLB redirects traffic from one continent to another when our South American datacenter is over capacity, we can save 20% of the resources we’d spend on hardware. In the larger regions, we’ll spread tasks across two or three clusters for extra resiliency.

Because the backends need to contact the Bigtable holding the data, we need to also design this storage element strategically. A backend in Asia contacting a Bigtable in the USA adds a significant amount of latency, so we replicate the Bigtable in each region. Bigtable replication helps us in two ways: it provides resilience should a Bigtable server fail, and it lowers data-access latency. While Bigtable only offers eventual consistency, it isn’t a major problem because we don’t need to update the contents often.

We’ve introduced a lot of terminology here; while you don’t need to remember it all, it’s useful for framing many of the other systems we’ll refer to later.

[9](https://landing.google.com/sre/sre-book/chapters/production-environment/#id-N1KFQTnFxhW-marker)Well, *roughly* the same. Mostly. Except for the stuff that is different. Some datacenters end up with multiple generations of compute hardware, and sometimes we augment datacenters after they are built. But for the most part, our datacenter hardware is homogeneous.

[10](https://landing.google.com/sre/sre-book/chapters/production-environment/#id-BWDuecjF7IPTj-marker)Some readers may be more familiar with Borg’s descendant, Kubernetes—an open source Container Cluster orchestration framework started by Google in 2014; see [*http://kubernetes.io*](http://kubernetes.io/) and [[Bur16\]](https://landing.google.com/sre/sre-book/chapters/bibliography#Bur16). For more details on the similarities between Borg and Apache Mesos, see [[Ver15\]](https://landing.google.com/sre/sre-book/chapters/bibliography#Ver15).

[11](https://landing.google.com/sre/sre-book/chapters/production-environment/#id-mX2u9tnIwix-marker)See [*http://grpc.io*](http://grpc.io/).

[12](https://landing.google.com/sre/sre-book/chapters/production-environment/#id-BWDu0tehOiB-marker)Protocol buffers are a language-neutral, platform-neutral extensible mechanism for serializing structured data. For more details, see [*https://developers.google.com/protocol-buffers/*](https://developers.google.com/protocol-buffers/).

[13](https://landing.google.com/sre/sre-book/chapters/production-environment/#id-0vYuXSpSqF0IzCmUg-marker)We assume the probability of two simultaneous task failures in our environment is low enough to be negligible. Single points of failure, such as top-of-rack switches or power distribution, may make this assumption invalid in other environments.
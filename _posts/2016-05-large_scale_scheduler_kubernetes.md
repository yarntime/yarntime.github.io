---
layout: post
title: 从Kubernetes看如何设计超大规模资源调度系统
---
*写于2016-05*

##背景
在大数据时代，为了合理分配大规模集群的资源，满足日益增多的服务和任务的资源需求，出现了诸如Borg，Mesos，YARN，Omega等一系列的集群资源调度系统。从系统的架构来考虑，可以把它们划分为集中式调度器（Borg），双层调度器（Mesos，YARN），以及共享状态调度器（Omega）。虽然架构不同，但是它们的设计目标（简单合理的利用集群资源）和主要职责（为任务分配主机资源）都是一样的。  

Borg是集中式调度器的代表，它作为集群调度器的鼻祖，在google有超过10年的生产经验，其上运行了包括GFS，BigTable，Gmail，MapReduce等各种类型的任务。Borg虽然是一个集中式的调度器，但是通过多主共同协作可以轻松支撑10K节点以上的集群，就像Borg论文中说的那样：We are not sure where the ultimate scalability limit to Borg’s centralized architecture will come from; so far, every time
we have approached a limit, we’ve managed to eliminate it。不论是Brog的设计理念，还是其中新颖的功能特性，都是各种开源调度系统可以借鉴和努力超越的目标。

Mesos和YARN的设计理念基本相似，它们是双层调度器的代表，尤其是Mesos，可以让多个框架公平的共享集群资源，已经被包括Twitter, Airbnb, Netflix在内的很多公司应用在生产环境中。双层调度器系统包括一个轻量级的中央化调度器，以及多个应用框架自己的调度器，中央化调度器把整块的资源分配给应用框架，再由应用框架调度器把资源切分后分配给具体的任务。通过这种分层设计，Mesos在模拟环境中可以支撑50K节点规模的集群，YARN的目标是最高支撑100K节点规模的集群。

Omega是共享状态调度器的代表，它也支持使用不同的调度器调度不同类型的任务，主要解决的是资源请求冲突的问题。Omega是google试验中的产品，没有在实际的生产环境中使用过。从论文的实验数据中可以看出，omega在可以支持的集群规模、调度效率、集群的资源利用率方面都已经赶超了Borg。

Kubernetes是google开源用来管理docker集群的项目，使用Kubernetes可以自动部署、运行、扩缩容docker应用。Kubernetes虽然和Borg有所不同，但是继承了Borg的设计理念，吸纳了Borg的精华，被业界认为是开源版的Borg。在[Kubernetes调度详解](http://www.skycloudsoftware.com/index.php/2016/04/12/kubernetes003.html)中作者简单介绍过Kubernetes已经实现的默认调度器，本文着重介绍一些Kubernetes在开发或准备开发的一些新特性，从这些特性中看如何设计超大规模的资源调度系统。

几种调度系统的简单对比图如下所示：  

![](https://bytebucket.org/skyform_com_cn/publish_assets/raw/f0aca2d3721cbb1f24f630d431656589170eb368/compare.png) 

##调度系统流程简介  

一些开源的云平台管理系统，如CloudStack，OpenStack等，它们的设计目标及管理的资源类型比较单一，都是使用一个中央化的调度器，根据主机上报的静态资源量，为虚拟机分配主机。  

Mesos和YARN的设计目标是通用的资源管理框架，它们可以支持Hadoop，Spark，S4等一系列的应用框架，使用不同的调度器调度不同类型的任务。它们把调度流程一般包括主机上报可使用的资源量，中央调度器申请资源分配给应用框架，应用框架再把资源分配给具体的任务。

上述的调度系统都是用任务申请时的资源使用情况作为输入进行调度，没有考虑在系统长时间运行后的资源变化对整个集群的影响，这可能会带来以下几个方面的负面影响： 

- 任务申请的资源量和实际使用的资源量不匹配，通常是申请量远远大于实际使用的资源量，导致集群中的大部分资源被浪费。
- 随着任务的增加删除，在集群中会有越来越多的资源碎片。这些资源碎片由于太小无法分配给任务而被浪费。
- 集群中的主机负载不均匀，有的主机负载很高，有的主机负载很低。 

为了解决上述问题，一个完整的资源调度流程通常包括以下几个步骤：

- 主机上报可用资源量
- 应用框架申请资源
- 调度器根据请求进行调度，选择合适的主机
- 应用程序部署到调度的主机上
- 采集程序定期采集应用程序及主机的性能信息，数据发送到存储后端
- 分析程序分析主机及应用程序的历史性能数据，处理后输出可用来调度的信息反馈给调度器
- 调度器根据真实的负载信息进行资源调整

流程如下图所示：  
![](https://bytebucket.org/skyform_com_cn/publish_assets/raw/cdf4a8003cf1bbb1b6d1dc9f038371d0f414227e/scheduler_step.png) 

下面我们来看一下kubernetes在整个调度系统流程中的考虑。


##主机的可用资源量
现在的调度器都是使用主机的容量做调度，但是主机的容量并不是主机可分配的资源量。比如YARN中的nodemanager上报的是主机的容量，实际上系统程序和nodemanager本身会消耗一部分的主机容量，主机可分配的资源量要比主机的容量少。如果用主机的容量做调度，可能会导致主机的负载过高导致系统不稳定的问题。
  
为了做更可靠的调度，尽量减少资源过量使用，kubernetes把主机的资源分为几个部分：

- Node Capacity：主机容量是一个固定值，是主机的实际的容量。
- System-Reserved：不属于kubernetes的进程占用的资源量。
- Kubelet Allocatable：可以被kubelet用来启动容器的资源量。
- Kube-Reserved：被kubernetes的组件占用的资源量，包括docker daemon，kubelet，kube-proxy等。

```shell
     [Allocatable] = [Node Capacity] - [Kube-Reserved] - [System-Reserved]
```

kubernetes调度器在调度Pod和kubelet在部署Pod做资源校验时都使用*Allocatable*资源量作为数据输入。

##任务的资源需求

如果能在任务提交的时候就预测出任务在运行期间会占用多少资源，对调度器来说会是一个重大的改进。现阶段的调度器使用的任务资源量限制都是由用户指定的固定值，这个值往往会大于实际需要的资源量，导致资源浪费。在虚拟机时代，很难预测某个虚拟机在运行期间的资源使用量，因为同样规格、同一模板的虚拟机，因为其上运行的业务不同，会导致资源使用量有较大差异。但是在容器时代，这个资源使用量是可以用历史数据估算出来的，因为同一镜像上跑的业务肯定是一样的。

在这个阶段，需要重点关注的问题是如何快速、准确的预测任务在运行期间会占用的资源量，减少对Pod启动时间带来的延迟，以及减少因Pod资源变化对主机负载带来的影响。

Kubernetes使用已经运行在容器中的镜像（包括名字和标签）的历史数据预测任务的资源使用量，容器在运行期间，会定期把CPU和MEMORY的负载信息存储到后端，周期默认是1分钟。

Kubernetes的资源初始化组件按照如下顺序为容器设置资源需求值：

- 使用7天内同样 image:tag 的容器历史数据进行预测
- 使用30天内同样 image:tag 的容器历史数据进行预测
- 使用30天内同样 image 的容器历史数据进行预测

在处理过程中按照上述顺序进行预测。如果没有找到相关的历史数据，会使用默认的LimitRange对容器进行资源限制。


##多调度器支持

现阶段运行在集群中的任务可以大致划分为两大类：long-running service（如Gmail）和batch job（如MapReduce），通常来说long-running service的优先级会高于batch job任务的优先级。

Kubernetes设计之初是以service为核心，其内部的概念如service，kube-proxy等都是为更好的服务long-running service而设计。在V1.2版本后，Kubernetes开始支持batch job类型的任务。调度这两种类型的任务需要考虑的因素有所不同，比如batch job类型的任务通常是低优先级的任务，更多的是考虑如何回填主机的资源空缺，从而提高整个集群的资源利用率；long-running类型的任务通常是高优先级的任务，更多的是考虑如果预留资源满足自身的SLA需求。Kubernetes现在只有一个默认的调度器，很难满足使用不同的调度策略调度不同类型任务的需求。可喜的是Kubernetes的调度框架是plugin形式的，也就是说，任务框架可以根据自身的需求定制调度策略。  

Kubernetes在支持多调度器时主要解决的问题包括：

- 调度Pod。一个Pod只能由一个调度器调度。可以通过在Pod定义时指定scheduler.alpha.kubernetes.io/name annotation来指定Pod要使用的调度器。
- 解决冲突。Kubernetes对多调度器的支持方式和YARN及Mesos不同，比如Mesos，它的资源是由Mesos-Master分配给各个框架的，框架只能看到Mesos允许它使用的那些资源，而且Mesos会保证同一份资源不会同时分配给多个框架，所以不存在资源冲突的问题。Kubernetes的实现方式更类似于Omega，各个调度器都可以看到集群中的所有资源，当两个调度器同时把Pod调度到同一个主机上时就可能会出现资源冲突。Kubernetes的解决方案是当有冲突发生时，由Kubelet把受影响的Pod重新返回给调度器，等待调度器重新调度。
- 除此之外，怎么样动态加载调度器也是一个需要重点关注的问题，这其中会涉及admission等一系列问题。

##多调度策略支持

Kubernetes通过一组规则，为每一个未调度的Pod选择一个主机。其过程包括主机筛选和主机打分两个阶段。
主机筛选的目的是过滤掉不符合Pod要求的主机，现在kubernetes中实现的过滤规则主要包括以下几种：

- NoDiskConflict：检查在此主机上是否存在卷冲突。如果这个主机已经挂载了卷，其它同样使用这个卷的Pod不能调度到这个主机上。GCE,        Amazon EBS, and Ceph RBD使用的规则如下：  
  
      - GCE允许同时挂载多个卷，只要这些卷都是只读的。  

      - Amazon EBS不允许不同的Pod挂载同一个卷。   

      - Ceph RBD不允许任何两个pods分享相同的monitor，match pool和 image。  
  
- NoVolumeZoneConflict：检查给定的zone限制前提下，检查如果在此主机上部署Pod是否存在卷冲突。假定一些volumes可能有zone调度约束， VolumeZonePredicate根据volumes自身需求来评估pod是否满足条件。必要条件就是任何volumes的zone-labels必须与节点上的zone-labels完全匹配。节点上可以有多个zone-labels的约束（比如一个假设的复制卷可能会允许进行区域范围内的访问）。目前，这个只对PersistentVolumeClaims支持，而且只在PersistentVolume的范围内查找标签。处理在Pod的属性中定义的volumes（即不使用PersistentVolume）有可能会变得更加困难，因为要在调度的过程中确定volume的zone，这很有可能会需要调用云提供商。  

- PodFitsResources：检查主机的资源是否满足Pod的需求。根据实际已经分配的资源量做调度，而不是使用已实际使用的资源量做调度。  

- PodFitsHostPorts：检查Pod内每一个容器所需的HostPort是否已被其它容器占用。如果有所需的HostPort不满足需求，那么Pod不能调度到这个主机上。  

- HostName：检查主机名称是不是Pod指定的HostName。  

- MatchNodeSelector：检查主机的标签是否满足Pod的*nodeSelector*属性需求。  

- MaxEBSVolumeCount：确保已挂载的EBS存储卷不超过设置的最大值。默认值是39。它会检查直接使用的存储卷，和间接使用这种类型存储的PVC。计算不同卷的总目，如果新的Pod部署上去后卷的数目会超过设置的最大值，那么Pod不能调度到这个主机上。  

- MaxGCEPDVolumeCount：确保已挂载的GCE存储卷不超过设置的最大值。默认值是16。规则同上。  

可以通过配置修改Kubernetes默认支持的过滤规则。

经过过滤后，再对符合需求的主机列表进行打分，最终选择一个分值最高的主机部署Pod。Kubernetes用一组优先级函数处理每一个待选的主机。每一个优先级函数会返回一个0-10的分数，分数越高表示主机越“好”， 同时每一个函数也会对应一个表示权重的值。最终主机的得分用以下公式计算得出：
 
    finalScoreNode = (weight1 * priorityFunc1) + (weight2 * priorityFunc2) + ... + (weightn * priorityFuncn) 

现在支持的优先级函数包括以下几种：

- LeastRequestedPriority：如果新的pod要分配给一个节点，这个节点的优先级就由节点空闲的那部分与总容量的比值（即（总容量-节点上pod的容量总和-新pod的容量）/总容量）来决定。CPU和memory权重相当，比值最大的节点的得分最高。需要注意的是，这个优先级函数起到了按照资源消耗来跨节点分配pods的作用。计算公式如下： 
 
		cpu((capacity - sum(requested)) * 10 / capacity) + memory((capacity - sum(requested)) * 10 / capacity) / 2 

- BalancedResourceAllocation：尽量选择在部署Pod后各项资源更均衡的机器。*BalancedResourceAllocation*不能单独使用，而且必须和*LeastRequestedPriority*同时使用，它分别计算主机上的cpu和memory的比重，主机的分值由cpu比重和memory比重的“距离”决定。计算公式如下：  

        score = 10 - abs(cpuFraction-memoryFraction)*10

- SelectorSpreadPriority：对于属于同一个service、replication controller的Pod，尽量分散在不同的主机上。如果指定了区域，则会尽量把Pod分散在不同区域的不同主机上。调度一个Pod的时候，先查找Pod对于的service或者replication controller，然后查找service或replication controller中已存在的Pod，主机上运行的已存在的Pod越少，主机的打分越高。  
  
- CalculateAntiAffinityPriority：对于属于同一个service的Pod，尽量分散在不同的具有指定标签的主机上。  

- ImageLocalityPriority：根据主机上是否已具备Pod运行的环境来打分。ImageLocalityPriority会判断主机上是否已存在Pod运行所需的镜像，根据已有镜像的大小返回一个0-10的打分。如果主机上不存在Pod所需的镜像，返回0；如果主机上存在部分所需镜像，则根据这些镜像的大小来决定分值，镜像越大，打分就越高。  

- NodeAffinityPriority（Kubernetes1.2实验中的新特性）：Kubernetes调度中的亲和性机制。Node Selectors（调度时将pod限定在指定节点上），支持多种操作符（In, NotIn, Exists, DoesNotExist, Gt, Lt），而不限于对节点labels的精确匹配。另外，Kubernetes支持两种类型的选择器，一种是“hard（requiredDuringSchedulingIgnoredDuringExecution）”选择器，它保证所选的主机必须满足所有Pod对主机的规则要求。这种选择器更像是之前的nodeselector，在nodeselector的基础上增加了更合适的表现语法。另一种是“soft（preferresDuringSchedulingIgnoredDuringExecution）”选择器，它作为对调度器的提示，调度器会尽量但不保证满足NodeSelector的所有要求。

##QoS

集群调度器的设计目标是简单合理的利用集群资源，也就是如何在保障业务稳定的前提下尽量提高集群的资源利用率，这是一个非常困难的问题。在被过度使用的系统中（资源请求总和>机器容量）系统会变得不稳定，任务可能最终会被杀掉。如果系统的CPU或者内存资源被耗尽，较为理想的做法是杀掉不太重要的任务，保证高优先级任务的资源需求，在有资源空闲时重新运行低优先级的任务。Borg通过混合部署高低优先级的任务使集群的资源使用率提高了20%以上。  

Kubernetes借鉴了Borg的实现思路，对于每一种资源，Kubernetes将容器分为3种QoS等级：Guaranteed, Burstable和Best-Effort，这三种优先级逐渐递减。

- 目前，仅关注内存方面。在CPU资源限制不能满足时容器并不会被杀掉（在系统任务或者守护进程占用大量CPU的情况下），容器只会被暂时阻塞。
- 内存最低限制为0的容器会被分级为内存Best-Effort级别的容器。这种容器没有最低资源限制，优先级也最低（如果系统内存被耗尽，这些容器的进程将首先被杀掉）。
- 资源请求与资源限制相同（非0）的容器被分级为内存Guaranteed级别，这些容器要求明确定义的资源数量，优先级也最高（在内存使用方面）。
- 其它所有的容器属于Burstable级别的容器，优先级中等。这些容器有最小的资源限制，在资源富余的时候，有权限使用更多资源。
- 在当前的分级策略和具体实施中，从技术层面看，Best-Effort容器是Burstable容器的一个子集（只是资源请求为0），但是Best-Effort容器是一个重要的特殊的类别，Best-Effort容器不需要任何资源限制，这样它们可以利用集群中未使用的资源。

对于每一种资源，容器可以指定一个资源请求和资源限制，0 <= 资源请求<= 资源限制<= 无穷。如果容器被成功调度至节点，容器的资源请求是能够保证的。Kubernetes把资源分为两类，可压缩的资源和不可压缩的资源。如果资源使用量大于可压缩的资源，容器会被阻塞但是不会被杀掉，如果资源使用量大于不可压缩的资源，容器会被杀掉。

可压缩的资源保障：

- 目前，仅支持CPU。
- 最小的CPU资源限制为10M。这是Linux内核限制决定的。
- 容器能够获得要求的CPU量，能否获得额外的CPU时间取决于其它运行的任务。
- 除了请求的CPU数量，额外的CPU资源是共享的。比如说，假定A容器指定需要CPU的60%，B容器请求CPU的30%，假设两个容器都尽可能多地使用CPU资源，那额外的10%的CPU资源将按照2:1的比例分别分配给容器A与容器B。
- 容器资源使用超过资源限制就会被阻塞，如果没有指定资源限制，当有CPU资源可以使用时，容器就可以使用额外的CPU。

不可压缩的资源保障： 

- 目前，仅支持内存。
- 容器可以获得请求的内存资源量，如果超出内存资源请求，容器会被杀掉（当其它容器需要内存时），但是如果容器消耗的资源小于请求的资源量，将不会被杀掉（除非系统任务或者守护进程需要更多的内存）。
- 在容器内存使用量超过内存资源限制时，容器会被杀掉。


##资源调整

资源调整在kubernetes中也称为rescheduler。当前YARN，Mesos等调度系统的调度决策都是在初期资源申请的时候做出的，随着系统的运行，以及其它任务的增加或删除，整个集群的资源使用情况会发生很大的变化，这个时候把其中的一部分任务迁移到别的主机上，对均衡集群中主机的负载、消除资源碎片会有很大帮助。应用框架在申请资源时，分配的主机只需要满足它们的一些条件限制，而应用并不关心任务会运行在哪台具体的主机上。容器时代的资源调整不同于虚拟机时代的资源调整，虚拟机热迁移技术是虚拟机集群资源调整的基础，容器集群资源调整的基础是服务发现、共享存储等技术。

资源调整在Kubernetes中的使用场景大概包括以下两种：

1. 聚合Pod：聚合Pod是指把Pod集中分配到部分主机上。这样做的好处有两个，第一个是集群的资源状态是一直在变化的，在batch-job类型的任务运行结束后，会在主机上留有资源碎片，这些资源碎片会因为不足以容纳未运行的任务的资源需求被浪费掉。把Pod集中起来部署到一部分主机上，可以把资源碎片聚合起来，把未运行的任务调度到这些资源上，从而提高集群的资源利用率。另一个好处是，如果每个主机上都只运行少量的任务，但是因为每台主机上都有任务在运行，必须保证所有的主机都是开机状态。把Pod集中起来部署到一部分主机上，可以关掉没有运行任务的主机，从而节约电力资源。  

2. 分散Pod：随着时间推移，集群中很可能会出现主机负载不均衡的情况，负载很高的主机其上的业务存在运行不稳定，同时负载很低的主机资源被大量浪费，资源调整可以改善这种情况，将高负载主机的任务重新调度到负载低的主机上，有利于维持集群的负载均衡，解决任务运行不稳定与资源浪费的情况。

资源调整也可以达到高优先级任务抢占低优先级任务资源优先运行的要求。  

当然在实现的过程中有很多的因素需要考虑，Kubernetes资源调整是有破坏性的，因为现在还没有容器热迁移的技术，资源调整只能通过杀掉Pod在其它机器上重启，怎么样在对Pod影响最小的情况下进行资源调整是一个重要的问题。资源调整的过程中，必须满足Pod在创建时的一些NodeAffinity限制，避免资源调整的结果和集群调度器产生冲突。

##结语

Kubernetes还很年轻，在经过1.2版本优化后才勉强可以支撑1K+节点的集群，但是它站在巨人Borg的肩膀上，站在比其它开源软件更高的角度去考虑怎么样能更好的管理集群资源，涉及到资源调度的各个过程。   
当然远远不止这些，还有一些方面也是Kubernetes已经在关注的，比如资源quota问题，在混合Guaranteed，Best-Effort等优先级的任务后，对quota增加作用范围；调度器和auto-scaling集成使用的问题；以及调度中考虑GPU资源分配的问题等。有兴趣的读者可以自行查阅相关资料。
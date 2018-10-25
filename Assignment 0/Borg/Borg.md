> Kubernetes 的前身 Google 的内部大型集群管理系统

## 为什么需要大型集群管理？
1. 充分利用资源，节省开支
大量的任务势必需要大型的集群系统，然而如果只给特定的机器分配特定的任务，必定会造成资源的浪费
![IMAGE](https://pic3.zhimg.com/v2-64d6e22aa7c47766e54ad578cbb5a305_r.jpg)
而如果允许及时调度，能够互相叠加的话
![IMAGE](https://pic2.zhimg.com/v2-f105158f601f1a13d924255a40ca4d08_r.jpg)
能使现有资源得到最大化的利用的同时，能够节省购买服务器的开销
2. 让需要大量资源占用的任务能够及时的分配到空闲资源
Google 这个公司需要大量的 MapReduce 工作去优化它的搜索引擎和准备数据给机器学期训练，所以这类公司就需要大量的机器可以让它跑 MapReduce。如果我用分开的集群去跑 MapReduce 那是一个非常大的浪费，我可不可以把 MapReduce 的工作叠加到所有的机器上面，然后只要机器有资源我就跑 MapReduce 呢？这个论文里面也说了事实上 Google 也是这么做的，所以上图就变成了这样：
![IMAGE](resources/EACDB4C9F62FE836731F3A2508F6AC91.jpg =2458x212)
这样就可以最大化 MapReduce 在集群上面可以榨取的 CPU。对于 Google 这类拿 CPU 换钱的公司，这样可以给它制造更多的利润。Borg 这类软件同时也要处理 MapReduce 所需要的 scheduling 的工作，所以 Borg 这类系统也会有 scheduling 的部分。

***
## Borg 的架构

![IMAGE](resources/83BB432D1AFFD777695835201A761689.jpg =1158x1028)
上图是 Borg 整体的架构图，我们可以看到：
* 每个集群是一个 Cell，会队友一个 BorgMaster
* 每个 cell 有 5 个 BorgMaster，但是只有一个真正的 leader。

* 每个机器上会有个 Borglet，BorgMaster 会跟 Borglet 通信，发送各类指令。
* Borglet 不跟 BorgMaster 主动通信，避免太多 Borglet 同时向 Borgmaster 发起请求导致 BorgMaster 崩溃。
* Borglet 和 BorgMaster 的通信之间加了一层 Link shared，将 Borglet 的变化传递给BorgMaster，减少通信和处理的成本。
***
## 具体的工作逻辑
### Task 的声明周期
![IMAGE](resources/AD240F318216A573AFA0AF8AE551BC8E.jpg =1160x932)
用户可以：
* 递交一个工作（submit）
* 销毁一个工作（kill）
* 更新一个工作（update）

而进一步的工作由 Borg 内部完成

### Borg 的内部控制
cell 里面一般更新是渐进的。销毁一个工作的时候用的是 SIGTERM+SIGKILL。
基本概念：
* 资源配置（alloc）：每个工作可以告知系统需要给我预留多少资源，这样系统可以确保不会太过激进的去叠加
* 优先级（priority）：可以告诉 borg 你的工作的优先级，高优先级的程序可以踢走低优先级的程序
* 整体工作配额（quota）：这个部分是在工作发送给 borg 的时候决定要不要接受这个工作，具体 quota 是在 capacity planning 的时候做的，是一个项目资源占比的问题而不是一个软件问题。
* 如何发送工作给 Borg：每个程序发现 Borg 是通过 Chubby 里面的文件的，文件会写明每个 BorgMaster 对应的 cell 跟它的网络信息。之后就是一般的 RPC。

### Scheduling ：Feasibility checking + Scoring
可行性检查（Feasibility checking）：检查出所有可以安排任务的机器
评估(Scoring)：选出最合适的机器来执行任务

打分的原则：
* 最小化需要干掉的其它的工作
* 这个机器上已经有这个工作所需要的软件了
* 分散 failure domain（这个概念在储存系统里面比较多）

性能优化的措施
* 为了优化系统的响应时间，打分之后的分数会被放到缓存里面，知道这个机器上面有变化了才会被重新计算
* 为了减少运算量，同样的机器在同样的需求下面所打的分数也会被放到缓存里面
* 每次检查可行性的时候，不会检查所有的机器的可行性，只要随机抽查到足够的机器符合条件就行

![IMAGE](resources/ECF2AC8B065249B43B8F97C577ECE27E.jpg =1082x666)
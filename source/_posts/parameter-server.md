---
title: 参数服务器
date: 2016-11-12 15:20:32
categories: ML
tags: [parameter-server, distribute, dist-lr]
---

参数服务器（Parameter Sever）是大规模机器学习的一个部分，比如 dmlc 中的 [ps-lite](https://github.com/dmlc/ps-lite), 比如微软的 [project adam](https://www.google.com/url?sa=t&rct=j&q=&esrc=s&source=web&cd=1&cad=rja&uact=8&ved=0ahUKEwiiqp-RiaPQAhUDsI8KHZnNDBMQFgggMAA&url=https%3A%2F%2Fpdfs.semanticscholar.org%2F043a%2Ffbd936c95d0e33c4a391365893bd4102f1a7.pdf&usg=AFQjCNFFnirPzFhPQQ4KTeDH5MX8ff1OEw&sig2=iQsdfQFyUcPhG8kkP9B0-g)。本质上来，参数服务器就是个分布式系统，用于更新机器学习的模型参数。因此，一个完善的参数服务器要考虑到错误容忍、可扩展性、数据一致性等。

<!--more-->

### 基本架构

以 ps-lite 为例，ps-lite 从架构上来说是由 server, worker, scheduler 组成，三者的职责如下

- Sheduler 是个调度器，比如控制训练时的数据同步，对失败结点和新添结点的处理。Sheduler 并不对系统资源使用进行监控和优化分配。
- Server 主要用于数据（在深度学习中，该数据就是梯度）的收集和模型的更新，server 之间会相互通信，进行数据的备份，防止 server 结点失败。
- Worker 用于训练模型。Worker 结点之间不进行通信，因此，如果有一个结点失败了，不会影响其他结点的训练。Worker 有两个主要的操作：push 数据到 server, 从 server pull 数据到本地。

代码层面，ps-lite 使用 Postoffice 类作为信息获取和消息通信的统一入口，使用 zeromq 进行消息通信，消息使用 protobuf 进行封装。

### 同步、异步

支持同步和异步的训练是参数服务器一个特性。

- 同步。直观上来讲，同步就是个 map-reduce 的过程。Server 结点必须等到所有数据都收集完整了，才能进行模型的更新。对于凸优化，同步的训练方式可以保证数据的收敛。但，同步的训练方式对机器的要求比较高，因为如果有一个 worker 结点计算速度过慢，会严重影响训练速度。此外，从实现上而言，同步更容易进行调试。

- 异步。每个 worker 结点可以在任何时刻从 server 结点拉取模型数据进行训练。即每个 worker 结点上的模型可能是不一样的。异步的方式从实现上更为简单，因为它只需要保证 server 结点对更新模型操作时的互斥，而不需要处理不同 worker 结点的同步问题。目前，对于大规模的学习任务，基本都会采用异步的训练方式，但并非所有的算法都支持异步的训练。异步通常来讲会降低模型的收敛速度，甚至导致模型不收敛。因此，实现异步训练，要有相应算法的支持。

在 ps-lite 中，可以通过 Barrier() 函数来实现结点间的同步。比如，如果我想在 server 结点初始化好模型前，阻塞所有 worker 结点的 pull 操作。可以在每个 worker 结点加入该操作（具体细节可参考 [dist-lr](https://github.com/ctliu3/dist-lr/blob/master/src/main.cc) 的实现）：

	ps::Postoffice::Get()->Barrier(ps::kWorkerGroup);
	
	
此外，ps-lite 也利用时间戳的方式来实现同步训练。每个 worker 结点在给 push 数据给 server 时，都会带上一个时间戳，该时间戳对应一个回调函数。Server 结点的 response 中会带上该时间戳，这样就使得 worker 结点会在 server 返回结果前阻塞，而在收到 response 时执行相应的函数。


### [dist-lr](https://github.com/ctliu3/dist-lr)

dist-lr 是我基于 ps-lite 写的一个分布式 logistic regression 的项目。目前实现的是带 L2 正则的单机版本，但只要做一些数据拷贝的处理，很容易扩展成多机的版本。支持同步和异步的训练。

[examples](https://github.com/ctliu3/dist-lr/tree/master/examples) 目录下放了个 demo，数据集是 [a9a](https://www.csie.ntu.edu.tw/~cjlin/libsvmtools/datasets/binary.html), 这是一个二分类数据，特征维度为 123, 正样本有 32561 个，负样本有 16281 个。为了测试数据并行的效果，我把训练集分成了 4 份。训练时迭代了 20 次。

|  | elapsed | accuracy |
| ------------- |:-------------:| :-----:|
| sync | 108s | 82.7% |
| async | 70s | 71.1% |

同步训练时，准确率是一直上升的，而异步时，准确率会有比较大的振荡，无法很好的收敛。下面是某一次异步训练时的收敛情况：

	19:16:08 Iteration   0, accuracy: 0.726061
	19:16:16 Iteration  20, accuracy: 0.832258
	19:16:23 Iteration  40, accuracy: 0.832934
	19:16:31 Iteration  60, accuracy: 0.840182
	19:16:39 Iteration  80, accuracy: 0.796634
	19:16:47 Iteration 100, accuracy: 0.847491
	19:16:55 Iteration 120, accuracy: 0.668509
	19:17: 3 Iteration 140, accuracy: 0.793133
	19:17:10 Iteration 160, accuracy: 0.848965
	19:17:18 Iteration 180, accuracy: 0.711013


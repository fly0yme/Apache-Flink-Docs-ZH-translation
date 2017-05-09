---
title: 分布式运行环境
nav-pos: 2
nav-title: 分布式运行
nav-parent_id: 概念
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

* This will be replaced by the TOC
{:toc}

## 任务和操作链

由于是分布式执行, Flink *chains* 操作子任务合并成 *tasks*.每个任务由一个线程执行.
链式操作合并任务是一个有效的优化: 它减少了线程到线程切换和缓冲的开销, 并且在减少延迟的同时增加了总体吞吐量.
可以在API中使用这种链式操作.

下图中的示例数据流由五个子任务执行,因此具有五个并行线程.

<img src="../fig/tasks_chains.svg" alt="Operator chaining into Tasks" class="offset" width="80%" />

{% top %}

## Job Managers, Task Managers, Clients

Flink 运行时由两种线程组成:

 - **JobManagers** (也称为 *masters*) 协调分布式执行. 他们安排任务, 协调检查点, 协调故障恢复等.Flink运行时至少有一个 Job Manager.对于高可用的则需要配置多个 JobManagers, 其中的一个作为 *领导*, 其他的处于 *备用*状态.

 - **任务管理者** (也被称为 *工作者*) 执行 *任务* (或者更具体的说, 是子任务) 的数据流,和缓冲以及转换的 *streams* data.

    Flink运行必须至少有一个TaskManager.

JobManagers和TaskManagers可以通过各种方式启动: 直接在机器上，容器中，或由YARN等资源框架进行管理. TaskManagers通过连接到 JobManagers,宣布自己是可用的, 并且分配工作.

**客户端**不是运行和程序执行的一部分, 而是被用来准备和发送的数据流的JobManager.之后, 客户端可以断开连接或保持连接以接收进度报告. 客户端作为Java/Scala 程序执行的一部分,改程序触发执行或者在命令行中通过命令执行 `./bin/flink run ...`.

<img src="../fig/processes.svg" alt="The processes involved in executing a Flink dataflow" class="offset" width="80%" />

{% top %}

## Task Slots and Resources

每一个 worker (TaskManager) 是一个 *JVM 进程*, 可以在单独的线程中会执行一个或多个子任务.
为了控制一个worker接收多少任务, 一个 worker 被叫做 **任务资源** (至少一个).

每一个 *任务资源* 代表一个TaskManager的固定的资源子集.例如， 一个TaskManager有三个 slots, 会将其管理内存的1/3用于每个slot. 这种资源隔离意味着不会和其他任务的子任务竞争管理资源,因此每一个管理任务都会有一个确定的保留内存.请注意不是CPU隔离; 只是隔离任务的管理内存.

通过调整任务的资源槽数,用户可以定义子任务间是怎样相互隔离的. 
每个TaskManager有一个资源槽意味着每一个任务组运行在一个独立的JVM上 (例如,也可以运行在一个独立的container中).每个TaskManager有对个资源槽
意味着更多的更多的子任务共享一个JVM. 在同一个JVM中的任务共享TCP连接(通过复用) 和心跳信息.他们可能会共享数据集和数据结构, 从而减少每个任务的开销.

<img src="../fig/tasks_slots.svg" alt="A TaskManager with Task Slots and Tasks" class="offset" width="80%" />

默认情况下，Flink允许子任务共享资源槽，即使它们是不同任务的子任务，只要它们来自相同的job.结果是一个插槽可以容纳整个工作管道.允许这种 *资源共享* 有两个主要好处：

- Flink集群需要与作业中使用的资源槽数和最高并行度完全相同。不需要计算一个程序总共包含多少任务（具有不同的并行度）。

- 更容易获得更好的资源利用率。没有资源槽分配，非密集的  *source/map()* 子任务将占用与资源密集型 *window* 子任务一样多的资源。示例中，我们通过资源槽分配，将基本并行度从2增加到6，可以充分利用时隙资源，同时确保重要子任务在TaskManagers之间公平分配。

<img src="../fig/slot_sharing.svg" alt="TaskManagers with shared Task Slots" class="offset" width="80%" />

这些API还包括一个*资源组* 机制，可用于防止不需要的时隙共享。

根据经验,资源槽最好的默认数量是CPU内核的数量.使用超线程技术，每个资源槽需要2个或更多的硬件线程。 

{% top %}

## 后端状态

键/值索引存储的确切数据结构取决于所选的**后端状态**。一个后端状态将数据存储在内存中的哈希映射中，另一个后端状态使用RocksDB(http://rocksdb.org)作为键/值存储。除了定义保存状态的数据结构之外，后端状态还实现逻辑以获取键/值状态的时间点快照，并将该快照存储为检查点的一部分。

<img src="../fig/checkpoints.svg" alt="checkpoints and snapshots" class="offset" width="60%" />

{% top %}

## 保存点

使用Data Stream API编写的程序可以从**保存点**恢复执行。保存点允许更新程序和Flink集群，而不会丢失任何状态。

保存点是 **手动触发的**手动触发的检查点.它们记录程序的快照并将其写入后端状态.他们依靠这个常规的检查点机制.执行过程中，定期在工作节点上快照并生成检查点。对于恢复，只需要完成最后的检查点，一旦新的检查点完成，可以安全地丢弃较旧的检查点。

保存点与这些定期检查点类似，除了它们**由用户触发**，并且在较新的检查点完成时不会自动过期。

{% top %}
